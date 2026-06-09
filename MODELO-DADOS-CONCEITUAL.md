# Modelo de Dados Conceitual por Microsserviço

> **Importante:** Este documento é uma proposta conceitual baseada nas responsabilidades descritas no DAS.
> Não é um modelo físico fechado. Serve para orientar o entendimento de negócio e as primeiras decisões de implementação.
> Identidades de respondentes devem ser pseudonimizadas em toda a solução (não usar CPF/CNPJ cru em eventos, logs ou caches).

---

## 1. Gestão de Pesquisas

### Visão de negócio
É o "cadastro mestre" do sistema. Tudo começa aqui.
Um gestor entra, cria uma pesquisa, configura perguntas, define para qual público ela serve,
define quanto tempo tem, quantas respostas aceita, e quando estiver pronta, **publica**.
A partir daí, a versão publicada é **imutável** — ninguém pode alterar o que já foi publicado.
Se precisar mudar, cria uma nova versão.

### Banco sugerido: **Relacional (PostgreSQL / SQL Server)**
**Por quê:** Os dados são altamente estruturados, têm relações claras entre si (pesquisa tem versões, versões têm perguntas, perguntas têm opções) e precisam de integridade transacional na hora de publicar. SQL é ideal aqui.

---

### Entidades

```sql
-- A pesquisa em si (o "produto" que será configurado)
CREATE TABLE pesquisa (
  id              UUID PRIMARY KEY,
  nome            VARCHAR(200) NOT NULL,
  descricao       TEXT,
  metrica         VARCHAR(20) NOT NULL,  -- 'CSAT' | 'NPS' | 'NES'
  estado          VARCHAR(20) NOT NULL,  -- 'rascunho' | 'publicada' | 'encerrada'
  criado_em       TIMESTAMP NOT NULL,
  atualizado_em   TIMESTAMP NOT NULL
);

-- Cada vez que uma pesquisa é publicada, gera uma versão imutável
-- A versão é o que os outros serviços consomem — nunca a pesquisa diretamente
CREATE TABLE versao_pesquisa (
  id                  UUID PRIMARY KEY,
  pesquisa_id         UUID NOT NULL REFERENCES pesquisa(id),
  numero_versao       INT NOT NULL,
  publicada_em        TIMESTAMP NOT NULL,
  expiracao_dias      INT,              -- quantos dias a instância fica aberta
  limite_respostas    INT,              -- null = sem limite
  estado              VARCHAR(20) NOT NULL DEFAULT 'ativa',  -- 'ativa' | 'revogada'
  -- snapshot completo da configuração no momento da publicação (imutável)
  configuracao_json   JSONB NOT NULL,   -- perguntas, opções, ordem, validações
  criterios_elegibilidade_json  JSONB, -- regras que a Seleção vai usar
  criterios_metrica_json        JSONB, -- fórmulas que a Apuração vai usar
  UNIQUE (pesquisa_id, numero_versao)
);

-- Templates de mensagem para convites (com placeholders permitidos)
-- Ex: "Olá {{cliente.nome}}, avalie seu atendimento na {{atendimento.unidade.nome}}"
CREATE TABLE template_mensagem (
  id                  UUID PRIMARY KEY,
  versao_pesquisa_id  UUID NOT NULL REFERENCES versao_pesquisa(id),
  canal               VARCHAR(50) NOT NULL,  -- 'email' | 'sms' | 'push' | 'app'
  assunto             VARCHAR(300),
  corpo               TEXT NOT NULL,
  placeholders_permitidos  TEXT[],  -- lista dos placeholders válidos neste template
  criado_em           TIMESTAMP NOT NULL
);

-- Lista de respondentes que NUNCA devem receber pesquisa
-- Cadastrada aqui, mas aplicada pela Seleção
CREATE TABLE blocklist (
  id                    UUID PRIMARY KEY,
  respondente_ref       VARCHAR(100) NOT NULL,  -- identificador pseudonimizado
  pesquisa_id           UUID REFERENCES pesquisa(id),  -- null = bloqueio global
  motivo_codigo         VARCHAR(50),
  vigente_ate           TIMESTAMP,  -- null = permanente
  criado_em             TIMESTAMP NOT NULL
);
```

### Evoluções recomendadas (materialização e governança)

