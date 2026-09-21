### PRD: Sistema de Gestão de Pedidos Webhooks de Notificação

Versão: 1.0
Data: 19 de setembro de 2026
Responsável: Marcos, Product Manager

---

### Resumo

A feature permitirá que clientes B2B recebam notificações automáticas quando o status de seus
pedidos mudar. Ela substituirá consultas repetidas à API por webhooks outbound seguros, rastreáveis
e tolerantes a falhas. A entrega será incorporada ao sistema existente sem bloquear a mudança de
status do pedido nem depender da disponibilidade imediata do cliente.

---

### Contexto e problema

Público-alvo

- clientes B2B que integram sistemas próprios à plataforma;
- equipes técnicas responsáveis por receber e processar eventos de pedidos;
- operadores que administram integrações de clientes;
- administradores que investigam falhas e executam replay.

Cenários de uso chave

- receber uma notificação poucos segundos após a mudança de status de um pedido;
- escolher quais status cada endpoint deseja receber;
- validar que a mensagem foi emitida pela plataforma e não foi alterada;
- consultar tentativas de entrega e investigar falhas;
- recuperar manualmente uma entrega enviada para a DLQ.

Onde essa feature será implantada

- na API existente de gestão de pedidos, construída com Node.js, Express, TypeScript, Prisma e
  MySQL, acompanhada por um processo worker separado no mesmo ambiente operacional.

Problemas priorizados

- Alta: clientes consultam `GET /orders` repetidamente, gerando atraso e chamadas desnecessárias.
- Alta: três clientes solicitaram formalmente notificações, e a Atlas Comercial indicou risco de
  migração para um concorrente se a necessidade não for atendida.
- Alta: uma entrega síncrona ligaria a mudança de status à latência e à disponibilidade do cliente.
- Alta: registrar o evento depois da transação poderia confirmar o pedido e perder a notificação.
- Média: falhas externas precisam ser recuperáveis e rastreáveis sem retries infinitos.

---

### Objetivos e métricas

| Objetivo | Métrica | Meta |
| --- | --- | --- |
| Reduzir espera | Tempo do commit à primeira tentativa | p95 menor que 10 segundos |
| Evitar perda | Mudanças com outbox correspondente | 100% com destino interessado |
| Dar destino às entregas | Eventos em estado terminal | 100%, sem eventos presos |
| Permitir consumo seguro | Entregas HTTPS assinadas e identificadas | 100% |
| Atender a demanda inicial | Clientes solicitantes aptos a consumir | 3 clientes |

---

### Escopo

Incluso

- cadastro, listagem, atualização e remoção de endpoints por cliente;
- seleção dos status de pedido observados por endpoint;
- geração de secret exclusiva e rotação com janela de 24 horas;
- registro atômico de eventos quando o status do pedido mudar;
- worker separado com polling a cada 2 segundos;
- entrega HTTPS assinada com HMAC-SHA256;
- garantia at-least-once e identificação estável por `X-Event-Id`;
- tentativa inicial e cinco retentativas com backoff, totalizando no máximo seis chamadas;
- DLQ persistida e replay administrativo auditado;
- consulta do histórico de tentativas e resultados;
- autorização de operadores por cliente;
- documentação do contrato para consumidores.

Fora de escopo

- envio HTTP síncrono dentro da mudança de status, descartado na reunião;
- Redis Streams, broker ou nova infraestrutura de mensageria, descartados nesta fase;
- trigger do MySQL para acionar o worker, descartada por não notificar processos externos;
- retry indefinido, descartado para evitar eventos e custos sem limite;
- garantia exactly-once, descartada por exigir coordenação entre sistemas independentes;
- eventos inbound enviados pelos clientes para a plataforma;
- múltiplos workers, particionamento e escala horizontal, adiados para uma fase futura;
- garantia de ordenação global, não solicitada pelos clientes;
- rate limiting de saída por cliente, adiado até existirem dados de uso;
- alerta por e-mail para endpoints degradados, adiado para uma fase futura;
- dashboard visual, reservado para um projeto separado do frontend;
- arquivamento ou expurgo automático após aproximadamente 30 dias, adiado;
- circuit breaker para destinos externos;
- emissão de webhook na criação inicial do pedido em `PENDING`.

