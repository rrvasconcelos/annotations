# Eventos, Idempotência e Consistência

## Princípios

- Eventos entre domínios são contratos públicos versionados.
- A solução não usa transações distribuídas.
- A consistência entre domínios é eventual.
- Fluxos críticos devem considerar Outbox e Inbox.

## Regras práticas

- Produtores críticos devem persistir evento para posterior publicação.
- Consumidores críticos devem deduplicar reentregas.
- Chaves de negócio e correlationId devem sustentar rastreabilidade.
- Eventos de expiração, não-resposta e encerramento por limite devem ser distinguíveis.

## O que o Copilot deve presumir

- Reentrega não pode gerar duplicidade funcional.
- Reprocessamento não pode ser implícito.
- Um evento não deve ser modelado sem consumidor real ou sem governança clara.
