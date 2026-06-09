# Riscos, Lacunas e Decisões Pendentes

## Lacunas que o DAS ainda deixa para artefatos acessórios

- SLOs e latências alvo
- Volumetria e throughput
- Retenção
- Concorrência esperada
- Política final de cache
- Política de fallback de placeholders
- Padrão corporativo de pseudonimização

## Riscos relevantes

- Fragmentação excessiva de microsserviços
- Eventos sem governança de contrato
- Reentrega sem idempotência
- Exposição de dados pessoais
- Observabilidade insuficiente
- Materialização divergente de versões
- Seleção como gargalo em jornada digital

## Regra de trabalho

Se um assunto ainda não tem definição arquitetural fechada, ele não deve ser inventado no código.
Deve ser elevado para documento técnico, ADR complementar ou runbook.