---

### Requisitos funcionais

#### RF-001 Gerenciar endpoints de webhook

O sistema deve permitir criar, listar, atualizar e remover endpoints pertencentes a um cliente.

**Fluxo principal**

- O operador autenticado seleciona um cliente ao qual possui acesso.
- Informa uma URL HTTPS e os status de interesse.
- O sistema valida os dados, cria o endpoint ativo e devolve sua secret uma única vez.
- Listagens e atualizações retornam a configuração sem material de secret.

**Fluxos alternativos e exceções**

- Um administrador pode gerenciar endpoints de qualquer cliente.
- Remover um endpoint impede novos eventos e cancela entregas ainda pendentes.
- Atualizar os filtros afeta apenas eventos produzidos depois da alteração.

**Erros previstos**

- cliente ou webhook inexistente;
- URL inválida, insegura ou já cadastrada para o cliente;
- usuário sem acesso ao cliente;
- lista de status vazia ou inválida.

**Prioridade:** alta

---

#### RF-002 Filtrar notificações por status

Cada endpoint deve receber somente mudanças cujo novo status esteja em sua configuração.

**Fluxo principal**

- O cliente seleciona um ou mais valores válidos de status.
- A cada mudança, o sistema localiza endpoints ativos interessados no novo status.
- Um evento independente é registrado para cada destino encontrado.

**Fluxos alternativos e exceções**

- Se nenhum endpoint estiver interessado, a mudança confirma sem criar evento.
- Alterações posteriores do filtro não modificam eventos já registrados.

**Erros previstos**

- status desconhecido;
- filtro vazio;
- endpoint inativo ou removido.

**Prioridade:** alta

---

#### RF-003 Gerar e rotacionar secrets

O sistema deve manter uma secret exclusiva por endpoint e permitir sua rotação segura.

**Fluxo principal**

- Na criação, o sistema gera e revela a secret uma única vez.
- O operador solicita a rotação de um endpoint autorizado.
- O sistema cria uma nova secret e informa quando a anterior deixará de ser válida.
- A secret anterior permanece utilizável por 24 horas para transição.

**Fluxos alternativos e exceções**

- Eventos novos usam a secret nova imediatamente.
- Eventos anteriores podem usar a secret antiga durante a janela de transição.
- Depois da janela, entregas pendentes usam a secret atual.

**Erros previstos**

- webhook inexistente ou removido;
- usuário sem acesso ao cliente;
- falha ao gerar ou proteger a nova secret.

**Prioridade:** alta

---

#### RF-004 Registrar eventos de forma atômica

Toda mudança confirmada deve registrar as notificações aplicáveis na mesma transação do pedido.

**Fluxo principal**

- O sistema valida a mudança de status e as regras de estoque.
- Atualiza pedido, estoque e histórico.
- Cria um snapshot para cada endpoint interessado.
- Confirma todas as alterações em conjunto.

**Fluxos alternativos e exceções**

- Sem destino interessado, a transação confirma sem outbox.
- Qualquer falha na criação do evento reverte a mudança inteira.
- O snapshot permanece inalterado durante retry e replay.

**Erros previstos**

- transição de status inválida;
- estoque insuficiente;
- payload acima de 64 KB;
- falha de persistência.

**Prioridade:** alta

---

#### RF-005 Entregar eventos assinados

O worker deve enviar cada evento por HTTPS com payload e headers documentados.

**Fluxo principal**

- O worker seleciona o evento elegível mais antigo sem quebrar a ordem do pedido.
- Assina os bytes do JSON com HMAC-SHA256 e a secret aplicável.
- Envia `X-Event-Id`, `X-Webhook-Id`, `X-Timestamp` e `X-Signature`.
- Qualquer resposta `2xx` conclui a entrega.

**Fluxos alternativos e exceções**

- Pedidos diferentes podem ser processados concorrentemente pelo mesmo worker.
- Respostas `3xx` não são seguidas.
- Um claim abandonado volta a ser elegível após o lease.

**Erros previstos**

- timeout, falha de DNS ou conexão;
- destino inseguro ou certificado inválido;
- resposta HTTP diferente de `2xx`;
- falha interna antes de concluir o claim.