```sql
-- Governança de publicação por versão (auditoria operacional)
CREATE TABLE historico_publicacao_versao (
  id                    UUID PRIMARY KEY,
  versao_pesquisa_id    UUID NOT NULL REFERENCES versao_pesquisa(id),
  event_id              UUID NOT NULL,
  schema_evento         VARCHAR(50) NOT NULL,      -- ex: pesquisa_versao_publicada.v1
  hash_conteudo         VARCHAR(64) NOT NULL,      -- hash do payload publicado
  tentativa             INT NOT NULL,
  status_publicacao     VARCHAR(30) NOT NULL,      -- 'pendente' | 'publicado' | 'falhou'
  erro_ultimo           TEXT,
  criado_em             TIMESTAMP NOT NULL,
  publicado_em          TIMESTAMP
);

-- Metadados de governança da versão (imutabilidade + rastreio)
-- Sugestão de evolução em versao_pesquisa:
-- hash_conteudo VARCHAR(64) NOT NULL
-- publicada_por VARCHAR(100) NOT NULL
-- motivo_publicacao TEXT
-- revogada_em TIMESTAMP
-- revogada_por VARCHAR(100)
-- versao_anterior_id UUID
```

---

## 2. Ingestão de Origens

### Visão de negócio
É a "portaria" do sistema. Qualquer sistema externo que queira avisar
que um cliente pode ser pesquisado passa por aqui primeiro.
O sistema de atendimento finaliza um atendimento e chama a API da Ingestão.
Ela não decide nada — só valida se a mensagem está bem formada, registra que chegou,
e passa para a frente via evento.

### Banco sugerido: **Relacional (PostgreSQL)**
**Por quê:** Precisa de rastreabilidade e auditoria de cada entrada recebida.
Tabelas simples, volume alto mas estrutura previsível. SQL funciona bem.
Uma fila (Kafka/RabbitMQ) complementa para o processamento assíncrono.

---

### Entidades

```sql
-- Cada chamada recebida de um sistema externo
CREATE TABLE origem_recebida (
  id                  UUID PRIMARY KEY,
  tipo                VARCHAR(30) NOT NULL,  -- 'atendimento' | 'lista-proativa' | 'integracao-externa'
  sistema_origem      VARCHAR(100) NOT NULL, -- qual sistema mandou
  identificador_externo VARCHAR(200),        -- id do atendimento no sistema de origem
  respondente_ref     VARCHAR(100) NOT NULL, -- identificador pseudonimizado do cliente
  canal               VARCHAR(50),           -- 'app' | 'web' | 'telefone' | etc
  payload_normalizado JSONB NOT NULL,        -- dados normalizados (sem PII desnecessário)
  estado              VARCHAR(20) NOT NULL,  -- 'aceita' | 'rejeitada' | 'reprocessando' | 'publicada'
  recebido_em         TIMESTAMP NOT NULL,
  correlation_id      UUID NOT NULL          -- rastreabilidade ponta a ponta
);

-- Registro de rejeições com motivo (para auditoria)
CREATE TABLE registro_rejeicao (
  id                  UUID PRIMARY KEY,
  origem_id           UUID NOT NULL REFERENCES origem_recebida(id),
  motivo_codigo       VARCHAR(50) NOT NULL,  -- 'payload_invalido' | 'sistema_nao_autorizado' | etc
  detalhe             TEXT,
  rejeitado_em        TIMESTAMP NOT NULL
);
```

### Evoluções recomendadas (idempotência e publicação)

```sql
-- Dedupe técnico da entrada para evitar retrabalho por reentrega da origem
CREATE TABLE dedupe_origem (
  id                    UUID PRIMARY KEY,
  sistema_origem        VARCHAR(100) NOT NULL,
  tipo                  VARCHAR(30) NOT NULL,
  identificador_externo VARCHAR(200) NOT NULL,
  payload_hash          VARCHAR(64) NOT NULL,
  primeiro_recebido_em  TIMESTAMP NOT NULL,
  ultimo_recebido_em    TIMESTAMP NOT NULL,
  total_reentregas      INT NOT NULL DEFAULT 0,
  UNIQUE (sistema_origem, tipo, identificador_externo)
);

-- Sugestão de evolução em origem_recebida:
-- idempotency_key_origem VARCHAR(200)
-- payload_hash VARCHAR(64)
-- publicado_event_id UUID
-- publicado_em TIMESTAMP
```

---

## 3. Seleção e Elegibilidade

