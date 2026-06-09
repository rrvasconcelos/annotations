# Entrega de Convites

## Papel

Serviço operacional especializado em acionar a plataforma de comunicação.

## O que faz

- Consome a informação necessária para acionar o convite quando a Execução disponibiliza a instância para entrega
- Solicita envio técnico ao provedor corporativo
- Controla tentativas, sucesso, falha e retry
- Registra ciclo operacional do convite

## O que não faz

- Não decide público
- Não cria instância
- Não altera questionário
- Não calcula resultado

## Persistência e eventos

- Base própria
- Interage com plataforma de comunicação externa
