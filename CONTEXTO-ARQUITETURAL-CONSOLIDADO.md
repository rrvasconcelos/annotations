# Contexto Arquitetural Consolidado

Data de consolidação: 2026-06-02
Escopo analisado: DAS principal, contexto global em `.github/ai`, kits por serviço e modelo conceitual de dados.

## 1. Visão executiva da solução

O Sistema de Métricas de Percepção foi desenhado como plataforma corporativa orientada a domínios para suportar CSAT, NPS, NES e evolução futura sem redesenhar o núcleo.

A arquitetura separa de forma explícita:
- Definição governada da pesquisa (Gestão)
- Decisão de elegibilidade (Seleção)
- Execução operacional da experiência e coleta (Execução)
- Entrega técnica de convites (Entrega)
- Cálculo e consolidação de resultados (Apuração)

Essa separação evita acoplamento indevido, reduz ambiguidades de ownership e permite evolução independente por domínio.

## 2. Princípios arquiteturais mandatórios

1. Microsserviços por domínio (não por entidade, tabela ou CRUD).
2. Persistência lógica própria por serviço; sem consulta direta entre bases.
3. Comunicação híbrida:
- Síncrona apenas quando há consumidor aguardando resposta imediata.
- Assíncrona para propagação de fatos, desacoplamento, retry e isolamento de falhas.
4. Gestão publica versões imutáveis; consumidores materializam localmente por domínio.
5. Seleção é o único domínio autorizador de elegibilidade (inclusive jornada digital).
6. Execução confirma recebimento da resposta sem aguardar Apuração.
7. Eventos interdomínio são contratos públicos versionados com idempotência.
8. Consistência entre domínios é eventual (sem transação distribuída).
9. Minimização de PII em eventos/logs/traces/cache.
10. Token/link de pesquisa é credencial sensível (opaco, expirável, não logável por inteiro).

## 3. Mapa de domínios e fronteiras

### 3.1 Gestão de Pesquisas
Faz:
- Definição, validação, versionamento e publicação da pesquisa.
- Governança de regras, templates, critérios de métrica e blocklist (cadastro).

Não faz:
- Elegibilidade por respondente.
- Resolução de placeholders em runtime.
- Criação de instância, coleta de resposta ou cálculo de resultado.

### 3.2 Ingestão de Origens
Faz:
- Entrada técnica, validação e normalização de origens autorizadas.
- Registro auditável de aceite/rejeição técnica.

Não faz:
- Decisão de elegibilidade.
- Criação de oportunidade, instância, convite ou resultado.

### 3.3 Seleção e Elegibilidade
Faz:
- Decisão autoritativa de elegibilidade.
- Aplicação de blocklist, quarentena, amostragem e prioridade.
- Decisão assíncrona (origens) e síncrona (jornada digital da Execução).

Não faz:
- Instância, token, renderização, resposta e métrica.

### 3.4 Execução de Pesquisas
Faz:
- Ciclo de vida da instância (token, snapshot, estado, expiração, limite).
- Materialização contextual de placeholders com fontes autorizadas.
- Registro da resposta bruta e fatos operacionais (resposta, não-resposta, encerramento).

Não faz:
- Governança de elegibilidade.
- Cálculo da métrica final.

### 3.5 Entrega de Convites
Faz:
- Orquestração operacional de envio (tentativas, falhas, retry).

Não faz:
- Elegibilidade, criação de instância, cálculo de resultado.

### 3.6 Apuração de Métricas e Resultados
Faz:
- Consumo assíncrono de fatos da Execução.
- Aplicação de critérios materializados e cálculo de métricas.
- Exposição de resultados consolidados.

Não faz:
- Recebimento de resposta de canal, elegibilidade, execução de instância.

## 4. Integrações e padrões de comunicação

### 4.1 Síncronas (latência crítica)
- Canal Digital -> Execução (consulta/materialização e submissão)
- Execução -> Seleção (elegibilidade em jornada digital)
- Execução -> SICLI/SIISO/LDAP (contexto autorizado para placeholders)
- Consumidor autorizado -> Apuração (consulta de resultados)

### 4.2 Assíncronas (desacoplamento)
- Gestão -> Seleção/Execução/Apuração (publicação de versão)
- Ingestão -> Seleção (origem registrada)
- Seleção -> Execução (oportunidade)
- Execução -> Entrega (instância criada)
- Execução -> Apuração (resposta/não-resposta/encerramento)

## 5. Eventos, idempotência e consistência

