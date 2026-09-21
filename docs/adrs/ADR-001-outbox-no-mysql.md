# ADR-001: Persistir eventos de webhook em uma outbox no MySQL

## Status

Proposto — houve consenso na reunião técnica, mas a implementação ainda não existe.

## Contexto

Uma mudança de status atualiza o pedido, registra o histórico e pode alterar estoque dentro da
mesma transação Prisma em `src/modules/orders/order.service.ts`. A chamada HTTP ao cliente não pode
participar dessa transação: lentidão ou indisponibilidade externa aumentaria a latência da API e
poderia provocar rollback de uma mudança de negócio válida.

Também é necessário impedir o estado em que a mudança de status é confirmada sem que a notificação
correspondente tenha sido registrada. O projeto já utiliza MySQL e Prisma, e a primeira versão não
deve introduzir Redis ou outro broker.

## Decisão

Adotaremos o padrão Transactional Outbox no MySQL existente.

Ao alterar o status, a aplicação consultará os webhooks ativos do cliente interessados no novo
status e gravará um evento por destino dentro da mesma transação que atualiza `orders`, estoque e
`order_status_history`. Uma operação do módulo de webhooks receberá o
`Prisma.TransactionClient` já aberto por `OrderService.changeStatus`; ela não abrirá uma transação
independente.

Cada evento terá UUID, referência ao webhook e ao pedido, estado de processamento, datas de criação
e próxima tentativa, além do payload JSON já renderizado. O snapshot preservará o estado observado
no momento da transição, mesmo que o pedido mude novamente antes do envio. A consulta do worker
terá índices compatíveis com estado, próxima tentativa e ordem de criação.

## Alternativas Consideradas

### Chamada HTTP síncrona em `OrderService`

Rejeitada porque acopla disponibilidade e tempo de resposta do cliente à transação do pedido.

### Publicação em Redis Streams ou broker dedicado

Ofereceria mecanismos próprios de consumo e escala, mas adicionaria infraestrutura e ainda exigiria
uma solução para atomicidade entre o commit do MySQL e a publicação no broker.

### Gravar o evento depois do commit

Rejeitada porque uma falha entre o commit e a gravação perderia definitivamente a notificação.

## Consequências

### Positivas

- Mudança de status e criação do evento são atômicas.
- A API não espera chamadas HTTP externas.
- A solução reutiliza MySQL, Prisma e o modelo operacional existente.
- O snapshot torna o evento independente de alterações posteriores no pedido.

### Negativas

- O banco passa a acumular estado operacional e precisa de índices, monitoramento e retenção.
- A transação de mudança de status ganha consultas e inserções adicionais.
- A outbox não entrega eventos sozinha; depende de um worker saudável.
- Um webhook por destino pode multiplicar o número de linhas para a mesma transição.