**Prioridade:** alta

---

#### RF-006 Repetir falhas e encaminhar à DLQ

Falhas de entrega devem seguir uma política limitada e terminar em uma DLQ persistida.

**Fluxo principal**

- Após a falha inicial, o sistema agenda retries em 1 min, 5 min, 30 min, 2 h e 12 h.
- Cada tentativa registra resultado, resposta limitada e duração.
- Se a sexta chamada falhar, o sistema marca o evento como falha definitiva e cria a DLQ.

**Fluxos alternativos e exceções**

- Uma resposta `2xx` em qualquer retry encerra o fluxo com sucesso.
- Eventos posteriores do mesmo pedido aguardam o evento anterior chegar a um estado terminal.
- Falhas de pedidos diferentes não bloqueiam umas às outras.

**Erros previstos**

- agendamento de retry não persistido;
- evento abandonado em processamento;
- criação inconsistente da DLQ.

**Prioridade:** alta

---

#### RF-007 Consultar histórico de entregas

O cliente deve consultar tentativas recentes de um endpoint autorizado.

**Fluxo principal**

- O usuário solicita as entregas de um webhook.
- O sistema valida o acesso ao cliente proprietário.
- Retorna tentativas paginadas da mais recente para a mais antiga.
- Cada item apresenta evento, tentativa, resultado, status HTTP, duração e resposta limitada.

**Fluxos alternativos e exceções**

- Resposta externa maior que o limite é truncada antes de ser armazenada.
- Secrets, assinaturas e headers sensíveis nunca aparecem no histórico.

**Erros previstos**

- webhook inexistente;
- usuário sem acesso;
- paginação inválida.

**Prioridade:** média

---

#### RF-008 Reprocessar eventos da DLQ

Um administrador deve poder recolocar um evento da DLQ na fila de entrega.

**Fluxo principal**

- O administrador seleciona uma entrada não reprocessada da DLQ.
- O sistema registra a identidade do administrador.
- O evento retorna a `PENDING` mantendo payload e `event_id`.
- O worker aplica novamente o fluxo normal de entrega.

**Fluxos alternativos e exceções**

- Uma nova falha final cria um novo ciclo de DLQ para o mesmo evento lógico.
- O replay é bloqueado se o endpoint não puder mais receber entregas.

**Erros previstos**

- usuário sem role `ADMIN`;
- entrada inexistente ou já reprocessada;
- endpoint removido ou inativo;
- falha na transação de replay.

**Prioridade:** alta

---

#### RF-009 Autorizar operações por cliente

Operadores devem acessar apenas recursos dos clientes aos quais estejam explicitamente vinculados.

**Fluxo principal**

- A API autentica o JWT e identifica usuário e role.
- Para cada operação, valida o vínculo com o cliente proprietário.
- Autoriza o operador vinculado ou um administrador.

**Fluxos alternativos e exceções**

- Administradores possuem acesso global.
- Operações por ID carregam o proprietário antes de autorizar a ação.

**Erros previstos**

- JWT ausente ou inválido;
- vínculo inexistente;
- recurso de outro cliente.

**Prioridade:** alta

---

#### RF-010 Permitir deduplicação pelo consumidor

O mesmo evento lógico deve manter uma identidade estável durante todo o ciclo de entrega.

**Fluxo principal**

- O sistema gera o `event_id` quando cria a outbox.
- Inclui o identificador no payload e no header `X-Event-Id`.
- Preserva o valor em retry, DLQ e replay.
- O consumidor usa esse ID para ignorar duplicatas.

**Fluxos alternativos e exceções**

- Uma queda após o processamento remoto pode provocar nova chamada com o mesmo ID.
- Endpoints diferentes recebem IDs próprios para a mesma mudança de pedido.

**Erros previstos**

- divergência entre header e payload;
- geração de novo ID durante retry ou replay.

**Prioridade:** alta

---

### Requisitos não funcionais

Performance

- p95 menor que 10 segundos entre o commit e o início da primeira tentativa;
- polling com intervalo default de 2 segundos;
- timeout total de 10 segundos por chamada;
- payload máximo de 65.536 bytes;
- batches pequenos e concorrência limitada para proteger o MySQL.

