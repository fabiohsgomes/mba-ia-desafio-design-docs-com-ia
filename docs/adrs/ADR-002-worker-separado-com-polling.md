# ADR-002: Executar entregas em worker separado com polling

## Status

Proposto — houve consenso na reunião técnica, mas a implementação ainda não existe.

## Contexto

Eventos persistidos na outbox precisam ser entregues sem bloquear a API. O requisito considera
tempo real uma entrega em menos de dez segundos em condições normais. O MySQL não oferece um
mecanismo equivalente ao `LISTEN/NOTIFY` do PostgreSQL, e a adoção de nova infraestrutura foi
descartada para a primeira versão.

Executar o consumidor dentro de `src/server.ts` ligaria seu ciclo de vida ao servidor HTTP e
dificultaria implantar, reiniciar e observar cada carga de trabalho de forma independente.

## Decisão

Criaremos `src/worker.ts` como entry point de um processo Node separado da API. O processo usará a
mesma `DATABASE_URL`, mas criará sua própria instância de `PrismaClient`, assim como cada processo
deve possuir seu próprio pool de conexões.

O worker consultará, a cada dois segundos, um lote pequeno dos eventos elegíveis mais antigos. A
lógica de processamento ficará em `src/modules/webhooks/`, e o entry point cuidará apenas de
bootstrap, sinais de encerramento, Prisma e logging. Cada chamada HTTP terá timeout de dez segundos.

A primeira versão operará com uma única instância ativa do worker. Antes do envio, o evento deverá
ser reivindicado atomicamente; registros abandonados em processamento por queda do processo deverão
voltar a ser elegíveis mediante lease ou timeout.

## Alternativas Consideradas

### Worker embutido no processo da API

Rejeitado porque replica consumidores quando a API escala e compartilha falhas e recursos com o
tráfego HTTP.

### Redis, fila gerenciada ou notificações externas

Permitiriam consumo mais reativo e escalável, mas introduziriam operação e custo não justificados
pelo volume inicial.

### Trigger no MySQL

Uma trigger executa SQL, mas não desperta de forma segura um processo Node externo.

## Consequências

### Positivas

- API e entrega podem ser implantadas, reiniciadas e dimensionadas separadamente.
- O polling de dois segundos atende a meta de latência sem infraestrutura adicional.
- Falhas de endpoints externos não bloqueiam requisições de pedidos.
- O worker reutiliza Prisma e Pino, reduzindo variação tecnológica.

### Negativas

- Polling adiciona até aproximadamente dois segundos de espera e consultas periódicas ao banco.
- Uma única instância limita throughput e constitui um ponto operacional de falha.
- Timeout, backlog e processamento sequencial podem exceder a meta de dez segundos.
- Claim, lease, shutdown e supervisão do processo aumentam a complexidade operacional.
