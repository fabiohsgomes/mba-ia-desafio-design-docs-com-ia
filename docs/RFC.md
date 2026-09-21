# RFC: Webhooks de notificação de mudanças de status de pedidos

## Metadados

- **Autora:** Larissa (Tech Lead)
- **Status:** Em revisão
- **Data de elaboração:** 18 de setembro de 2026
- **Revisores:** Marcos (Product Manager), Bruno (Engenharia de Pedidos), Diego (Plataforma) e
  Sofia (Segurança)

## Resumo executivo

Propomos oferecer webhooks outbound para notificar clientes B2B quando seus pedidos mudarem de
status. A mudança do pedido e o evento serão gravados atomicamente em uma outbox no MySQL. Um
worker separado da API consultará essa outbox a cada dois segundos e fará as entregas HTTP com
assinatura HMAC-SHA256, retry limitado e DLQ. O contrato será at-least-once e usará um
`X-Event-Id` estável para deduplicação pelo consumidor.

A abordagem atende à expectativa de entrega em menos de dez segundos sem vincular a transação do
pedido à disponibilidade do cliente e sem adicionar Redis ou um broker nesta fase. A solução
reutilizará Prisma, MySQL, Pino, Zod, `AppError`, autenticação e a organização modular existentes.

## Contexto e problema

Clientes B2B hoje consultam repetidamente `GET /orders` para descobrir alterações. Esse polling é
lento e custoso para os consumidores, que consideram aceitável uma notificação recebida em até dez
segundos. O escopo é exclusivamente de eventos enviados pela plataforma; receber eventos dos
clientes não faz parte da proposta.

Uma chamada síncrona durante a mudança de status não é aceitável. Essa operação já atualiza pedido,
histórico e estoque em uma transação, e não deve depender da latência ou disponibilidade de um
sistema externo. Ao mesmo tempo, confirmar a mudança sem registrar a notificação criaria risco de
perda definitiva do evento.

## Proposta técnica

### Visão geral

Adotaremos uma outbox transacional no MySQL existente. Para cada endpoint ativo interessado no novo
status, a transação de mudança do pedido persistirá um evento com UUID e snapshot do payload. Assim,
commit e registro da notificação ocorrerão juntos; rollback descartará ambos.

Um processo Node separado da API fará polling da outbox a cada dois segundos. A primeira versão terá
um único worker, preservando a ordem por pedido enquanto os eventos forem processados pelos mais
antigos. Cada chamada terá timeout de dez segundos. A solução privilegia simplicidade operacional e
atendimento da meta inicial; escala horizontal e ordenação com múltiplos workers não fazem parte
desta fase.

Falhas transitórias serão reagendadas com backoff de 1 minuto, 5 minutos, 30 minutos, 2 horas e
12 horas. Após a tentativa inicial e cinco retentativas, o evento seguirá para uma DLQ persistida em
tabela separada. Um endpoint administrativo, restrito à role `ADMIN` e auditado, permitirá replay.

Cada configuração de webhook terá uma secret exclusiva. O corpo JSON será assinado com HMAC-SHA256,
e apenas URLs HTTPS serão aceitas. A rotação manterá a credencial anterior válida por 24 horas. As
entregas terão semântica at-least-once: `X-Event-Id` permanecerá estável entre tentativas e replay,
permitindo que o consumidor elimine duplicatas.

O domínio seguirá a organização atual em `src/modules/webhooks/`, enquanto o worker terá entry point
próprio. A integração com pedidos será limitada à publicação do evento dentro da transação ativa;
transporte HTTP, assinatura, retry e DLQ permanecerão no módulo de webhooks. Contratos de payload,
modelagem, algoritmos de claim e detalhes dos endpoints serão especificados no [FDD](FDD.md).

### Escopo desta fase

- CRUD autenticado de configurações e filtros por status;
- histórico de entregas por webhook;
- entrega assinada, retry, DLQ e replay administrativo;
- documentação do contrato de deduplicação para os clientes.