### Visão de negócio
É o "juiz" do sistema. Toda vez que chega uma origem (evento da Ingestão)
ou uma consulta digital (da Execução), a Seleção verifica:
*"Esse cliente pode receber pesquisa agora?"*
Ela registra cada decisão — positiva ou negativa — para auditoria.
Se positivo, gera uma **Oportunidade** e avisa a Execução via evento.

### Banco sugerido: **Relacional (PostgreSQL)**
**Por quê:** Decisões de elegibilidade são registros auditáveis com campos bem definidos.
O histórico de quarentena precisa de queries de data/hora precisas.
SQL com índices bem montados resolve. Se o volume de decisões for muito alto
(milhões/dia), pode evoluir para uma tabela de quarentena em Redis para lookup rápido.

---

### Entidades

```sql
-- Cópia local das regras publicadas pela Gestão
-- A Seleção não consulta a Gestão em runtime — usa essa cópia
CREATE TABLE materializacao_regra (
  id                  UUID PRIMARY KEY,
  versao_pesquisa_id  UUID NOT NULL,         -- referência lógica (não FK cruzada)
  regras_json         JSONB NOT NULL,         -- critérios de elegibilidade copiados da Gestão
  materializado_em    TIMESTAMP NOT NULL,
  hash_versao         VARCHAR(64) NOT NULL,   -- para detectar divergências
  ativa               BOOLEAN DEFAULT TRUE
);

-- Cada decisão tomada — positiva ou negativa — registrada para auditoria
CREATE TABLE decisao_elegibilidade (
  id                  UUID PRIMARY KEY,
  origem_id           VARCHAR(200),           -- id da origem ou da consulta digital
  respondente_ref     VARCHAR(100) NOT NULL,  -- identificador pseudonimizado
  versao_pesquisa_id  UUID NOT NULL,
  resultado           VARCHAR(20) NOT NULL,   -- 'elegivel' | 'nao_elegivel'
  motivo_codigo       VARCHAR(50),            -- 'quarentena' | 'blocklist' | 'fora_publico' | etc
  regra_aplicada      VARCHAR(100),           -- qual regra foi determinante
  decidido_em         TIMESTAMP NOT NULL,
  correlation_id      UUID NOT NULL
);

-- Oportunidade gerada quando a decisão é positiva
-- Publicada como evento para a Execução processar
CREATE TABLE oportunidade (
  id                  UUID PRIMARY KEY,
  decisao_id          UUID NOT NULL REFERENCES decisao_elegibilidade(id),
  respondente_ref     VARCHAR(100) NOT NULL,
  versao_pesquisa_id  UUID NOT NULL,
  canal               VARCHAR(50),
  criado_em           TIMESTAMP NOT NULL,
  publicado_em        TIMESTAMP             -- quando o evento foi publicado com sucesso
  -- coluna de outbox: null = ainda não publicado, preenchido = publicado
);

-- Controle de quarentena por respondente/pesquisa/canal
CREATE TABLE quarentena (
  id                  UUID PRIMARY KEY,
  respondente_ref     VARCHAR(100) NOT NULL,
  pesquisa_id         UUID,            -- null = quarentena global
  canal               VARCHAR(50),     -- null = todos os canais
  inicio_quarentena   TIMESTAMP NOT NULL,
  fim_quarentena      TIMESTAMP NOT NULL,
  regra_aplicada      VARCHAR(100)
);
```

### Evoluções recomendadas (explicabilidade da decisão)

```sql
-- Trilho explicável de regras avaliadas em cada decisão
CREATE TABLE explicacao_decisao (
  id                    UUID PRIMARY KEY,
  decisao_id            UUID NOT NULL REFERENCES decisao_elegibilidade(id),
  ordem_avaliacao       INT NOT NULL,
  regra_codigo          VARCHAR(100) NOT NULL,
  resultado_regra       VARCHAR(20) NOT NULL,      -- 'aprovou' | 'reprovou' | 'nao_aplicavel'
  detalhe_avaliacao     JSONB,
  avaliado_em           TIMESTAMP NOT NULL
);

-- Controle explícito de materialização por versão consumida na Seleção
CREATE TABLE materializacao_versao_selecao (
  id                    UUID PRIMARY KEY,
  pesquisa_id           UUID NOT NULL,
  numero_versao         INT NOT NULL,
  versao_pesquisa_id    UUID NOT NULL,
  event_id_origem       UUID NOT NULL,
  hash_payload          VARCHAR(64) NOT NULL,
  status_materializacao VARCHAR(30) NOT NULL,      -- 'processando' | 'materializado' | 'falhou'
  tentativas            INT NOT NULL DEFAULT 1,
  erro_ultimo           TEXT,
  materializado_em      TIMESTAMP,
  UNIQUE (pesquisa_id, numero_versao)
);

-- Sugestão de evolução em decisao_elegibilidade:
-- tipo_decisao VARCHAR(30) NOT NULL      -- 'assincrona_origem' | 'sincrona_digital'
-- blocklist_hit BOOLEAN DEFAULT FALSE
-- janela_quarentena_aplicada INTERVAL
-- prioridade_aplicada VARCHAR(50)
```

