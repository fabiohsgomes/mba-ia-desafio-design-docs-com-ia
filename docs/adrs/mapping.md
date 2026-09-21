# Mapeamento arquitetural do sistema

## 1. Objetivo e escopo da análise

Este documento registra a Fase 1 do levantamento de ADRs: um mapa modular da API de
gerenciamento de pedidos e dos pontos de extensão necessários para a feature de webhooks
outbound discutida em `TRANSCRICAO.md`.

A leitura foi deliberadamente limitada ao escopo solicitado:

- código da aplicação em `src/`;
- modelo, migrações e seed em `prisma/`;
- testes e helpers em `tests/`;
- contexto funcional e decisões da reunião em `TRANSCRICAO.md`.

O mapa distingue o comportamento verificado no código atual das mudanças propostas na
transcrição. Nenhuma implementação de webhook existe ainda nas áreas analisadas.

## 2. Visão geral do projeto

Trata-se de uma API HTTP modular para gestão de usuários, clientes, produtos e pedidos. O
sistema expõe rotas versionadas sob `/api/v1`, persiste dados em MySQL por meio do Prisma e
autentica chamadas com JWT. A composição de dependências é manual: `src/app.ts` instancia
repositories, services e controllers, enquanto `src/routes/index.ts` agrega os routers dos
domínios.

### Stack identificada

| Área | Tecnologia/padrão | Evidência |
| --- | --- | --- |
| Runtime e linguagem | Node.js, TypeScript e módulos ES | imports com extensão `.js` em `src/` |
| API HTTP | Express | `src/app.ts`, `src/routes/index.ts` |
| Persistência | Prisma Client sobre MySQL | `src/config/database.ts`, `prisma/schema.prisma` |
| Validação | Zod em variáveis de ambiente e entrada HTTP | `src/config/env.ts`, `src/middlewares/validate.middleware.ts` |
| Autenticação | JWT Bearer; papéis `ADMIN` e `OPERATOR` | `src/middlewares/auth.middleware.ts` |
| Senhas | bcrypt | `src/modules/auth/auth.service.ts`, `src/modules/users/user.service.ts` |
| Observabilidade | Pino e request ID | `src/shared/logger/index.ts`, `src/middlewares/request-logger.middleware.ts` |
| Testes | Vitest, Supertest e banco real via Prisma | `tests/auth.test.ts`, `tests/orders.test.ts`, `tests/setup.ts` |

## 3. Forma arquitetural atual

Os domínios de negócio seguem uma separação consistente:

```text
requisição HTTP
  -> routes (autenticação e validação)
  -> controller (adaptação HTTP)
  -> service (regras e orquestração)
  -> repository/Prisma (persistência)
  -> MySQL
```

As exceções percorrem o caminho inverso via `next(err)` e são traduzidas pelo middleware
central de erros. A estrutura não é uma separação rígida de todas as consultas: services mais
simples usam repositories, mas `OrderService` recebe também o `PrismaClient` para coordenar
transações de negócio com múltiplas tabelas.

## 4. Mapa de módulos

As estimativas indicam o tamanho relativo do escopo arquitetural, não esforço de implementação.

### MOD-01 — Bootstrap e composição da aplicação

- **Localização:** `src/app.ts`, `src/server.ts`, `src/routes/index.ts`
- **Escopo:** médio
- **Responsabilidade atual:** construir o grafo controller/service/repository, configurar Express,
  montar as rotas versionadas, iniciar o servidor HTTP e encerrar servidor e Prisma de forma
  controlada.
- **Dependências principais:** todos os módulos de domínio, middlewares, Prisma e logger.
- **Ponto de extensão:** o módulo de webhooks deverá ser composto em `buildControllers` e montado
  no router central. A reunião também propõe `src/worker.ts` como segundo entry point, separado de
  `src/server.ts` e com Prisma próprio por processo.

### MOD-02 — Autenticação e usuários

- **Localização:** `src/modules/auth/`, `src/modules/users/`,
  `src/middlewares/auth.middleware.ts`
- **Escopo:** médio
- **Responsabilidade atual:** cadastro, login, emissão/verificação de JWT, exposição do usuário
  autenticado e autorização por papel.