Disponibilidade

- meta mensal de 99,9% para API e processamento de webhooks, registrada como hipótese a validar;
- recuperação automática de eventos abandonados após expiração do lease;
- reinício do worker sem perda de eventos confirmados.

Segurança e autorização

- autenticação JWT obrigatória em todas as rotas de configuração e histórico;
- autorização por vínculo usuário-cliente e acesso global apenas para `ADMIN`;
- replay restrito a `ADMIN` e auditado;
- HTTPS obrigatório, HMAC-SHA256 por endpoint e secrets protegidas em repouso;
- redirects desabilitados e bloqueio de destinos privados ou reservados;
- secrets, assinaturas e tokens ausentes de logs e respostas não autorizadas.

Observabilidade

- logs estruturados com `event_id`, `webhook_id`, tentativa, duração e resultado;
- consultas operacionais para backlog, idade da outbox, leases vencidos e DLQ;
- correlação por `event_id` sem nova dependência de métricas ou tracing nesta fase.

Confiabilidade e integridade de dados

- pedido, estoque, histórico e outbox confirmam ou revertem juntos;
- entrega at-least-once com ID estável;
- retry persistido, claim atômico e lease recuperável;
- eventos posteriores do mesmo pedido não ultrapassam o evento anterior não terminal.

Compatibilidade e portabilidade

- manter Node.js 20, Express, TypeScript, Prisma, MySQL e APIs REST JSON sob `/api/v1`;
- preservar contratos e regras existentes de pedidos;
- executar API e worker com a mesma versão da aplicação e do schema.

Compliance

- manter trilha de tentativas, falhas definitivas e usuário responsável por cada replay;
- limitar dados de pedido ao necessário para o evento;
- definir retenção e expurgo antes de crescimento significativo das tabelas.

Acessibilidade no frontend consumidor

- não aplicável nesta entrega, pois não haverá interface visual;
- eventual dashboard será tratado em projeto separado com requisitos próprios de acessibilidade.

---

### Arquitetura e abordagem

Abordagem

- extensão do monólito modular existente com entrega assíncrona por outbox transacional;
- detalhes de arquitetura estão no [RFC](RFC.md) e de implementação no [FDD](FDD.md).

Componentes

- módulo `src/modules/webhooks` para configuração, publicação, entrega e histórico;
- MySQL como fonte de verdade da outbox, tentativas e DLQ;
- processo worker separado da API, com instância própria de PrismaClient;
- módulo de pedidos como origem das mudanças de status;
- middlewares e recursos compartilhados de autenticação, validação, erros e logs.

Integrações

- `OrderService.changeStatus` publica eventos dentro da transação Prisma atual;
- consumidores externos recebem HTTP JSON por HTTPS;
- portal de desenvolvedores publica payload, headers, assinatura e comportamento de duplicidade;
- não existem cache, broker externo ou fluxo inbound nesta fase.

### Decisões e trade-offs

#### Decisão: usar outbox no MySQL existente

- **Justificativa:** garante atomicidade com a mudança do pedido sem introduzir nova infraestrutura.
- **Trade-off:** aumenta gravações, índices, retenção e responsabilidade operacional do MySQL.

#### Decisão: executar worker separado com polling

- **Justificativa:** desacopla chamadas externas do ciclo da API e atende à meta de latência.
- **Trade-off:** adiciona consultas periódicas, supervisão própria e um ponto único de
  processamento.

#### Decisão: limitar retry e usar DLQ

- **Justificativa:** recupera indisponibilidades temporárias sem manter eventos pendentes para
  sempre.
- **Trade-off:** uma falha definitiva pode levar cerca de 15 horas e exigir replay manual.

#### Decisão: autenticar com HMAC-SHA256 por endpoint

- **Justificativa:** permite validar origem e integridade com tecnologia amplamente disponível.
- **Trade-off:** a plataforma precisa custodiar secrets recuperáveis e administrar rotação.

#### Decisão: oferecer entrega at-least-once

- **Justificativa:** evita perda quando o resultado de uma chamada é incerto.
- **Trade-off:** o consumidor deve armazenar `event_id` e deduplicar efeitos.

#### Decisão: iniciar com um único worker

