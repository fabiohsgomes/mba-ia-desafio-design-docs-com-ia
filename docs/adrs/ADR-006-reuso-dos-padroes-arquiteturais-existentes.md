# ADR-006: Reutilizar os padrões arquiteturais existentes no módulo de webhooks

## Status

Proposto — houve consenso na reunião técnica, mas a implementação ainda não existe.

## Contexto

A API organiza cada domínio em `controller`, `service`, `repository`, `routes` e `schemas`. A
composição manual ocorre em `src/app.ts`, e `src/routes/index.ts` monta os routers sob `/api/v1`.
Validação, autenticação, autorização, erros, banco e logs já possuem implementações compartilhadas.

O fluxo mais sensível da feature intercepta a transação existente em
`src/modules/orders/order.service.ts`. Criar padrões paralelos para webhooks aumentaria a carga de
manutenção e produziria respostas e observabilidade inconsistentes.

## Decisão

O domínio será implementado em `src/modules/webhooks/`, com os mesmos papéis de arquivos dos demais
módulos. Controllers adaptarão HTTP, services concentrarão regras, repositories acessarão Prisma,
routes aplicarão middleware e schemas Zod validarão entradas.

A solução reutilizará:

- `AppError` e o middleware central, com códigos de domínio prefixados por `WEBHOOK_`;
- `authenticate` no CRUD e `requireRole('ADMIN')` no replay da DLQ;
- Pino para logs estruturados da API e do worker;
- Prisma/MySQL e as convenções de UUID, relações, índices e nomes físicos;
- composição explícita em `buildControllers` e `buildApiRouter`;
- o padrão de testes de integração com Vitest, Supertest e banco real.

A integração com pedidos será uma capacidade estreita, como
`publishWebhookEvent(tx, order, fromStatus, toStatus)`, que recebe a transação ativa. O módulo de
pedidos não conhecerá HTTP, assinatura, retry ou DLQ. O worker terá entry point separado, mas sua
lógica de domínio permanecerá no módulo de webhooks.

## Alternativas Consideradas

### Criar um microsserviço de webhooks

Isolaria implantação e escala, mas duplicaria autenticação, contratos, observabilidade e acesso a
dados, além de reintroduzir o problema de publicação atômica entre sistemas.

### Implementar toda a lógica dentro do módulo de pedidos

Reduziria arquivos inicialmente, mas acoplaria mudança de status a transporte, segurança e política
operacional de entrega.

### Introduzir um framework interno ou container de injeção de dependências

Poderia melhorar composição futura, porém não é necessário para uma feature e criaria uma segunda
mudança arquitetural sem benefício imediato.

## Consequências

### Positivas

- A feature mantém contratos, testes e organização familiares ao projeto.
- Infraestrutura transversal já testada é reaproveitada.
- A fronteira estreita preserva o foco do módulo de pedidos.
- Novos erros e rotas mantêm comportamento consistente com a API existente.

### Negativas

- A composição manual em `src/app.ts` cresce com novas dependências.
- O worker compartilha código com a API e exige atenção para não importar bootstrap HTTP.
- Repositories e services continuam acoplados aos tipos gerados pelo Prisma.
- O padrão atual não resolve sozinho multitenancy, custódia de secrets ou concorrência do worker.