- **Modelo de identidade verificado:** o token contém `sub`, `email` e `role`; `AuthUser` não contém
  `customerId`.
- **Ponto de extensão:** o replay de DLQ proposto deve reutilizar `authenticate` e
  `requireRole('ADMIN')`. O CRUD comum de webhooks será autenticado, mas a associação segura entre
  usuário e cliente precisa ser explicitada, pois a transcrição decidiu receber `customer_id` por
  body ou path e o código atual não comprova vínculo usuário-cliente.

### MOD-03 — Clientes

- **Localização:** `src/modules/customers/`
- **Escopo:** pequeno
- **Responsabilidade atual:** CRUD autenticado e listagem paginada com busca por nome, e-mail ou
  documento.
- **Padrão interno:** controller, service, repository, routes e schemas; conflitos de e-mail são
  expressos com `ConflictError`.
- **Ponto de extensão:** cada configuração de webhook proposta pertence a um cliente. O modelo
  `Customer` em `prisma/schema.prisma` deverá ganhar a relação correspondente.

### MOD-04 — Produtos e estoque

- **Localização:** `src/modules/products/`
- **Escopo:** pequeno
- **Responsabilidade atual:** CRUD autenticado de produtos, paginação, busca, unicidade de SKU e
  estado ativo.
- **Integração relevante:** o estoque é debitado ou reposto por `OrderService` durante transições
  específicas de status. Isso faz parte da mesma fronteira transacional na qual a outbox deverá
  ser gravada.

### MOD-05 — Pedidos e máquina de estados

- **Localização:** `src/modules/orders/`
- **Escopo:** grande e crítico
- **Responsabilidade atual:** criar pedidos e histórico inicial, calcular totais, reservar número
  sequencial, listar/consultar pedidos, validar transições, alterar estoque, registrar histórico e
  restringir exclusão.
- **Regras de transição:** estão isoladas em `src/modules/orders/order.status.ts`; débito ocorre em
  `PENDING -> PAID` e reposição ao cancelar a partir de `PAID` ou `PROCESSING`.
- **Fronteira transacional crítica:** `OrderService.changeStatus` abre uma transação Prisma que lê o
  pedido, valida a transição, altera estoque, atualiza `orders`, cria
  `order_status_history` e relê o agregado.
- **Ponto de extensão:** segundo a reunião, o snapshot do evento e as linhas de outbox destinadas
  aos webhooks interessados devem ser criados dentro dessa mesma transação. A proposta é uma
  operação como `publishWebhookEvent(tx, order, fromStatus, toStatus)`, que recebe
  `Prisma.TransactionClient` sem acoplar `OrderService` a todo o repository de webhooks.

### MOD-06 — Persistência e modelo relacional

- **Localização:** `src/config/database.ts`, `prisma/schema.prisma`, `prisma/migrations/`,
  `prisma/seed.ts`
- **Escopo:** grande e transversal
- **Responsabilidade atual:** conexão Prisma/MySQL e modelos `User`, `Customer`, `Product`, `Order`,
  `OrderItem`, `OrderStatusHistory` e `OrderNumberSequence`.
- **Convenções verificadas:** identificadores de entidades são UUID em `CHAR(36)`; nomes físicos
  usam snake_case via `@@map`; relações e filtros frequentes têm índices explícitos.
- **Ponto de extensão:** a proposta requer persistência para configuração de endpoints, outbox,
  histórico/tentativas de entrega e dead letter. A outbox precisa de índices que sustentem a busca
  por estado, próxima tentativa e ordem de criação. O UUID do evento nasce na inserção e permanece
  estável durante retries e replay, salvo decisão posterior em contrário.

### MOD-07 — Webhooks outbound (proposto)

- **Localização alvo:** `src/modules/webhooks/`
- **Escopo:** grande
- **Estado:** inexistente no código analisado; definido funcionalmente em `TRANSCRICAO.md`.
- **Responsabilidades propostas:** CRUD de endpoints por cliente, geração e rotação de secret,
  filtro por status, produção transacional de eventos, serialização do snapshot, assinatura,
  envio HTTP, registro de tentativas, consulta de entregas, retry, DLQ e replay administrativo.
