# Runtime e Fluxos

## Publicação e materialização

A Gestão publica uma versão imutável.
Seleção, Execução e Apuração materializam localmente o que precisam.

## Origem assíncrona com elegibilidade

1. Sistema ou origem autorizada envia atendimento ou lista proativa.
2. Ingestão valida tecnicamente e normaliza.
3. Ingestão publica origem registrada.
4. Seleção aplica blocklist, quarentena, elegibilidade, amostragem e prioridade.
5. Se elegível, Seleção publica oportunidade criada.
6. Execução cria ou recupera instância a partir da oportunidade.

## Jornada digital com elegibilidade síncrona

1. Canal solicita pesquisa disponível ao fim da jornada.
2. Execução consulta a Seleção em tempo síncrono.
3. Seleção responde com decisão de elegibilidade.
4. Execução materializa snapshot, templates e contexto autorizado.
5. Canal recebe a pesquisa materializada ou a ausência dela.

## Instância criada

A criação da instância acontece dentro do processamento síncrono da Execução.
Não há publicação de evento específico apenas pelo fato de a instância ter sido criada.
O que segue para outros consumidores são os fatos operacionais posteriores, como resposta registrada, não-resposta ou encerramento.

## Submissão de resposta e apuração assíncrona

1. Respondente envia a resposta.
2. Execução valida instância, token, status e expiração.
3. Execução grava resposta bruta e itens.
4. Execução publica resposta registrada.
5. Apuração calcula resultados depois, de forma assíncrona.

## Expiração e não-resposta

Execução identifica expiração, não-resposta ou limite atingido e publica o fato operacional correspondente.

## Falha e reentrega

Consumidores críticos devem reconhecer reentrega sem novo efeito funcional.
