# Apuração de Resultados

## Papel

Transforma respostas e não-respostas em resultados interpretáveis e agregados.

## O que faz

- Consome resposta registrada e fatos de não-resposta
- Calcula resultados segundo a política da métrica
- Mantém rastreabilidade entre resultado, resposta, instância e versão
- Expõe resultados consolidados para consumidores autorizados

## O que não faz

- Não recebe resposta do canal
- Não decide elegibilidade
- Não cria instância
- Não envia convite
- Não lê persistência interna de outros serviços

## Persistência e eventos

- Base própria
- Consome fatos publicados pela Execução
- Expondo API de resultados para consumo autorizado