---

## 4. Execução de Pesquisas

### Visão de negócio
É o "coração operacional" — o serviço mais complexo.
Ele recebe a oportunidade (ou consulta digital), cria a instância da pesquisa,
gera o token seguro, monta o que o cliente vai ver, registra a resposta,
e controla prazo e limite de respostas.
**Tudo que acontece na vida de uma pesquisa para um cliente passa aqui.**

### Banco sugerido: **Relacional + NoSQL híbrido**

| Tabela | Banco | Motivo |
|---|---|---|
| `instancia_pesquisa`, `token`, `resposta`, `nao_resposta` | **PostgreSQL** | Precisão transacional, controle de estado, integridade |
| `snapshot_pesquisa` | **PostgreSQL ou MongoDB** | O snapshot é um JSON complexo; JSONB do Postgres aguenta; se ficar muito grande, Mongo é melhor |
| `materializacao_contextual` | **Redis (cache TTL)** | Dados de contexto resolvidos via SICLI/SIISO — buscar de novo é caro; cache com expiração é o caminho |

---

### Entidades

```sql
-- A ocorrência concreta de uma pesquisa para um respondente
CREATE TABLE instancia_pesquisa (
  id                  UUID PRIMARY KEY,
  oportunidade_id     UUID,                  -- vem da Seleção (fluxo assíncrono)
  respondente_ref     VARCHAR(100) NOT NULL,  -- pseudonimizado
  versao_pesquisa_id  UUID NOT NULL,
  canal               VARCHAR(50),
  estado              VARCHAR(30) NOT NULL,   -- 'aguardando' | 'respondida' | 'expirada' | 'encerrada_limite' | 'nao_respondida'
  criado_em           TIMESTAMP NOT NULL,
  expira_em           TIMESTAMP,             -- calculado com base nos dias definidos na versão
  limite_respostas    INT,
  total_respostas     INT DEFAULT 0,
  correlation_id      UUID NOT NULL
);

-- Token opaco — credencial que o canal usa para submeter resposta
-- NUNCA logar o valor_hash completo
CREATE TABLE token_pesquisa (
  id                  UUID PRIMARY KEY,
  instancia_id        UUID NOT NULL REFERENCES instancia_pesquisa(id),
  valor_hash          VARCHAR(200) NOT NULL,  -- token opaco, nunca exposto em log
  expira_em           TIMESTAMP NOT NULL,
  usado               BOOLEAN DEFAULT FALSE,
  revogado_em         TIMESTAMP
);

-- Foto imutável do que foi apresentado ao respondente
-- Garante rastreabilidade: "o que exatamente ele viu?"
CREATE TABLE snapshot_pesquisa (
  id                  UUID PRIMARY KEY,
  instancia_id        UUID NOT NULL REFERENCES instancia_pesquisa(id),
  versao_pesquisa_id  UUID NOT NULL,
  conteudo_json       JSONB NOT NULL,  -- perguntas, opções, textos resolvidos
  gerado_em           TIMESTAMP NOT NULL
);

-- Resposta bruta do respondente — exatamente o que ele enviou
-- Ownership exclusivo da Execução. Apuração não acessa essa tabela diretamente.
CREATE TABLE resposta (
  id                  UUID PRIMARY KEY,
  instancia_id        UUID NOT NULL REFERENCES instancia_pesquisa(id),
  token_id            UUID NOT NULL REFERENCES token_pesquisa(id),
  resposta_json       JSONB NOT NULL,  -- alternativas escolhidas, textos livres, etc
  canal               VARCHAR(50),
  submetido_em        TIMESTAMP NOT NULL,
  publicado_em        TIMESTAMP        -- outbox: null = evento ainda não publicado
);

-- Registro quando não houve resposta (expirou, atingiu limite, etc)
CREATE TABLE nao_resposta (
  id                  UUID PRIMARY KEY,
  instancia_id        UUID NOT NULL REFERENCES instancia_pesquisa(id),
  motivo_codigo       VARCHAR(30) NOT NULL,  -- 'expirado_tempo' | 'limite_atingido' | 'encerrado'
  registrado_em       TIMESTAMP NOT NULL,
  publicado_em        TIMESTAMP              -- outbox
);
```