Regras mandatórias:
- Evento é contrato público versionado.
- Envelope mínimo: identificação, versão, correlação/rastreabilidade e chave idempotente.
- Produtor crítico com Outbox (ou equivalente).
- Consumidor crítico com Inbox/deduplicação.
- Reentrega não pode produzir efeito duplicado.

Distinções obrigatórias de fatos operacionais:
- Expiração temporal
- Não-resposta
- Encerramento por limite de respostas

Esses fatos devem permanecer semanticamente distintos para auditoria e apuração corretas.

## 6. Segurança, privacidade e governança

Diretrizes:
- Pseudonimização por `respondenteRef` como padrão interno.
- Evitar CPF/e-mail/telefone/nome completo em payload interno quando não estritamente necessário.
- Não registrar token completo nem URL sensível em logs/traces/eventos.
- IAM corporativo, menor privilégio por workload e segredos em cofre.
- Rate limit na borda (APIM/gateway) com política formal definida.

## 7. Fluxos de runtime essenciais

1. Publicação de pesquisa e materialização local por domínio.
2. Origem assíncrona: Ingestão aceita tecnicamente e Seleção decide elegibilidade.
3. Jornada digital: Execução consulta Seleção de forma síncrona antes de materializar.
4. Submissão de resposta: Execução persiste e confirma, Apuração processa assíncrono.
5. Expiração/não-resposta/limite: Execução controla estado e publica fatos.
6. Falha e reentrega: deduplicação obrigatória no consumo.

## 8. Riscos e lacunas já identificados na documentação

Riscos relevantes:
- Gargalo/latência na Seleção para jornadas digitais.
- Inconsistência de materialização entre serviços.
- Exposição acidental de PII/tokens.
- Governança insuficiente de contratos de eventos.
- Observabilidade insuficiente para diagnóstico ponta a ponta.

Lacunas/pendências arquiteturais:
- SLOs/latência alvo por jornada e canal.
- Volumetria, throughput, concorrência e retenção.
- Estratégia final de cache e invalidação.
- Política formal de fallback de placeholders.
- Política de rate limit em produção.
- Padrão corporativo de pseudonimização.

## 9. Compatibilidade entre DAS e modelo conceitual

Alinhamentos principais:
- Ownership por domínio.
- Materialização local.
- Outbox/idempotência.
- Pseudonimização e cuidado com dados sensíveis.

Ponto de atenção:
- O modelo conceitual usa exemplos com CSAT/NPS/NES em campos tipados.
- O DAS exige evolução para futuras métricas.
- Interpretação recomendada: enumerações no modelo são ilustrativas, não bloqueio definitivo.

## 10. Checklist prático para qualquer implementação

1. A mudança respeita o ownership do serviço atual?
2. Há algum cálculo/decisão fora do domínio correto?
3. Existe leitura direta de banco de outro serviço? (se sim, bloquear)
4. O contrato de evento está versionado e com idempotência?
5. O fluxo síncrono é realmente necessário para UX?
6. A confirmação ao usuário depende apenas do domínio correto?
7. Token/PII podem vazar em logs/traces/payloads?
8. Há estratégia clara para retry, deduplicação e DLQ?
9. A rastreabilidade ponta a ponta está coberta?
10. Lacunas pendentes foram elevadas para ADR/runbook em vez de codificadas ad hoc?

## 11. Posicionamento arquitetural para o projeto

A solução está corretamente orientada por domínio e com fronteiras bem definidas no DAS.

A prioridade para evolução segura não é criar novos componentes, e sim fechar decisões pendentes de qualidade arquitetural (SLO, cache, fallback, rate limit, governança de eventos e observabilidade).

Sem essas decisões fechadas, o maior risco é desvio silencioso de fronteira e inconsistência operacional entre serviços.

## 12. Próximos passos recomendados (ordem sugerida)

1. Fechar baseline de SLO/timeout/circuit-breaker por fluxo digital.
2. Publicar política de fallback de placeholders por classe de erro.
3. Formalizar estratégia de cache (TTL, invalidação, sensibilidade de dados).
4. Publicar catálogo de eventos versionado com chaves idempotentes.
5. Definir política APIM de rate limit para PRD.
6. Definir padrão corporativo de pseudonimização e mascaramento.
7. Instrumentar observabilidade ponta a ponta por correlationId.
8. Definir controles de concorrência para limite de respostas na Execução.
9. Validar volumetria e retenção para decisões de persistência por domínio.
10. Executar checklist arquitetural como gate de revisão técnica.
