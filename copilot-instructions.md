# Copilot Instructions - Sistema de Métricas de Percepção

Você deve tratar este repositório como a base de arquitetura do Sistema de Métricas de Percepção.
Antes de sugerir código, leia o contexto global e o kit do serviço atual.

## Ordem de contexto

1. [README-ARQUITETURA.md](README-ARQUITETURA.md)
2. [Visão Geral do DAS](.github/ai/00-visao-geral-das.md)
3. [Objetivos e Escopo](.github/ai/01-objetivos-e-escopo.md)
4. [Domínios e Responsabilidades](.github/ai/02-dominios-e-responsabilidades.md)
5. [Integrações e Comunicação](.github/ai/03-integracoes-e-comunicacao.md)
6. [Eventos, Idempotência e Consistência](.github/ai/04-eventos-idempotencia-e-consistencia.md)
7. [Segurança, Privacidade e Tokens](.github/ai/05-seguranca-privacidade-e-tokens.md)
8. [Runtime e Fluxos](.github/ai/06-runtime-e-fluxos.md)
9. O documento do serviço em [.github/ai/servicos](.github/ai/servicos)

## Regras duras

- Não mova responsabilidade entre serviços.
- Não consulte persistência de outro serviço diretamente.
- Não decida elegibilidade fora da Seleção.
- Não confirme resposta esperando apuração.
- Não calcule métrica na Execução.
- Não materialize mensagens com dados pessoais sem necessidade arquitetural explícita.
- Não exponha token completo, URL sensível ou PII em logs, traces ou eventos.
- Não crie contratos implícitos para eventos.
- Não assuma que cache substitui o domínio autoritativo.

## Critérios de resposta

- Se a dúvida for de arquitetura, responda a partir do DAS e da responsabilidade do serviço.
- Se a dúvida for de implementação, respeite a fronteira do serviço atual e o vocabulário do glossário.
- Se houver conflito entre dois comportamentos, prefira o texto do DAS e sinalize a divergência.
