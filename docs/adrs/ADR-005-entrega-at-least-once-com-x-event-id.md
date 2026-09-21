# ADR-005: Garantir entrega at-least-once com X-Event-Id

## Status

Proposto — houve consenso na reunião técnica, mas a implementação ainda não existe.

## Contexto

Não existe transação distribuída entre o MySQL da plataforma e o sistema HTTP do cliente. O cliente
pode processar um evento enquanto a resposta se perde, ou o worker pode cair antes de registrar o
sucesso. Nesses casos, evitar nova tentativa perderia eventos; tentar novamente pode produzir
duplicatas.

## Decisão

O sistema oferecerá semântica at-least-once. Um UUID `event_id` será criado quando o evento entrar
na outbox e permanecerá inalterado no payload e no header `X-Event-Id` durante todas as tentativas,
passagem pela DLQ e replay administrativo.

O cliente será responsável por deduplicar eventos usando esse identificador. O payload também
conterá `event_id`, `event_type: order.status_changed`, timestamp, pedido, cliente e estados
anterior e novo. `X-Webhook-Id` identificará a configuração de destino.

O contrato público deverá declarar explicitamente que duplicatas são esperadas e que uma resposta
de sucesso não constitui uma garantia de processamento exatamente uma vez no consumidor.

## Alternativas Consideradas

### Exactly-once

Exigiria coordenação, protocolo ou armazenamento compartilhado com cada consumidor. Não é viável
garantir essa semântica apenas com HTTP entre sistemas independentes.

### At-most-once

Evitaria duplicatas ao não repetir entregas incertas, mas permitiria perda silenciosa de eventos.

### Gerar novo ID em cada retry

Simplificaria o envio, porém impediria o cliente de reconhecer que se trata do mesmo evento lógico.

## Consequências

### Positivas

- Falhas incertas podem ser repetidas sem abandonar o evento.
- Um identificador estável permite deduplicação determinística no consumidor.
- Retry e replay compartilham o mesmo contrato de identidade.

### Negativas

- Clientes precisam manter uma store ou janela de IDs já processados.
- Consumidores que ignorarem o contrato poderão aplicar o mesmo efeito mais de uma vez.
- A plataforma não consegue afirmar exactly-once de ponta a ponta.
- Retenção do identificador no cliente precisa ser compatível com a janela de retry e replay.