Não estão incluídos dashboard visual, alerta por e-mail nem infraestrutura de mensageria dedicada.

## Alternativas consideradas

### Chamada HTTP síncrona na mudança de status

Foi descartada porque faria a duração e o sucesso da transação dependerem do endpoint do cliente.
Embora reduzisse componentes, ampliaria latência e poderia bloquear ou reverter uma mudança válida
por uma falha externa.

### Redis Streams ou broker dedicado

Ofereceria consumo reativo e uma trajetória mais direta para escala horizontal. Foi descartado nesta
fase porque adicionaria infraestrutura e operação para um volume ainda não demonstrado e não
resolveria, sozinho, a atomicidade entre a alteração no MySQL e a publicação da mensagem.

### Trigger no MySQL para despertar o consumidor

Reduziria o polling em tese, mas uma trigger executa SQL e não notifica com segurança um processo
Node externo. A integração necessária seria improvisada e mais frágil que polling de dois segundos.

### Entrega exactly-once

Evitaria a responsabilidade de deduplicação no cliente, mas exigiria coordenação entre sistemas
independentes que HTTP não oferece. Preferimos at-least-once com identificador estável, aceitando
duplicatas como trade-off para não perder eventos em falhas incertas.

## Questões em aberto

- **Rate limiting por cliente:** o volume inicial será observado antes de definir limites, filas por
  destino ou controle de rajadas.
- **Escala e ordenação:** múltiplos workers exigirão particionamento por pedido ou outro mecanismo
  de coordenação; por ora, a garantia depende de um único worker.
- **Retenção:** a reunião sugeriu arquivar entregas após cerca de 30 dias, mas período, processo de
  expurgo e requisitos de auditoria ainda precisam ser definidos.
- **Notificação de endpoint degradado:** alertas por e-mail após falhas consecutivas foram adiados
  para uma fase futura, após medição do impacto.
- **Rotação de secret:** o contrato precisa esclarecer qual chave assina durante as 24 horas de
  convivência e como o consumidor distingue ou valida as duas credenciais.

## Impacto e riscos

### Impactos esperados

- Clientes deixam de depender de polling frequente e recebem mudanças em poucos segundos em
  condições normais.
- A transação de pedidos passa a realizar consultas e gravações adicionais para a outbox.
- API e worker terão ciclos de implantação e observabilidade separados, embora compartilhem código,
  banco e padrões operacionais.
- Consumidores precisarão validar assinaturas e armazenar `event_id` para deduplicação.

### Riscos e mitigações

- **Backlog ou indisponibilidade do worker:** monitorar idade e volume da outbox; manter claim
  atômico e recuperação de eventos abandonados.
- **Duplicidade de efeitos no cliente:** documentar explicitamente at-least-once e preservar o
  mesmo `X-Event-Id` em retry, DLQ e replay.
- **Vazamento de secrets:** usar uma credencial por endpoint, protegê-la em repouso, omiti-la de
  listagens e aplicar redaction nos logs; a implementação terá revisão de Segurança antes do deploy.
- **Abuso de destinos HTTP:** exigir HTTPS é necessário, mas insuficiente; controles contra SSRF,
  redirects maliciosos e DNS rebinding deverão ser tratados antes da liberação.
- **Crescimento das tabelas operacionais:** indexar as consultas do worker e definir política de
  retenção antes que o volume comprometa o MySQL.

## Decisões relacionadas

- [ADR-001](adrs/ADR-001-outbox-no-mysql.md) — Outbox no MySQL
- [ADR-002](adrs/ADR-002-worker-separado-com-polling.md) — Worker separado com polling
- [ADR-003](adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md) — Retry com backoff e DLQ
- [ADR-004](adrs/ADR-004-autenticacao-hmac-sha256-por-endpoint.md) — HMAC-SHA256 por endpoint
- [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md) — Entrega at-least-once
- [ADR-006](adrs/ADR-006-reuso-dos-padroes-arquiteturais-existentes.md) — Reuso dos padrões
