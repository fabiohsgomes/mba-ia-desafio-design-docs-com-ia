# ADR-003: Aplicar retry com backoff limitado e dead letter queue

## Status

Proposto — houve consenso na reunião técnica, mas a implementação ainda não existe.

## Contexto

Endpoints de clientes podem ficar temporariamente indisponíveis, exceder o timeout ou responder com
erro. Descartar o evento na primeira falha tornaria a entrega frágil; tentar indefinidamente
deixaria eventos presos e consumiria recursos sem limite. A operação também precisa conservar
evidências para diagnóstico e permitir recuperação manual.

A reunião definiu cinco retries e enumerou cinco intervalos após a primeira falha. Neste ADR,
`retry` significa uma nova chamada posterior à chamada inicial. Portanto, os cinco intervalos
representam cinco retries além do envio inicial, totalizando no máximo seis chamadas.

## Decisão

Após a tentativa inicial, serão permitidas até cinco retentativas, agendadas em `1 minuto`,
`5 minutos`, `30 minutos`, `2 horas` e `12 horas`. O worker persistirá contagem, próxima tentativa,
último erro e histórico de cada entrega; ele não permanecerá dormindo entre tentativas.

Esgotada a política, o evento será copiado ou movido atomicamente para uma tabela
`webhook_dead_letter`, contendo evento, payload, motivo e data da falha. Um endpoint administrativo
permitirá replay, protegido por `authenticate` e `requireRole('ADMIN')`. A operação registrará o
usuário que solicitou o replay e recolocará o evento como pendente preservando seu `event_id`.

## Alternativas Consideradas

### Retry indefinido

Maximizaria a chance de entrega após indisponibilidades longas, mas manteria lixo operacional e
custo sem limite para endpoints abandonados.

### Três tentativas em uma janela curta

Reduziria custo e tempo até falha permanente, mas não cobriria manutenções de algumas horas.

### Marcar falha definitiva apenas na própria outbox

Usaria menos tabelas, mas misturaria trabalho elegível com evidências permanentes e dificultaria
consulta e replay da DLQ.

## Consequências

### Positivas

- Falhas transitórias podem se recuperar sem intervenção.
- O limite impede retries eternos.
- A DLQ preserva evidência e oferece um fluxo auditável de replay.
- Agendar `nextAttemptAt` não bloqueia o worker entre as tentativas.

### Negativas

- Um evento pode levar cerca de quinze horas para esgotar a política.
- A tentativa inicial mais cinco retries produz até seis chamadas por evento.
- Persistir tentativas e DLQ aumenta volume e complexidade de limpeza.
- A classificação de erros retryable e a ordenação de eventos posteriores ainda exigem cuidado.