### Evoluções recomendadas (snapshot, concorrência e ciclo de vida)

```sql
-- Histórico de transição de estado da instância
CREATE TABLE historico_estado_instancia (
  id                    UUID PRIMARY KEY,
  instancia_id          UUID NOT NULL REFERENCES instancia_pesquisa(id),
  estado_anterior       VARCHAR(30),
  estado_novo           VARCHAR(30) NOT NULL,
  motivo_transicao      VARCHAR(50),
  alterado_em           TIMESTAMP NOT NULL,
  correlation_id        UUID NOT NULL
);

-- Controle consistente de limite de respostas por versão (evita estouro concorrente)
CREATE TABLE controle_limite_respostas (
  id                    UUID PRIMARY KEY,
  versao_pesquisa_id    UUID NOT NULL,
  limite_respostas      INT NOT NULL,
  total_aceitas         INT NOT NULL DEFAULT 0,
  lock_version          BIGINT NOT NULL DEFAULT 0,
  atualizado_em         TIMESTAMP NOT NULL,
  UNIQUE (versao_pesquisa_id)
);

-- Sugestão de evolução em snapshot_pesquisa:
-- hash_snapshot VARCHAR(64) NOT NULL
-- schema_snapshot_versao VARCHAR(30) NOT NULL
-- template_versao VARCHAR(30)
```

```
-- Cache de materialização contextual (Redis)
-- Chave: instancia_id + placeholder_chave
-- TTL: curto (minutos), nunca persistido além do necessário

SET exec:ctx:{instancia_id}:{placeholder_chave} = "valor_resolvido"
EXPIRE exec:ctx:{instancia_id}:{placeholder_chave} 300  -- 5 minutos
```

---

## 5. Entrega de Convites

### Visão de negócio
Quando a Execução cria uma instância, ela publica um evento dizendo:
*"Existe uma pesquisa esperando esse cliente — alguém precisa convidá-lo."*
A Entrega recebe isso, tenta enviar via provedor (email, SMS, push),
registra o resultado de cada tentativa e, se falhar, agenda retry.
**Ela não decide nada de negócio — só gerencia o ciclo técnico do envio.**

### Banco sugerido: **Relacional (PostgreSQL)**
**Por quê:** Controle de tentativas, estados e retries são dados tabulares simples.
Uma fila de retries pode usar o próprio banco com controle de `proxima_tentativa_em`
ou uma fila externa (RabbitMQ com dead-letter).

---

### Entidades

```sql
-- O convite a ser enviado
CREATE TABLE convite (
  id                  UUID PRIMARY KEY,
  instancia_id        UUID NOT NULL,          -- referência lógica para Execução
  canal               VARCHAR(50) NOT NULL,   -- 'email' | 'sms' | 'push'
  destinatario_ref    VARCHAR(200) NOT NULL,  -- pseudonimizado (hash do email/telefone)
  estado              VARCHAR(30) NOT NULL,   -- 'solicitado' | 'em_envio' | 'enviado' | 'falha' | 'cancelado'
  criado_em           TIMESTAMP NOT NULL,
  maximo_tentativas   INT DEFAULT 3
);

-- Cada tentativa de envio (para rastreabilidade e retry)
CREATE TABLE tentativa_envio (
  id                    UUID PRIMARY KEY,
  convite_id            UUID NOT NULL REFERENCES convite(id),
  numero_tentativa      INT NOT NULL,
  estado                VARCHAR(20) NOT NULL,  -- 'enviado' | 'falha_temporaria' | 'falha_permanente'
  provedor_resposta     TEXT,                  -- resposta bruta do provedor
  tentado_em            TIMESTAMP NOT NULL,
  proxima_tentativa_em  TIMESTAMP,             -- null se não houver retry agendado
  erro_detalhe          TEXT
);
```

