# Regras de Implementação

## Regras de ouro

- Implementação deve respeitar a fronteira do domínio atual.
- Regra de negócio central não deve ser duplicada em canal ou integração.
- Persistência deve ser própria do serviço.
- Eventos críticos devem ser idempotentes e rastreáveis.
- Dados sensíveis devem ser minimizados.

## Noções proibidas

- Consultar banco de outro serviço por atalho.
- Fazer cálculo de resultado na Execução.
- Validar elegibilidade fora da Seleção.
- Assumir que cache substitui decisão autoritativa.
- Persistir segredo em log, trace ou manifesto.
