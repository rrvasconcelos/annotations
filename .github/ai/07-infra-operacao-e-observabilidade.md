# Infraestrutura, Operação e Observabilidade

## Implantação

- A solução é implantada em Kubernetes.
- APIs e workers devem ser empacotados como imagens de contêiner.
- A mesma imagem deve ser promovida entre ambientes sempre que possível.

## Ambientes

- PRD
- NPRD
- DES
- TQS

## Requisitos operacionais

- APIs devem expor readiness e liveness.
- Workers devem expor heartbeat ou sinal operacional equivalente.
- Readiness não deve depender de integrações não essenciais.
- Segredos devem vir de cofre corporativo ou mecanismo equivalente.

## Execução crítica

A API de Execução tem criticidade especial porque atende canal digital e pode depender de Seleção e de fontes de contexto.
Timeout, circuit breaker, métricas próprias e tratamento de falha são obrigatórios.