### Evoluções recomendadas (operação de retry)

```sql
-- Agenda operacional de retries com backoff
CREATE TABLE agenda_retry_convite (
  id                    UUID PRIMARY KEY,
  convite_id            UUID NOT NULL REFERENCES convite(id),
  tentativa_atual       INT NOT NULL,
  proxima_tentativa_em  TIMESTAMP NOT NULL,
  politica_retry        VARCHAR(50) NOT NULL,   -- ex: 'exponencial_jitter'
  status                VARCHAR(30) NOT NULL,   -- 'agendado' | 'executando' | 'concluido' | 'cancelado'
  atualizado_em         TIMESTAMP NOT NULL
);

-- Sugestão de evolução em tentativa_envio:
-- provider_message_id VARCHAR(200)
-- codigo_retorno_provedor VARCHAR(50)
-- correlation_id UUID
```

---

## 6. Apuração de Resultados

### Visão de negócio
Depois que o cliente respondeu, a Execução publicou um evento com a resposta bruta.
A Apuração consome esse evento, pega os critérios que materializou da Gestão,
e calcula o valor da métrica (ex: nota CSAT = 4, NPS promotor, etc).
Ela também mantém **agregados consolidados** para que dashboards e APIs
de resultado possam consultar sem precisar recalcular tudo do zero.

### Banco sugerido: **Relacional para resultados + NoSQL/OLAP para agregados**

| Tabela | Banco | Motivo |
|---|---|---|
| `materializacao_criterio`, `resultado_apurado` | **PostgreSQL** | Dados estruturados, rastreabilidade por instância |
| `agregado_pesquisa` | **PostgreSQL** (pequeno volume) ou **ClickHouse/BigQuery** (volume alto) | Agregados históricos com queries analíticas pesadas se beneficiam de banco colunar |

---

### Entidades

```sql
-- Cópia local dos critérios de cálculo publicados pela Gestão
CREATE TABLE materializacao_criterio (
  id                  UUID PRIMARY KEY,
  versao_pesquisa_id  UUID NOT NULL,
  criterios_json      JSONB NOT NULL,   -- como transformar resposta bruta em métrica
  materializado_em    TIMESTAMP NOT NULL,
  hash_versao         VARCHAR(64) NOT NULL
);

-- Resultado calculado para cada instância respondida
CREATE TABLE resultado_apurado (
  id                    UUID PRIMARY KEY,
  instancia_id          UUID NOT NULL,   -- referência lógica (não FK cruzada com Execução)
  versao_pesquisa_id    UUID NOT NULL,
  metrica_codigo        VARCHAR(20) NOT NULL,  -- 'CSAT' | 'NPS' | 'NES'
  valor_calculado       DECIMAL(10,4),
  classificacao         VARCHAR(30),     -- 'promotor' | 'neutro' | 'detrator' (NPS), '5_estrelas' (CSAT), etc
  apurado_em            TIMESTAMP NOT NULL,
  baseado_em_resposta_id UUID NOT NULL   -- rastreabilidade
);

-- Visão consolidada para consumidores analíticos e dashboards
-- Atualizado de forma assíncrona à medida que chegam novas respostas
CREATE TABLE agregado_pesquisa (
  id                  UUID PRIMARY KEY,
  pesquisa_id         UUID NOT NULL,
  versao_pesquisa_id  UUID NOT NULL,
  metrica_codigo      VARCHAR(20) NOT NULL,
  canal               VARCHAR(50),       -- null = todos os canais
  total_instancias    INT DEFAULT 0,
  total_respondidas   INT DEFAULT 0,
  total_nao_respondidas INT DEFAULT 0,
  valor_agregado      DECIMAL(10,4),     -- média, score, etc conforme métrica
  periodo_inicio      DATE NOT NULL,
  periodo_fim         DATE NOT NULL,
  calculado_em        TIMESTAMP NOT NULL
);
```

### Evoluções recomendadas (reprocessamento controlado)

