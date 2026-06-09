# Visão Geral do DAS

O Sistema de Métricas de Percepção é uma plataforma corporativa para configurar, executar, coletar e apurar pesquisas de percepção.
O DAS estabelece uma arquitetura orientada a domínios para suportar métricas como CSAT, NPS e NES, sem amarrar o núcleo da solução a uma métrica específica.

## Intenção arquitetural

O documento prioriza fronteiras claras entre domínios, separação entre definição e runtime, comunicação híbrida e persistência própria por serviço.

## Blocos principais

- Gestão de Pesquisas
- Ingestão de Origens de Pesquisa
- Seleção de Público e Elegibilidade
- Execução de Pesquisas
- Entrega de Convites
- Apuração de Métricas e Resultados

## Sistemas vizinhos e dependências externas

- IAM para autenticação e autorização corporativa
- Broker de eventos para integração assíncrona interna
- Plataforma de comunicação para envio de convites
- SICLI, SIISO e LDAP como fontes autorizadas de contexto na Execução
- APIs corporativas para gestão, ingestão, execução e resultados

## Leitura correta do DAS

O DAS não descreve só componentes.
Ele define quem decide, quem executa, quem persiste e quem publica fatos.

## Diagramas de apoio

- [Contexto técnico](../../IMGS/c4_contexto.svg)
- [Containers](../../IMGS/c4_containers.svg)