- **Justificativa:** simplifica operação e ordenação por pedido no volume inicial.
- **Trade-off:** limita throughput e exige evolução antes de escalar horizontalmente.

#### Decisão: reutilizar padrões do projeto

- **Justificativa:** mantém organização, erros, logs, validação e testes consistentes.
- **Trade-off:** amplia a composição manual e o acoplamento ao Prisma no monólito.

#### Decisão: armazenar payload como snapshot

- **Justificativa:** preserva o estado do pedido no momento da mudança.
- **Trade-off:** aumenta armazenamento e duplica dados entre destinos.

---

### Dependências

#### Técnica: schema e migration do MySQL

A equipe de Pedidos deve entregar as tabelas, relações e índices de endpoints, acesso por cliente,
outbox, tentativas e DLQ antes de ativar o worker.

#### Técnica: criptografia e configuração operacional

Plataforma deve disponibilizar a chave de criptografia das secrets, variáveis do worker e processo
de implantação com a mesma versão da API e do schema.

#### Organizacional: revisão de segurança

Sofia deve revisar geração, armazenamento, rotação, assinatura, autorização e proteção contra SSRF.
A agenda deve reservar pelo menos dois dias úteis antes do deploy.

#### Organizacional: documentação para clientes

Marcos deve coordenar a publicação do contrato de payload, headers, assinatura, retry e duplicidade
no portal de desenvolvedores.

#### Organizacional: preparação operacional

A equipe de Plataforma deve disponibilizar runbook e consultas para backlog, leases vencidos, falhas
e DLQ antes da liberação.

#### Externa: endpoints e implementação dos consumidores

Cada cliente deve fornecer URL HTTPS, validar HMAC-SHA256, responder em até 10 segundos e deduplicar
eventos pelo `event_id`.

#### Externa: validação dos clientes iniciais

Atlas Comercial, MaxDistribuição e Nova Cargo devem validar o contrato e uma entrega ponta a ponta
antes da liberação ampla.

---

### Riscos e mitigação

#### Endpoint configurável permitir SSRF

- **Probabilidade:** alta
- **Impacto:** acesso a serviços internos ou exfiltração de dados.
- **Mitigação:**
  - aceitar apenas HTTPS e porta permitida;
  - bloquear loopback, redes privadas, link-local e faixas reservadas;
  - validar DNS antes de cada conexão e não seguir redirects;
  - revisar a implementação com Segurança.
- **Plano de contingência:** desativar o endpoint, cancelar pendências e investigar por
  `webhook_id`.

#### Consumidor aplicar o mesmo evento mais de uma vez

- **Probabilidade:** média
- **Impacto:** efeitos duplicados no sistema do cliente.
- **Mitigação:**
  - preservar `event_id` durante todo o ciclo;
  - destacar a semântica at-least-once na documentação;
  - validar deduplicação com os clientes iniciais.
- **Plano de contingência:** fornecer histórico do evento para reconciliação.

#### Backlog pressionar o MySQL

- **Probabilidade:** média
- **Impacto:** aumento da latência e degradação de transações da API.
- **Mitigação:**
  - usar índices compatíveis com claim e histórico;
  - limitar lote e concorrência;
  - acompanhar backlog e idade do evento mais antigo;
  - criar eventos somente para destinos interessados.
- **Plano de contingência:** reduzir concorrência ou pausar o worker e avaliar broker dedicado.

#### Worker único ficar indisponível

- **Probabilidade:** média
- **Impacto:** interrupção temporária das entregas e crescimento da outbox.
- **Mitigação:**
  - supervisionar o processo e reiniciá-lo automaticamente;
  - persistir todos os estados no MySQL;
  - recuperar claims após o lease.
- **Plano de contingência:** reiniciar o worker e acompanhar o esvaziamento do backlog.

#### Secret ser exposta

- **Probabilidade:** média
- **Impacto:** falsificação de notificações destinadas ao endpoint comprometido.
- **Mitigação:**
  - usar secret exclusiva e criptografada por endpoint;
  - revelar somente em criação e rotação;
  - aplicar redaction em logs;
  - realizar revisão de segurança antes do deploy.
- **Plano de contingência:** rotacionar a secret ou desativar o endpoint e auditar acessos.

