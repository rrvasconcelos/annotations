# Checklist de Arquitetura

Antes de concluir um PR ou iniciar um serviço, confirme:

- O serviço sabe claramente o que faz e o que não faz.
- A responsabilidade de elegibilidade está na Seleção quando aplicável.
- A gestão de definição está separada do runtime.
- O serviço não acessa persistência de outro domínio.
- Os eventos publicados são versionados e consumíveis.
- A resposta ao usuário não depende indevidamente de apuração.
- Tokens, PII e URLs sensíveis não aparecem em logs ou eventos.
- A observabilidade cobre falhas, duplicidade e reentrega.
