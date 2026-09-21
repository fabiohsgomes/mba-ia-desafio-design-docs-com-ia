# Da Reunião ao Documento: Design Docs Gerados por IA

### **Sobre o desafio**

O desafio consiste em transformar uma transquição de uma reunião na qual discute o desenvolvimento de uma nova feature em um sistema já existem em um conjuntos de documentos técnicos e não técnicos. Esses documentos tem o objeto de informar para todos os responsáveis pelo sistema que vão do PM ao Desenvolvedor, o que é a feature, quais componentes, stack, padrões e tradeoff estão presentes e o roteiro de desenvolvimento.

### **Ferramentas de IA utilizadas**

- Plugin ADRs Management: utilizado para documentar decisões arquiteturais de forma sistemática;
- Skill FDD Interview: utilizado para gerar o documento FDD.md, através de uma entrevista;
- Skill PRD Interview: utilizado para gerar o documento PRD.md, através de uma entrevista.

### **Workflow adotado**

Eu segui o workflow sugerido. Comecei pedido ao modelo que explora-se o code base para reconhecer os padrões, já existentes e a transquição para entender o que foi discutido na reunião. A partir daí utilizei o plugin ADRs Management para produzir os ADRs.
Depois que as ADRs foram criadas, através de prompt próprio, solicitei ao modelo que produzisse o RFC.md, utilizando a transcrição e os ADRs como contexto.
O FDD.md e o PRD.md eu utilizei as skills mensionadas na seção **Ferramentas de IA utilizadas**.
Todas as interações foram realizadas na mesma seção com compactação em 40% e 50%  de uso do contexto.

### **Prompts customizados**

pelo menos 2 prompts relevantes que você escreveu ou adaptou, mostrados em blocos de código

#### Criação das ADRs
```markdown
[Plugin ADRs Management]

1- produza ADRs em arquivos separados dentro de [adrs](docs/adrs/) , nomeados no formato `ADR-NNN-titulo-em-kebab-case.md` (ex: `ADR-001-outbox-no-mysql.md`).
2- Cada ADR deve seguir o formato MADR com as seções: Status, Contexto, Decisão, Alternativas Consideradas (pelo menos 1 alternativa real discutida ou plausível), Consequências (positivas e negativas, com trade-off explícito).
3- Pelo menos 1 ADR deve referenciar explicitamente arquivos, módulos ou padrões do código existente.
4- O conjunto de ADRs deve cobrir, no mínimo, 5 das 6 decisões principais discutidas na reunião:
- Padrão Outbox no MySQL
- Política de retry com backoff e DLQ
- Autenticação HMAC-SHA256 com secret por endpoint
- Garantia at-least-once com `X-Event-Id`
- Worker em processo separado em polling
- Reuso dos padrões existentes do projeto
5- Decisões técnicas secundárias (formato de payload, timeouts, headers, entre outras) podem virar ADRs adicionais
```

#### Criacao da RFC
```markdown
### Objetivo

Baseado em [TRANSCRICAO.md](TRANSCRICAO.md), produza um RFC em `docs/RFC.md` com a proposta técnica da solução, no formato de um documento submetido à equipe para revisão.

### Contexto

O RFC opera em nível de arquitetura: apresenta a abordagem escolhida, as alternativas que foram colocadas na mesa e as questões deixadas em aberto. Ele responde "o que propomos e por quê". Deve ser um documento conciso sem detalhamento de implementação

### Saída

O documento deve ter os seguintes tópicos:

- Metadados (autor, status, data, revisores); use os participantes da reunião como revisores
- Resumo executivo (TL;DR) da proposta
- Contexto e problema
- Proposta técnica (visão geral da solução, sem descer ao detalhe de implementação do FDD)
- Alternativas consideradas (pelo menos 2 alternativas reais discutidas e descartadas na reunião, cada uma com o trade-off que levou ao descarte)
- Questões em aberto (pelo menos 2 pontos levantados na reunião e não decididos ou adiados)
- Impacto e riscos
- Decisões relacionadas (links para os [adrs](docs/adrs/) correspondentes)
```
### **Iterações e ajustes**

Dois ajustes foram necessários durante a revisão dos documentos, através do chekilist.

O primeiro foi detectado no FDD onde o item destacado abaixo estava atendido apenas parcialmente.

``Seção "Contratos públicos" inclui pelo menos 4 endpoints HTTP com payload de exemplo (request e response) e status codes``

Alguns endpoints não tinham exemplos de request e/ou response.
Como o problema era de complexidade baixa, somente com uma única interação o problema foi resolvido.

O segundo sugiu uma resolução de semântica em relação à política de retry. A transcrição usa a expressão “5
tentativas”. No meu entendimento e do própio modelo durante a construção dos documentos a primeira tentativa, não conta entre as 5, inclusive isso é percebido na política de backoff, porém na hora de passar os documentos pelo checklit o modelo não considerou como atendida o item destacato abaixo.

`Nenhum requisito, decisão ou restrição registrada nos documentos contradiz a transcrição ou o código`

Após uma breve checagem na transcrição para validar a minha tese, expliquei para o modelo que não se tratava de item não atendido mas sim de entendimento. Solicitei então ao modelo que deixasse claro questão nos documentos e isso foi feito. PRD, RFC, FDD, ADR-003, mapeamento e Tracker foram verificados ou ajustados para usar a mesma definição de 1 tentativa mais 5 retry, totalizando 6 chamadas.

### **Como navegar a entrega**

A ordem sugerida de leitura parte do contexto de produto, avança para a proposta arquitetural e o
detalhamento técnico, registra as decisões e termina na rastreabilidade:

1. [`docs/PRD.md`](docs/PRD.md): apresenta o problema, o público-alvo, os objetivos, o escopo, os
   requisitos, os riscos e os critérios de aceitação da feature.
2. [`docs/RFC.md`](docs/RFC.md): consolida a proposta arquitetural, as alternativas consideradas,
   os impactos e as questões que permanecem em aberto.
3. [`docs/FDD.md`](docs/FDD.md): detalha como implementar a solução, incluindo fluxos, contratos,
   persistência, segurança, resiliência, observabilidade e integração com o código existente.
4. ADRs, para consultar individualmente o contexto e os trade-offs de cada decisão arquitetural:
   - [`docs/adrs/README.md`](docs/adrs/README.md): índice e orientação de leitura dos ADRs.
   - [`docs/adrs/mapping.md`](docs/adrs/mapping.md): mapeamento do código usado para identificar as
     decisões arquiteturais.
   - [`ADR-001`](docs/adrs/ADR-001-outbox-no-mysql.md): outbox no MySQL.
   - [`ADR-002`](docs/adrs/ADR-002-worker-separado-com-polling.md): worker separado com polling.
   - [`ADR-003`](docs/adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md): retry com
     backoff e DLQ.
   - [`ADR-004`](docs/adrs/ADR-004-autenticacao-hmac-sha256-por-endpoint.md): autenticação
     HMAC-SHA256 por endpoint.
   - [`ADR-005`](docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md): entrega
     at-least-once com `X-Event-Id`.
   - [`ADR-006`](docs/adrs/ADR-006-reuso-dos-padroes-arquiteturais-existentes.md): reuso dos
     padrões arquiteturais existentes.
5. [`docs/TRACKER.md`](docs/TRACKER.md): relaciona requisitos, decisões e restrições às respectivas
   evidências na transcrição ou no código.