#### Usuário acessar recurso de outro cliente

- **Probabilidade:** alta
- **Impacto:** exposição de dados e alteração indevida de destinos.
- **Mitigação:**
  - criar vínculo explícito entre usuário e cliente;
  - centralizar a autorização no serviço;
  - testar acesso cruzado em todas as rotas.
- **Plano de contingência:** suspender as rotas afetadas e auditar usuário, cliente e webhook.

#### Retry bloquear eventos posteriores do mesmo pedido

- **Probabilidade:** média
- **Impacto:** atraso de até a conclusão ou DLQ do evento anterior.
- **Mitigação:**
  - bloquear somente eventos do mesmo pedido;
  - permitir avanço de pedidos diferentes;
  - expor estado e idade no histórico.
- **Plano de contingência:** investigar o endpoint e aguardar o estado terminal antes do replay.

#### Mudança de status ficar mais lenta

- **Probabilidade:** baixa
- **Impacto:** maior latência e contenção na operação de pedidos.
- **Mitigação:**
  - consultar inscrições por índices;
  - inserir múltiplos eventos em lote;
  - manter chamadas externas fora da transação.
- **Plano de contingência:** revisar plano de execução, índices e quantidade de destinos por
  cliente.

#### Cliente implementar o contrato incorretamente

- **Probabilidade:** média
- **Impacto:** rejeição de assinaturas, duplicidade ou acúmulo de falhas.
- **Mitigação:**
  - publicar exemplos de assinatura e deduplicação;
  - oferecer ambiente de validação com os três clientes iniciais;
  - documentar claramente timeout, retries e códigos de sucesso.
- **Plano de contingência:** desativar temporariamente o endpoint e apoiar a correção da integração.

---

### Critérios de aceitação

Checklist objetivo que define se a feature está pronta.

- [ ] Cadastro, edição, remoção, listagem e rotação respeitam a autorização por cliente.
- [ ] Mudança de status e criação da outbox confirmam ou revertem juntas.
- [ ] Eventos são criados somente para endpoints ativos interessados no novo status.
- [ ] A primeira tentativa começa em menos de 10 segundos no p95.
- [ ] Toda entrega usa HTTPS, HMAC-SHA256, `X-Event-Id` e payload de até 64 KB.
- [ ] Retry usa os cinco intervalos definidos e a falha final segue para a DLQ.
- [ ] Replay exige `ADMIN`, preserva `event_id` e registra quem o executou.
- [ ] Histórico apresenta tentativa, resultado, resposta limitada e duração.
- [ ] Secrets não aparecem em listagens, históricos, erros ou logs.
- [ ] Operadores não acessam webhooks ou entregas de outros clientes.
- [ ] Eventos abandonados são recuperados após expiração do lease.
- [ ] Duplicatas mantêm o mesmo identificador no header e no payload.
- [ ] Os três clientes iniciais validam cadastro, assinatura, entrega e deduplicação.
- [ ] Contrato público e runbook operacional estão publicados.
- [ ] Revisão de segurança é concluída antes do deploy.
- [ ] `npm test`, `npm run lint` e `npm run build` passam.

---

### Testes e validação

Tipos de teste obrigatórios

- testes unitários para assinatura, criptografia, backoff, payload e validação de destinos;
- testes de integração com MySQL para atomicidade, claim, lease, retry, DLQ e replay;
- testes HTTP com Supertest para CRUD, autorização, validação e histórico;
- testes de segurança para SSRF, acesso cruzado, redaction e rotação de secret;
- testes de resiliência para timeout, conexão recusada, resposta não `2xx` e queda do worker;
- teste de carga controlado para latência de entrega e impacto no MySQL;
- testes de compatibilidade para preservar os contratos existentes de pedidos.

Estratégia de validação

- usar relógio falso e servidor HTTPS controlado para manter testes determinísticos;
- executar a suíte contra banco de teste real e serializado;
- medir o p95 entre commit e início da entrega sob carga representativa;
- realizar revisão exploratória de segurança com Sofia;
- executar homologação ponta a ponta com Atlas Comercial, MaxDistribuição e Nova Cargo;
- exigir `npm test`, `npm run lint` e `npm run build` antes da aprovação.