- **Forma esperada:** repetir `controller`, `service`, `repository`, `routes` e `schemas`, com a lógica
  de consumo em `webhook.processor.ts` ou `webhook.worker.ts`.
- **Contrato proposto:** evento `order.status_changed`, JSON sem itens e com identificadores,
  estados anterior/novo, timestamp, cliente e dados básicos do pedido; headers
  `X-Event-Id`, `X-Webhook-Id`, `X-Signature`, `X-Timestamp` e `Content-Type`.

### MOD-08 — Worker de entrega (proposto)

- **Localização alvo:** `src/worker.ts` e processador no módulo de webhooks
- **Escopo:** grande e operacional
- **Estado:** inexistente no código analisado.
- **Responsabilidade proposta:** processo Node separado da API, conectado ao mesmo MySQL por uma
  instância própria de `PrismaClient`, fazendo polling a cada 2 segundos, lendo lotes pequenos dos
  eventos elegíveis e persistindo o resultado.
- **Limitação acordada:** um único worker preserva ordenação implícita por criação para eventos de
  um pedido; múltiplos workers exigiriam particionamento ou locking e ficaram fora desta fase.
- **Aspectos ainda a detalhar:** claim atômico, lease/recuperação de registros em processamento,
  tamanho do lote, concorrência interna e comportamento diante de shutdown.

### MOD-09 — Infraestrutura HTTP compartilhada

- **Localização:** `src/middlewares/`, `src/shared/errors/`, `src/shared/http/`
- **Escopo:** médio e transversal
- **Responsabilidade atual:** autenticação/autorização, validação de body/query/params, request ID,
  logging de requisições, paginação e contrato uniforme de erros.
- **Ponto de extensão:** erros de webhook devem derivar de `AppError`, usar códigos com prefixo
  `WEBHOOK_` e passar pelo middleware atual. Schemas Zod devem rejeitar URL não HTTPS e entradas
  inválidas. A validação de URL precisará ir além da forma sintática caso o design cubra SSRF.

### MOD-10 — Observabilidade

- **Localização:** `src/shared/logger/index.ts`, `src/middlewares/request-logger.middleware.ts`
- **Escopo:** pequeno hoje; impacto transversal na feature
- **Responsabilidade atual:** logs estruturados Pino, nível configurável, identificação de serviço,
  request ID e redaction de credenciais usuais.
- **Ponto de extensão:** logs do worker devem reutilizar Pino e correlacionar `eventId`, `webhookId`,
  tentativa, duração e resultado. O redaction atual cobre password/token, mas não cobre `secret`;
  a secret de HMAC não pode aparecer em logs.

### MOD-11 — Testes de integração

- **Localização:** `tests/`
- **Escopo:** médio
- **Responsabilidade atual:** exercitar a API com Supertest e banco real, criar dados por factories e
  limpar tabelas antes de cada teste. `tests/orders.test.ts` verifica criação, totais, transições,
  estoque, paginação e erros de domínio.
- **Ponto de extensão:** adicionar limpeza das tabelas de webhook respeitando FKs e testes para CRUD,
  autorização de replay, filtro por status, atomicidade status/outbox, snapshot imutável, assinatura,
  timeout, retry, DLQ, replay e duplicidade at-least-once.

## 5. Fluxos arquiteturais relevantes

### 5.1 Mudança de status atual

1. `PATCH /api/v1/orders/:id/status` passa por autenticação e Zod.
2. `OrderController.changeStatus` delega ao service com o ID do usuário.
3. `OrderService.changeStatus` abre uma transação Prisma.
4. A transação valida a máquina de estados e ajusta estoque quando necessário.
5. A mesma transação atualiza o pedido e cria o histórico.
6. O pedido completo é relido e retornado antes do commit.

### 5.2 Mudança de status com outbox proposta

1. Os passos atuais permanecem na mesma transação.
2. Após determinar `from` e `to`, são consultados os endpoints ativos daquele cliente interessados
   no novo status.
