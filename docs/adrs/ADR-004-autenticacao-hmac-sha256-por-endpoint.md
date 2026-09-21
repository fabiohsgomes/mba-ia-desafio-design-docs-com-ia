# ADR-004: Autenticar webhooks com HMAC-SHA256 por endpoint

## Status

Proposto — houve consenso na reunião técnica, mas a implementação ainda não existe.

## Contexto

Os webhooks transportam dados de pedidos para sistemas fora da plataforma. O consumidor precisa
verificar a origem da requisição e a integridade do corpo sem depender apenas de endereço IP. Uma
credencial global ampliaria o impacto de vazamento e impediria revogação isolada por cliente.

## Decisão

Cada endpoint terá uma secret aleatória exclusiva, gerada pela plataforma e devolvida apenas na
criação ou rotação. O worker calculará HMAC-SHA256 sobre os bytes exatos do corpo JSON enviado e
publicará o resultado em `X-Signature`. A entrega também incluirá `X-Event-Id`, `X-Webhook-Id`,
`X-Timestamp` e `Content-Type: application/json`.

Somente URLs HTTPS serão aceitas. A secret será rotacionável, com a credencial anterior mantida por
24 horas para transição. A implementação deverá especificar de forma interoperável qual chave assina
durante essa janela antes de disponibilizar o endpoint de rotação.

Como o segredo precisa ser recuperado para assinar, ele não pode ser armazenado apenas como hash.
Deverá ser protegido em repouso e nunca aparecer em respostas de listagem, logs ou histórico. A
configuração de redaction de `src/shared/logger/index.ts` deverá incluir campos de secret.

## Alternativas Consideradas

### Secret HMAC global

Seria mais simples, mas o vazamento de um cliente comprometeria todos os endpoints.

### Assinatura assimétrica

Evitaria distribuir segredo de assinatura, porém aumenta gestão de chaves e complexidade de adoção
para os clientes sem necessidade demonstrada nesta fase.

### TLS sem assinatura da aplicação

Protege o transporte, mas não fornece ao consumidor uma prova no nível da mensagem.

## Consequências

### Positivas

- Clientes podem verificar origem e integridade com bibliotecas amplamente disponíveis.
- Comprometimento e rotação ficam isolados por endpoint.
- HTTPS e assinatura oferecem camadas complementares de proteção.

### Negativas

- A plataforma passa a custodiar segredos recuperáveis e precisa de criptografia e gestão de chave.
- Rotação com duas secrets aumenta estados e casos de teste.
- Assinar somente o corpo não autentica `X-Timestamp`; proteção forte contra replay exigirá evoluir
  o contrato de assinatura ou usar o timestamp presente no próprio payload.
- Validar apenas o esquema HTTPS não elimina SSRF, redirects maliciosos ou DNS rebinding.
