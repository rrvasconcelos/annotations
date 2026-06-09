# Objetivos e Escopo

## Objetivo principal

Disponibilizar uma plataforma governada para pesquisas de percepção, com suporte a múltiplas métricas e diferentes origens de pesquisa.

## Objetivos em nível arquitetural

- Permitir gestão versionada de pesquisas
- Garantir elegibilidade centralizada em um único domínio
- Suportar materialização contextual controlada por placeholders
- Manter rastreabilidade ponta a ponta
- Preservar evolução independente entre domínios

## Escopo incluído

- Definição e publicação de pesquisas
- Ingestão de atendimentos e listas proativas
- Decisão de elegibilidade
- Materialização e execução da experiência de resposta
- Entrega de convites
- Apuração e exposição de resultados consolidados
- Controle de blocklist, quarentena, expiração e limite de respostas

## Escopo excluído

- Operação interna dos canais digitais
- Gestão de clientes, unidades e funcionários
- Execução dos atendimentos de origem
- Estratégia corporativa de CX como um todo
- Exploração analítica avançada fora dos resultados expostos pela solução
- Detalhes de contrato completo de APIs e payloads em nível de implementação

## Premissas centrais

- Pesquisa não é métrica.
- Elegibilidade não deve ser replicada em canal, ingestão ou execução.
- Dados pessoais devem ser minimizados sempre que possível.