3. Para cada destino aplicável, grava-se uma linha de outbox com UUID e payload já renderizado.
4. Falha em qualquer gravação provoca rollback de status, histórico, estoque e outbox.
5. Após o commit, o worker independente encontra o evento; a chamada HTTP nunca ocorre dentro da
   requisição de mudança de status.

### 5.3 Entrega proposta

1. O worker busca eventos elegíveis em ordem de criação.
2. O corpo JSON é validado contra o limite de 64 KB.
3. A entrega usa HTTPS, timeout de 10 segundos e HMAC-SHA256 com secret do endpoint.
4. Sucesso registra a entrega; falha registra evidência e agenda nova tentativa.
5. A progressão acordada é `1m / 5m / 30m / 2h / 12h`: uma chamada inicial seguida de cinco
   retries, totalizando no máximo seis chamadas antes da DLQ.
6. Ao esgotar a política, o evento é persistido em DLQ; um administrador pode solicitar replay.

## 6. Preocupações transversais e lacunas de design

### Atomicidade e consistência

O ponto mais forte do desenho é reutilizar a transação já existente em
`src/modules/orders/order.service.ts`. A futura API interna de publicação deve aceitar o transaction
client ativo; abrir outra transação ou gravar a outbox após o commit quebraria a garantia pretendida.

### Semântica de entrega

A reunião escolheu at-least-once. Consequentemente, confirmação remota e persistência local não são
atômicas: uma falha após o cliente processar e antes de o worker registrar sucesso pode gerar nova
entrega. `X-Event-Id` deve permanecer estável para permitir deduplicação pelo consumidor.

### Segurança

- HMAC-SHA256 e secret exclusiva por endpoint foram decididos.
- TLS é obrigatório e a URL deve ser HTTPS.
- A rotação terá janela de 24 horas, mas é preciso especificar qual secret assina durante a janela
  e como o receptor valida ambas.
- O contrato deve dizer exatamente quais bytes entram no HMAC. A transcrição diz “sobre o corpo”,
  mas também envia `X-Timestamp`; cobrir o timestamp na assinatura fortaleceria a proteção contra
  replay e precisa de decisão explícita.
- Como HMAC exige acesso ao segredo, hashing unidirecional não basta. A estratégia de criptografia
  em repouso e gestão de chave não foi definida.
- HTTPS sintático não impede SSRF; redirects, DNS, loopback e faixas privadas precisam de política.

### Autorização e multitenancy

O JWT atual identifica usuário e papel, não cliente. A transcrição reconhece essa diferença e opta
por `customer_id` no body/path. Sem uma regra adicional, autenticação não equivale a autorização
sobre o cliente informado. O ADR ou design da API precisa declarar se a API é interna e ampla ou
qual vínculo será usado para isolar clientes.

### Concorrência, ordenação e recuperação

Single-worker e ordenação por criação são a decisão inicial, mas retry pode permitir que eventos
posteriores ultrapassem um evento falho. Também faltam protocolo de claim, lease e recuperação após
crash. Esses detalhes determinam se um evento pode ficar preso em `PROCESSING` e se o sistema mantém
ordem por pedido sob falhas.

### Retenção e dados sensíveis

Entregas devem expor os últimos 100 resultados com payload, resposta e duração. É necessário limitar
o tamanho do corpo de resposta armazenado, redigir dados sensíveis e definir retenção. Arquivamento
de entregas após cerca de 30 dias foi mencionado, mas ficou fora do escopo desta feature.

## 7. Fronteiras de dependência esperadas

```text
orders service
  -> publisher/enqueuer de webhooks (usa TransactionClient recebido)
      -> modelos de configuração + outbox no MySQL

worker entry point
  -> processor de webhooks
      -> repository de outbox/deliveries/DLQ
      -> cliente HTTP + assinatura HMAC
      -> PrismaClient próprio do processo
      -> logger compartilhado

webhooks HTTP API
  -> controller -> service -> repository
  -> autenticação, requireRole, Zod, AppError e paginação existentes
```

O módulo de pedidos conhece apenas a capacidade de enfileirar eventos dentro da transação; não deve
conhecer transporte HTTP, retry ou DLQ. O worker conhece o contrato persistido de entrega, mas não
deve alterar regras da máquina de estados de pedidos.

