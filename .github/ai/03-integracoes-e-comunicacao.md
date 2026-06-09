# Integrações e Comunicação

## Regra geral

A solução usa comunicação híbrida.
Síncrono quando alguém aguarda resposta imediata.
Assíncrono quando o objetivo é propagar fatos, desacoplar processamento, permitir retry ou isolar falhas.

## Integrações externas

| Integração | Tipo | Papel |
|---|---|---|
| API de Gestão | HTTP | Interface administrativa |
| API de Ingestão | HTTP/arquivo | Entrada de origens autorizadas |
| API de Execução | HTTP | Consulta e submissão na jornada digital |
| API de Resultados | HTTP | Exposição de resultados consolidados |
| Broker de eventos | Mensageria | Integração entre domínios |
| Plataforma de comunicação | API ou integração corporativa | Envio técnico de convites |
| IAM | Integração corporativa | Autenticação e autorização |
| SICLI, SIISO e LDAP | API/LDAP | Contexto autorizado para materialização |

## Diretriz importante

Origem, elegibilidade, execução e apuração são responsabilidades diferentes.
O fato de dois domínios trocarem mensagens não significa que eles compartilhem decisão.
