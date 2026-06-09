# Ingestão de Origens

## Papel

Fronteira técnica de entrada para atendimentos, integrações e listas proativas autorizadas.

## O que faz

- Recebe origem por API ou arquivo
- Valida contrato técnico
- Normaliza payloads recebidos
- Registra rejeições técnicas e duplicidades
- Publica fato de origem registrada

## O que não faz

- Não decide elegibilidade
- Não aplica blocklist ou quarentena
- Não cria oportunidade ou instância
- Não escolhe a pesquisa

## Persistência e eventos

- Base própria
- Evento de origem registrada para a Seleção