## 8. Inventário preliminar de decisões para ADRs

Este inventário prepara a identificação da Fase 2. Cada item possui alternativa real discutida ou
plausível e pode originar um ADR independente.

| ID candidato | Decisão | Opções relevantes | Evidência/motivação |
| --- | --- | --- | --- |
| ADR-C01 | Persistência transacional de eventos | Outbox no MySQL; chamada síncrona; broker/Redis Streams | A reunião rejeitou HTTP dentro de `changeStatus` e nova infraestrutura; a transação existente fornece a fronteira atômica. |
| ADR-C02 | Execução da entrega | Processo separado com polling de 2 s; worker embutido na API; notificação externa | MySQL não oferece equivalente a LISTEN/NOTIFY; latência alvo é menor que 10 s. |
| ADR-C03 | Retry e falha permanente | Backoff limitado + DLQ separada; retry indefinido; menos tentativas; estado final na outbox | A reunião acordou `1m/5m/30m/2h/12h`, DLQ persistida e replay admin. |
| ADR-C04 | Autenticidade da entrega | HMAC-SHA256 por endpoint; secret global; assinatura assimétrica; sem assinatura | Requisito de verificar origem/integridade e limitar impacto de vazamento. |
| ADR-C05 | Semântica de entrega | At-least-once + `X-Event-Id`; exactly-once; at-most-once | Falhas entre envio e confirmação tornam duplicatas possíveis; consumidor deduplica pelo UUID. |
| ADR-C06 | Organização do código | Reutilizar módulo em camadas e shared infra; subsistema independente; lógica no módulo orders | A codebase já usa controller/service/repository/routes/schemas, AppError, Zod, Prisma e Pino. |
| ADR-C07 | Momento e conteúdo do payload | Snapshot na inserção; renderização no envio; referência apenas a `order_id` | Snapshot preserva o estado observado quando a transição ocorreu. |
| ADR-C08 | Ordenação e concorrência | Single-worker inicial; paralelismo com partição/locking | A reunião aceita a limitação inicial, mas retries e escala futura exigem semântica explícita. |
| ADR-C09 | Rotação e armazenamento de secrets | Duas secrets por 24 h com criptografia; corte imediato; secret global | Grace period foi decidido, mas direção da assinatura e custódia ainda não. |
| ADR-C10 | Segurança de destinos | HTTPS + defesa SSRF; validação apenas sintática; allowlist | A reunião exige HTTPS, porém URLs são controladas por usuários autenticados. |

## 9. Diretrizes para a próxima fase

1. Criar ADRs pequenos e independentes para, no mínimo, ADR-C01 a ADR-C06.
2. Marcar como **Proposto** o que ainda não está implementado, mesmo quando houve consenso na reunião.
3. Citar diretamente `src/modules/orders/order.service.ts`, `src/app.ts`,
   `src/routes/index.ts`, `src/middlewares/auth.middleware.ts`, `src/shared/errors/` e
   `src/shared/logger/index.ts` quando a decisão depender do desenho atual.
4. Separar decisão de requisito: por exemplo, “payload máximo de 64 KB” pode ser consequência ou
   regra secundária, enquanto outbox e semântica at-least-once merecem ADRs próprios.
5. Documentar explicitamente a semântica de retries, rotação, assinatura, claim/lease,
   ordenação sob retry e autorização por cliente antes de tratar o design como implementável.
6. Preservar nos ADRs os trade-offs negativos: polling adiciona latência e carga; at-least-once
   transfere deduplicação ao consumidor; HMAC exige custódia de segredo recuperável; outbox aumenta
   estado operacional no banco; single-worker limita throughput.

## 10. Síntese

A feature encaixa-se na arquitetura atual por dois motivos: existe um padrão modular claro para a
API e existe uma transação de mudança de status que já reúne pedido, histórico e estoque. A outbox
deve se acoplar somente a essa fronteira transacional, enquanto entrega, retry e DLQ permanecem em
um processo independente. Os ADRs subsequentes devem formalizar não apenas as seis escolhas centrais
da reunião, mas também as condições operacionais e de segurança necessárias para que essas escolhas
sejam verificáveis em produção.
