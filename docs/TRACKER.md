# Tracker de Rastreabilidade

Este tracker relaciona requisitos, decisões, restrições, alternativas, riscos e refinamentos de
implementação dos documentos da feature às evidências disponíveis na transcrição ou no código.
Quando uma linha representa detalhamento técnico derivado, a fonte aponta para a decisão ou para o
padrão existente que motivou esse detalhamento, sem afirmar que a implementação já existe.

## Matriz de rastreabilidade

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-CTX-01 | `docs/PRD.md` | Problema | Clientes B2B consultam `GET /orders` repetidamente para detectar mudanças. | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-02 | `docs/PRD.md` | Público-alvo | Atlas Comercial, MaxDistribuição e Nova Cargo solicitaram notificações. | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-03 | `docs/PRD.md` | Risco de negócio | A Atlas indicou risco de migrar para um concorrente sem a feature. | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-04 | `docs/PRD.md` | Restrição de escopo | A solução trata somente webhooks outbound. | TRANSCRICAO | [09:02] Marcos |
| PRD-CTX-05 | `docs/PRD.md` | Restrição técnica | A entrega não pode bloquear a mudança de status do pedido. | TRANSCRICAO | [09:04] Bruno |
| PRD-OBJ-01 | `docs/PRD.md` | Objetivo mensurável | Iniciar a primeira entrega em menos de 10 segundos no p95. | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | `docs/PRD.md` | Objetivo | Registrar atomicamente toda notificação com destino interessado. | TRANSCRICAO | [09:06] Diego |
| PRD-OBJ-03 | `docs/PRD.md` | Objetivo | Identificar e assinar todas as entregas HTTPS. | TRANSCRICAO | [09:20] Sofia |
| PRD-OBJ-04 | `docs/PRD.md` | Objetivo | Atender os três clientes solicitantes na entrega inicial. | TRANSCRICAO | [09:00] Marcos |
| PRD-SCP-01 | `docs/PRD.md` | Escopo | CRUD de endpoints e filtros de status por cliente. | TRANSCRICAO | [09:31] Marcos |
| PRD-SCP-02 | `docs/PRD.md` | Escopo | Secret exclusiva por endpoint e rotação com janela de 24 horas. | TRANSCRICAO | [09:21] Sofia |
| PRD-SCP-03 | `docs/PRD.md` | Escopo | Worker separado com polling a cada dois segundos. | TRANSCRICAO | [09:10] Larissa |
| PRD-SCP-04 | `docs/PRD.md` | Escopo | Histórico de entregas por webhook. | TRANSCRICAO | [09:34] Marcos |
| PRD-SCP-05 | `docs/PRD.md` | Escopo | Replay administrativo da DLQ com auditoria. | TRANSCRICAO | [09:36] Sofia |
| PRD-OOS-01 | `docs/PRD.md` | Fora de escopo | Chamada HTTP síncrona na mudança de status foi descartada. | TRANSCRICAO | [09:06] Diego |
| PRD-OOS-02 | `docs/PRD.md` | Fora de escopo | Redis Streams ou broker dedicado foram descartados nesta fase. | TRANSCRICAO | [09:07] Diego |
| PRD-OOS-03 | `docs/PRD.md` | Fora de escopo | Trigger MySQL para acionar o worker foi descartada. | TRANSCRICAO | [09:09] Diego |
| PRD-OOS-04 | `docs/PRD.md` | Fora de escopo | Retry indefinido foi descartado. | TRANSCRICAO | [09:15] Diego |
| PRD-OOS-05 | `docs/PRD.md` | Fora de escopo | Garantia exactly-once foi descartada. | TRANSCRICAO | [09:25] Diego |
| PRD-OOS-06 | `docs/PRD.md` | Fora de escopo | Múltiplos workers e particionamento foram adiados. | TRANSCRICAO | [09:13] Diego |
| PRD-OOS-07 | `docs/PRD.md` | Fora de escopo | Ordenação global não é uma garantia desta fase. | TRANSCRICAO | [09:13] Larissa |
| PRD-OOS-08 | `docs/PRD.md` | Fora de escopo | Rate limiting por cliente será decidido após observação. | TRANSCRICAO | [09:39] Larissa |
| PRD-OOS-09 | `docs/PRD.md` | Fora de escopo | Alerta por e-mail foi adiado para uma fase futura. | TRANSCRICAO | [09:37] Larissa |
| PRD-OOS-10 | `docs/PRD.md` | Fora de escopo | Dashboard visual pertence a um projeto separado. | TRANSCRICAO | [09:40] Larissa |
| PRD-OOS-11 | `docs/PRD.md` | Fora de escopo | Arquivamento automático após cerca de 30 dias foi adiado. | TRANSCRICAO | [09:08] Diego |
| PRD-RF-01 | `docs/PRD.md` | Requisito Funcional | RF-001 permite criar, listar, alterar e remover webhooks. | TRANSCRICAO | [09:33] Bruno |
| PRD-RF-02 | `docs/PRD.md` | Requisito Funcional | RF-002 filtra eventos pelo novo status na inserção da outbox. | TRANSCRICAO | [09:34] Bruno |
| PRD-RF-03 | `docs/PRD.md` | Requisito Funcional | RF-003 gera secret por endpoint e permite rotação. | TRANSCRICAO | [09:21] Sofia |
| PRD-RF-04 | `docs/PRD.md` | Requisito Funcional | RF-004 grava pedido, histórico, estoque e outbox atomicamente. | TRANSCRICAO | [09:40] Bruno |
| PRD-RF-05 | `docs/PRD.md` | Requisito Funcional | RF-005 envia JSON assinado por HTTPS com headers definidos. | TRANSCRICAO | [09:44] Diego |
| PRD-RF-06 | `docs/PRD.md` | Requisito Funcional | RF-006 aplica cinco retries após a chamada inicial, com no máximo seis chamadas. | TRANSCRICAO | [09:17] Larissa |
| PRD-RF-07 | `docs/PRD.md` | Requisito Funcional | RF-007 expõe histórico de entregas com resultado e duração. | TRANSCRICAO | [09:34] Marcos |
| PRD-RF-08 | `docs/PRD.md` | Requisito Funcional | RF-008 permite replay manual da DLQ por administrador. | TRANSCRICAO | [09:18] Diego |
| PRD-RF-09 | `docs/PRD.md` | Requisito Funcional | RF-009 restringe replay à role `ADMIN`. | TRANSCRICAO | [09:36] Sofia |
| PRD-RF-10 | `docs/PRD.md` | Requisito Funcional | RF-010 preserva `event_id` para deduplicação pelo consumidor. | TRANSCRICAO | [09:25] Diego |
| PRD-RNF-01 | `docs/PRD.md` | Requisito Não Funcional | Polling tem intervalo padrão de dois segundos. | TRANSCRICAO | [09:09] Diego |
| PRD-RNF-02 | `docs/PRD.md` | Requisito Não Funcional | Cada chamada externa tem timeout de dez segundos. | TRANSCRICAO | [09:42] Diego |
| PRD-RNF-03 | `docs/PRD.md` | Requisito Não Funcional | O payload é limitado a 64 KB e deve falhar se exceder o teto. | TRANSCRICAO | [09:24] Larissa |
| PRD-RNF-04 | `docs/PRD.md` | Segurança | URLs de webhook devem usar HTTPS. | TRANSCRICAO | [09:23] Sofia |
| PRD-RNF-05 | `docs/PRD.md` | Segurança | O corpo é autenticado com HMAC-SHA256. | TRANSCRICAO | [09:22] Sofia |
| PRD-RNF-06 | `docs/PRD.md` | Confiabilidade | Entrega é at-least-once e pode produzir duplicatas. | TRANSCRICAO | [09:24] Diego |
| PRD-RNF-07 | `docs/PRD.md` | Integridade | Pedido e evento confirmam ou revertem juntos. | TRANSCRICAO | [09:41] Diego |
| PRD-RNF-08 | `docs/PRD.md` | Compatibilidade | A feature reutiliza Node, Prisma, MySQL, Pino, Zod e padrões atuais. | TRANSCRICAO | [09:30] Larissa |
| PRD-DEC-01 | `docs/PRD.md` | Decisão | Usar outbox no MySQL existente. | TRANSCRICAO | [09:08] Larissa |
| PRD-DEC-02 | `docs/PRD.md` | Decisão | Executar um worker separado em polling. | TRANSCRICAO | [09:11] Diego |
| PRD-DEC-03 | `docs/PRD.md` | Decisão | Limitar retries e persistir uma DLQ separada. | TRANSCRICAO | [09:18] Diego |
| PRD-DEC-04 | `docs/PRD.md` | Decisão | Usar HMAC-SHA256 com secret por endpoint. | TRANSCRICAO | [09:22] Sofia |
| PRD-DEC-05 | `docs/PRD.md` | Decisão | Oferecer at-least-once em vez de exactly-once. | TRANSCRICAO | [09:26] Larissa |
| PRD-DEC-06 | `docs/PRD.md` | Decisão | Começar com uma única instância do worker. | TRANSCRICAO | [09:12] Diego |
| PRD-DEC-07 | `docs/PRD.md` | Decisão | Reutilizar a arquitetura modular e recursos compartilhados. | TRANSCRICAO | [09:30] Larissa |
| PRD-DEC-08 | `docs/PRD.md` | Decisão | Persistir o payload como snapshot na inserção. | TRANSCRICAO | [09:52] Larissa |
| PRD-DEP-01 | `docs/PRD.md` | Dependência | Plataforma deve operar API e worker sobre o mesmo banco. | TRANSCRICAO | [09:11] Bruno |
| PRD-DEP-02 | `docs/PRD.md` | Dependência | Segurança precisa de ao menos dois dias úteis para revisão. | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-03 | `docs/PRD.md` | Dependência externa | Consumidores devem deduplicar por `event_id`. | TRANSCRICAO | [09:25] Diego |
| PRD-DEP-04 | `docs/PRD.md` | Dependência externa | Marcos documentará o contrato no portal de desenvolvedores. | TRANSCRICAO | [09:26] Marcos |
| PRD-RSK-01 | `docs/PRD.md` | Risco | Duplicatas podem gerar efeitos repetidos no consumidor. | TRANSCRICAO | [09:25] Sofia |
| PRD-RSK-02 | `docs/PRD.md` | Risco | Acúmulo na outbox pode degradar o worker e pressionar o banco. | TRANSCRICAO | [09:07] Bruno |
| PRD-RSK-03 | `docs/PRD.md` | Risco | Worker único limita throughput e sua queda atrasa entregas. | TRANSCRICAO | [09:13] Diego |
| PRD-RSK-04 | `docs/PRD.md` | Risco | Vazamento de secret exige rotação isolada por endpoint. | TRANSCRICAO | [09:22] Diego |
| PRD-RSK-05 | `docs/PRD.md` | Risco | Inserções adicionais podem aumentar a duração de `changeStatus`. | TRANSCRICAO | [09:04] Bruno |
| PRD-CA-01 | `docs/PRD.md` | Critério de Aceitação | CRUD e rotação respeitam autorização do cliente. | TRANSCRICAO | [09:32] Larissa |
| PRD-CA-02 | `docs/PRD.md` | Critério de Aceitação | Outbox e mudança de status são atômicas. | TRANSCRICAO | [09:41] Diego |
| PRD-CA-03 | `docs/PRD.md` | Critério de Aceitação | Primeira tentativa ocorre em menos de dez segundos. | TRANSCRICAO | [09:02] Marcos |
| PRD-CA-04 | `docs/PRD.md` | Critério de Aceitação | Retry segue 1m, 5m, 30m, 2h e 12h antes da DLQ. | TRANSCRICAO | [09:17] Larissa |
| PRD-CA-05 | `docs/PRD.md` | Critério de Aceitação | Replay exige `ADMIN` e registra o autor. | TRANSCRICAO | [09:36] Sofia |
| PRD-TST-01 | `docs/PRD.md` | Estratégia de Teste | Preservar testes de integração com banco real e Supertest. | CODIGO | `tests/orders.test.ts` |
| PRD-TST-02 | `docs/PRD.md` | Estratégia de Teste | Reutilizar factories de autenticação, clientes, produtos e pedidos. | CODIGO | `tests/helpers/factories.ts` |
| PRD-TST-03 | `docs/PRD.md` | Estratégia de Teste | Limpar novas tabelas respeitando a ordem usada no setup atual. | CODIGO | `tests/setup.ts` |
| RFC-PROP-01 | `docs/RFC.md` | Proposta | Criar webhook outbound para mudanças de status de pedidos. | TRANSCRICAO | [09:02] Marcos |
| RFC-PROP-02 | `docs/RFC.md` | Proposta | Gravar evento em outbox na mesma transação do pedido. | TRANSCRICAO | [09:06] Diego |
| RFC-PROP-03 | `docs/RFC.md` | Proposta | Usar processo separado com polling de dois segundos. | TRANSCRICAO | [09:10] Larissa |
| RFC-PROP-04 | `docs/RFC.md` | Proposta | Aplicar retry limitado, DLQ e replay administrativo. | TRANSCRICAO | [09:18] Diego |
| RFC-PROP-05 | `docs/RFC.md` | Proposta | Assinar cada entrega com secret exclusiva do endpoint. | TRANSCRICAO | [09:21] Sofia |
| RFC-PROP-06 | `docs/RFC.md` | Proposta | Preservar ordem por pedido enquanto houver um worker. | TRANSCRICAO | [09:13] Larissa |
| RFC-ALT-01 | `docs/RFC.md` | Alternativa Descartada | Chamada HTTP síncrona acoplaria a transação ao cliente. | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | `docs/RFC.md` | Alternativa Descartada | Redis ou broker adicionaria infraestrutura prematuramente. | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-03 | `docs/RFC.md` | Alternativa Descartada | Trigger MySQL não acorda um processo externo com segurança. | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | `docs/RFC.md` | Alternativa Descartada | Exactly-once exigiria coordenação entre sistemas. | TRANSCRICAO | [09:25] Diego |
| RFC-OPEN-01 | `docs/RFC.md` | Questão em Aberto | Rate limiting será avaliado após observação do volume. | TRANSCRICAO | [09:39] Larissa |
| RFC-OPEN-02 | `docs/RFC.md` | Questão em Aberto | Escala com múltiplos workers exigirá particionamento ou locks. | TRANSCRICAO | [09:13] Diego |
| RFC-OPEN-03 | `docs/RFC.md` | Questão em Aberto | Retenção e arquivamento após cerca de 30 dias não foram definidos. | TRANSCRICAO | [09:08] Diego |
| RFC-OPEN-04 | `docs/RFC.md` | Questão em Aberto | Alertas por e-mail para endpoint degradado foram adiados. | TRANSCRICAO | [09:37] Larissa |
| RFC-OPEN-05 | `docs/RFC.md` | Questão em Aberto | Uso das duas chaves durante a rotação precisa ser detalhado. | TRANSCRICAO | [09:21] Sofia |
| RFC-RSK-01 | `docs/RFC.md` | Risco | Backlog ou indisponibilidade do worker atrasa entregas. | TRANSCRICAO | [09:07] Bruno |
| RFC-RSK-02 | `docs/RFC.md` | Risco | At-least-once transfere a deduplicação ao consumidor. | TRANSCRICAO | [09:25] Sofia |
| RFC-RSK-03 | `docs/RFC.md` | Risco | Secret vazada pode comprometer um destino. | TRANSCRICAO | [09:22] Diego |
| RFC-IMP-01 | `docs/RFC.md` | Impacto | `changeStatus` ganha consultas e inserções dentro da transação. | TRANSCRICAO | [09:40] Bruno |
| RFC-IMP-02 | `docs/RFC.md` | Impacto | API e worker possuem ciclos de vida separados. | TRANSCRICAO | [09:11] Diego |
| FDD-FLW-01 | `docs/FDD.md` | Fluxo | `changeStatus` publica o evento antes do commit da transação atual. | TRANSCRICAO | [09:40] Bruno |
| FDD-FLW-02 | `docs/FDD.md` | Fluxo | Somente endpoints ativos interessados no novo status geram outbox. | TRANSCRICAO | [09:34] Bruno |
| FDD-FLW-03 | `docs/FDD.md` | Fluxo | O worker lê eventos pendentes mais antigos em lotes pequenos. | TRANSCRICAO | [09:08] Diego |
| FDD-FLW-04 | `docs/FDD.md` | Fluxo | Resposta bem-sucedida entrega; falhas agendam retry ou DLQ. | TRANSCRICAO | [09:15] Diego |
| FDD-FLW-05 | `docs/FDD.md` | Fluxo | Eventos posteriores preservam ordem por pedido no worker único. | TRANSCRICAO | [09:12] Diego |
| FDD-FLW-06 | `docs/FDD.md` | Fluxo | Payload permanece como snapshot durante retry e replay. | TRANSCRICAO | [09:52] Larissa |
| FDD-RET-01 | `docs/FDD.md` | Regra de Retry | Uma chamada inicial é seguida por cinco retries, totalizando até seis chamadas. | TRANSCRICAO | [09:17] Larissa |
| FDD-RET-02 | `docs/FDD.md` | Regra de Retry | Backoff usa 1m, 5m, 30m, 2h e 12h. | TRANSCRICAO | [09:17] Diego |
| FDD-SEC-01 | `docs/FDD.md` | Segurança | HMAC usa SHA-256 sobre o corpo enviado. | TRANSCRICAO | [09:22] Sofia |
| FDD-SEC-02 | `docs/FDD.md` | Segurança | Cada endpoint possui uma secret única e rotacionável. | TRANSCRICAO | [09:21] Sofia |
| FDD-SEC-03 | `docs/FDD.md` | Segurança | A secret anterior permanece válida por 24 horas. | TRANSCRICAO | [09:21] Sofia |
| FDD-SEC-04 | `docs/FDD.md` | Segurança | URL sem HTTPS é rejeitada pela validação. | TRANSCRICAO | [09:23] Sofia |
| FDD-CTR-01 | `docs/FDD.md` | Contrato HTTP | Criar webhook usa `POST` com URL e filtros de status. | TRANSCRICAO | [09:31] Marcos |
| FDD-CTR-02 | `docs/FDD.md` | Contrato HTTP | Configurações oferecem `GET`, `PATCH` e `DELETE`. | TRANSCRICAO | [09:33] Bruno |
| FDD-CTR-03 | `docs/FDD.md` | Contrato HTTP | Histórico usa `GET /webhooks/:id/deliveries`. | TRANSCRICAO | [09:34] Marcos |
| FDD-CTR-04 | `docs/FDD.md` | Contrato HTTP | Replay usa `POST /admin/webhooks/dead-letter/:id/replay`. | TRANSCRICAO | [09:18] Diego |
| FDD-CTR-05 | `docs/FDD.md` | Contrato Outbound | Payload contém tipo, timestamp, pedido, cliente, status e total. | TRANSCRICAO | [09:43] Diego |
| FDD-CTR-06 | `docs/FDD.md` | Contrato Outbound | Headers incluem `X-Event-Id`, `X-Signature` e `X-Timestamp`. | TRANSCRICAO | [09:44] Diego |
| FDD-CTR-07 | `docs/FDD.md` | Contrato Outbound | Header `X-Webhook-Id` identifica o endpoint cadastrado. | TRANSCRICAO | [09:44] Sofia |
| FDD-CTR-08 | `docs/FDD.md` | Contrato Outbound | O payload omite itens para permanecer enxuto. | TRANSCRICAO | [09:43] Diego |
| FDD-ERR-01 | `docs/FDD.md` | Padrão de Erro | Erros do módulo usam prefixo `WEBHOOK_`. | TRANSCRICAO | [09:29] Larissa |
| FDD-ERR-02 | `docs/FDD.md` | Padrão de Erro | Erros específicos estendem o padrão `AppError`. | TRANSCRICAO | [09:28] Bruno |
| FDD-OBS-01 | `docs/FDD.md` | Observabilidade | API e worker reutilizam Pino para logs estruturados. | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-02 | `docs/FDD.md` | Observabilidade | Histórico registra sucesso, falha, resposta e tempo de resposta. | TRANSCRICAO | [09:34] Marcos |
| FDD-INT-01 | `docs/FDD.md` | Integração com Código | Estender `OrderService.changeStatus` dentro da transação Prisma. | CODIGO | `src/modules/orders/order.service.ts` |
| FDD-INT-02 | `docs/FDD.md` | Integração com Código | Montar dependências do módulo na composição manual existente. | CODIGO | `src/app.ts` |
| FDD-INT-03 | `docs/FDD.md` | Integração com Código | Registrar rotas do módulo sob o prefixo `/api/v1`. | CODIGO | `src/routes/index.ts` |
| FDD-INT-04 | `docs/FDD.md` | Integração com Código | Reutilizar `authenticate` e `requireRole('ADMIN')`. | CODIGO | `src/middlewares/auth.middleware.ts` |
| FDD-INT-05 | `docs/FDD.md` | Integração com Código | Reutilizar validação de schemas Zod em body, params e query. | CODIGO | `src/middlewares/validate.middleware.ts` |
| FDD-INT-06 | `docs/FDD.md` | Integração com Código | Criar erros `WEBHOOK_` como subclasses de `AppError`. | CODIGO | `src/shared/errors/app-error.ts` |
| FDD-INT-07 | `docs/FDD.md` | Integração com Código | Deixar o middleware central tratar `AppError`, Zod e Prisma. | CODIGO | `src/middlewares/error.middleware.ts` |
| FDD-INT-08 | `docs/FDD.md` | Integração com Código | Ampliar redaction e reutilizar a configuração Pino. | CODIGO | `src/shared/logger/index.ts` |
| FDD-INT-09 | `docs/FDD.md` | Integração com Código | Worker cria sua própria instância usando a factory Prisma. | CODIGO | `src/config/database.ts` |
| FDD-INT-10 | `docs/FDD.md` | Integração com Código | Novo worker segue o bootstrap e shutdown do servidor. | CODIGO | `src/server.ts` |
| FDD-INT-11 | `docs/FDD.md` | Integração com Código | Novos modelos e relações serão adicionados ao schema atual. | CODIGO | `prisma/schema.prisma` |
| FDD-INT-12 | `docs/FDD.md` | Integração com Código | Testes devem preservar os comportamentos atuais dos pedidos. | CODIGO | `tests/orders.test.ts` |
| FDD-LIM-01 | `docs/FDD.md` | Restrição | Worker e API usam o mesmo MySQL e Prisma por processos separados. | TRANSCRICAO | [09:30] Bruno |
| FDD-LIM-02 | `docs/FDD.md` | Restrição | Limite máximo do payload é 65.536 bytes. | TRANSCRICAO | [09:24] Larissa |
| FDD-LIM-03 | `docs/FDD.md` | Restrição | Timeout total de cada envio é dez segundos. | TRANSCRICAO | [09:42] Diego |
| FDD-LIM-04 | `docs/FDD.md` | Restrição | O replay é protegido por role `ADMIN`. | TRANSCRICAO | [09:36] Sofia |
| FDD-CMP-01 | `docs/FDD.md` | Compatibilidade | `PATCH /orders/:id/status` mantém as regras de negócio atuais. | CODIGO | `src/modules/orders/order.routes.ts` |
| FDD-DAT-01 | `docs/FDD.md` | Modelo de Dados | Outbox usa UUID como o restante do projeto. | TRANSCRICAO | [09:51] Larissa |
| FDD-DAT-02 | `docs/FDD.md` | Modelo de Dados | DLQ fica em tabela separada com payload, motivo e timestamp. | TRANSCRICAO | [09:18] Diego |
| FDD-DAT-03 | `docs/FDD.md` | Modelo de Dados | Configuração associa URL, secret, cliente e estado ativo. | TRANSCRICAO | [09:21] Bruno |
| ADR-001 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Decisão Arquitetural | Persistir uma outbox transacional no MySQL existente. | TRANSCRICAO | [09:08] Larissa |
| ADR-002 | `docs/adrs/ADR-002-worker-separado-com-polling.md` | Decisão Arquitetural | Executar entregas em worker separado com polling de dois segundos. | TRANSCRICAO | [09:11] Diego |
| ADR-003 | `docs/adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md` | Decisão Arquitetural | Usar backoff limitado, DLQ separada e replay administrativo. | TRANSCRICAO | [09:18] Diego |
| ADR-004 | `docs/adrs/ADR-004-autenticacao-hmac-sha256-por-endpoint.md` | Decisão Arquitetural | Autenticar entregas com HMAC-SHA256 e secret por endpoint. | TRANSCRICAO | [09:22] Sofia |
| ADR-005 | `docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md` | Decisão Arquitetural | Oferecer at-least-once com `X-Event-Id` estável. | TRANSCRICAO | [09:26] Larissa |
| ADR-006 | `docs/adrs/ADR-006-reuso-dos-padroes-arquiteturais-existentes.md` | Decisão Arquitetural | Reutilizar módulos, erros, middleware, Pino, Zod e Prisma existentes. | TRANSCRICAO | [09:30] Larissa |

## Indicadores de cobertura

- Itens rastreados: 138.
- Linhas com fonte `TRANSCRICAO`: 122 de 138, ou 88,4%.
- Linhas com fonte `CODIGO`: 16 de 138, distribuídas por 15 caminhos reais.
- Documentos cobertos: PRD, RFC, FDD e os seis ADRs.
- Cobertura documental estimada: superior a 80% dos itens identificáveis. Itens repetitivos de
  mitigação, exemplos de payload e detalhes puramente ilustrativos foram consolidados na decisão,
  requisito ou contrato correspondente.