```sql
-- Lote de reprocessamento autorizado e auditável
CREATE TABLE lote_reprocessamento (
  id                    UUID PRIMARY KEY,
  solicitante_ref       VARCHAR(100) NOT NULL,
  motivo                TEXT NOT NULL,
  escopo_json           JSONB NOT NULL,         -- período, pesquisas, versões, canais
  status                VARCHAR(30) NOT NULL,   -- 'solicitado' | 'em_execucao' | 'concluido' | 'falhou'
  criado_em             TIMESTAMP NOT NULL,
  concluido_em          TIMESTAMP
);

-- Itens processados dentro do lote
CREATE TABLE item_reprocessamento (
  id                    UUID PRIMARY KEY,
  lote_id               UUID NOT NULL REFERENCES lote_reprocessamento(id),
  instancia_id          UUID,
  resultado_apurado_id  UUID,
  status_item           VARCHAR(30) NOT NULL,   -- 'processado' | 'ignorado' | 'falhou'
  detalhe               TEXT,
  processado_em         TIMESTAMP
);

-- Sugestão de evolução em resultado_apurado:
-- event_id_resposta_origem UUID NOT NULL
-- algoritmo_versao VARCHAR(30) NOT NULL
-- calculo_hash VARCHAR(64) NOT NULL
-- reprocessado_flag BOOLEAN DEFAULT FALSE
-- lote_reprocessamento_id UUID
```

---

## Resumo geral: banco por serviço

| Microsserviço | Banco principal | Complemento | Motivo principal |
|---|---|---|---|
| Gestão de Pesquisas | PostgreSQL | — | Estrutura relacional, integridade transacional na publicação |
| Ingestão de Origens | PostgreSQL | Fila (Kafka/RabbitMQ) | Rastreabilidade de entradas, volume previsível |
| Seleção e Elegibilidade | PostgreSQL | Redis (quarentena lookup) | Decisões auditáveis + lookup rápido de quarentena |
| Execução de Pesquisas | PostgreSQL | Redis (cache contextual) | Transações de estado + cache de placeholders |
| Entrega de Convites | PostgreSQL | — | Tabela de tentativas e retries simples |
| Apuração de Resultados | PostgreSQL | ClickHouse/BigQuery (se volume alto) | Resultados rastreáveis + agregados analíticos |

---

## Persistência de eventos por domínio (Outbox/Inbox)

Além da persistência de negócio, cada microsserviço pode manter sua persistência operacional de eventos.

- **Outbox** fica no domínio que produz o evento.
- **Inbox** fica no domínio que consome o evento.
- Isso não substitui o banco de negócio do serviço; é um controle para publicação, consumo, idempotência e reprocessamento.

Leitura conceitual por serviço:

- **Gestão de Pesquisas**: tende a ter **Outbox** para publicar pesquisa publicada, versão publicada ou revogação.
- **Ingestão de Origens**: tende a ter **Outbox** para publicar origem registrada ou rejeição técnica.
- **Seleção e Elegibilidade**: tende a ter **Inbox** para consumir origens recebidas e **Outbox** para publicar oportunidade quando houver elegibilidade.
- **Execução de Pesquisas**: tende a ter **Inbox** para consumir oportunidade e **Outbox** para publicar instância criada, resposta registrada e não-resposta/encerramento.
- **Entrega de Convites**: tende a ter **Inbox** para consumir instância criada e, se publicar status operacional, também **Outbox** para envio solicitado, enviado, falha ou retry.
- **Apuração de Resultados**: tende a ter **Inbox** para consumir resposta/não-resposta e, se publicar fatos derivados, também **Outbox** para resultados calculados ou agregados atualizados.

Regra prática:

1. Se o domínio **publica** evento, ele precisa de um controle local de **Outbox** ou mecanismo equivalente.
2. Se o domínio **consome** evento, ele precisa de **Inbox** ou mecanismo equivalente para evitar duplicidade.
3. O evento não pertence ao broker; ele nasce no domínio e é persistido ali antes da publicação, ou é registrado ali quando recebido.

### Tabelas conceituais de apoio

Estas tabelas são conceituais e representam o controle operacional de eventos dentro do próprio domínio.
Elas não substituem as tabelas de negócio.

```sql
-- Controle de eventos a publicar pelo domínio
CREATE TABLE outbox_evento (
  id                  UUID PRIMARY KEY,
  nome_evento         VARCHAR(100) NOT NULL,   -- ex: pesquisa_publicada, resposta_registrada
  aggregate_id        VARCHAR(200),            -- id do fato de negócio que originou o evento
  tipo_agregado       VARCHAR(100),            -- ex: pesquisa, instancia, resposta
  payload_json        JSONB NOT NULL,          -- evento pronto para publicação
  versao_evento       VARCHAR(20) NOT NULL,    -- versão do contrato do evento
  schema_evento       VARCHAR(50) NOT NULL,    -- ex: survey.version-published.v1
  correlation_id      UUID NOT NULL,
  chave_particionamento VARCHAR(200),          -- ordenação estável no broker
  checksum_payload    VARCHAR(64),
  status_publicacao   VARCHAR(30) NOT NULL,    -- pendente | publicado | falhou
  tentativas_publicacao INT DEFAULT 0,
  proxima_tentativa_em TIMESTAMP,
  criado_em           TIMESTAMP NOT NULL,
  publicado_em        TIMESTAMP,
  erro_ultimo         TEXT,
  dead_letter_em      TIMESTAMP
);

-- Controle de eventos recebidos e processados pelo domínio
CREATE TABLE inbox_evento (
  id                  UUID PRIMARY KEY,
  nome_evento         VARCHAR(100) NOT NULL,   -- ex: origem_registrada, oportunidade_criada
  message_id          VARCHAR(200) NOT NULL,   -- identificador único da mensagem do broker
  chave_idempotencia  VARCHAR(200) NOT NULL,   -- chave usada para evitar duplicidade funcional
  consumidor_nome     VARCHAR(100) NOT NULL,   -- serviço/worker consumidor
  origem_evento       VARCHAR(100),            -- serviço/tema de onde veio
  payload_json        JSONB,
  versao_evento       VARCHAR(20) NOT NULL,
  schema_evento       VARCHAR(50),
  checksum_payload    VARCHAR(64),
  correlation_id      UUID NOT NULL,
  status_processamento VARCHAR(30) NOT NULL,   -- recebido | processando | processado | rejeitado
  recebido_em         TIMESTAMP NOT NULL,
  processado_em       TIMESTAMP,
  erro_ultimo         TEXT
);

-- Índices recomendados (conceituais)
-- UNIQUE (consumidor_nome, message_id)
-- UNIQUE (consumidor_nome, chave_idempotencia)
```

---

## Regras arquiteturais que impactam o modelo

1. **Nenhum serviço acessa a tabela de outro.** As únicas ligações entre serviços são via eventos ou chamadas de API.
2. **Respondente nunca com CPF/CNPJ cru** — sempre `respondente_ref` (hash ou pseudônimo).
3. **Token nunca aparece completo em log** — só sufixo ou referência por `id`.
4. **Outbox nas tabelas críticas** — campo `publicado_em` null enquanto o evento não foi confirmado enviado ao broker.
5. **Versão publicada é imutável** — qualquer mudança cria nova versão; isso impacta o `versao_pesquisa_id` presente em quase todas as tabelas.

---

## Decisões arquiteturais de persistência por domínio

1. Identidade canônica de versão
- Em consumidores da definição (Seleção, Execução e Apuração), armazenar pelo menos:
  - `pesquisa_id`
  - `numero_versao`
  - `versao_pesquisa_id`
  - `versao_publicada_em`
- Objetivo: rastreabilidade e reconciliação entre domínios sem consulta cruzada de banco.

2. Materialização é persistência local derivada
- Materializar não significa obrigatoriamente uma coluna JSON.
- Pode ser tabela normalizada, projeção desnormalizada, JSONB ou modelo híbrido.
- A decisão é local ao microserviço e deve otimizar uso do domínio, não replicar o banco da Gestão.

3. Outbox e Inbox são persistências operacionais mandatórias nos fluxos críticos
- Publicador crítico: Outbox com status, retry e dead-letter.
- Consumidor crítico: Inbox com dedupe por `message_id` e `chave_idempotencia` por consumidor.
- Objetivo: tolerar reentrega sem duplicidade funcional.

4. Execução deve preservar integridade de limite e ciclo de vida
- Controle de concorrência para limite de respostas.
- Histórico de transição de estado para auditoria de expiração, não-resposta e encerramento.

5. Apuração deve suportar reprocessamento governado
- Reprocessamento com autorização, escopo e trilha auditável.
- Resultados históricos não devem mudar sem operação explícita de reprocessamento.

6. Evento para Entrega deve refletir intenção operacional
- Preferir nome semântico como `instancia_disponibilizada_para_entrega`.
- Evitar ambiguidade com evento técnico genérico de "instância criada".
