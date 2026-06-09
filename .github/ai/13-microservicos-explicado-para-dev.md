# Microserviços Explicados para Dev

Este arquivo traduz o DAS para uma visão prática de implementação.
Use este material quando você precisar entender rapidamente qual microserviço deve receber uma regra, endpoint, evento ou dado.

## Visão rápida

- Gestão define a pesquisa.
- Ingestão recebe origens.
- Seleção decide elegibilidade.
- Execução cria a experiência de resposta.
- Entrega envia convite.
- Apuração calcula resultado.

Se uma mudança não respeita essa ordem de responsabilidades, provavelmente está no serviço errado.

## 1) Gestão de Pesquisas

### O que este serviço representa

É o lugar onde a pesquisa nasce como definição oficial: perguntas, versões, regras, métricas, templates e configurações governadas.

### Quando o dev mexe nele

- Criar ou alterar cadastro de pesquisa
- Publicar nova versão
- Validar definição antes de publicar
- Manter catálogo de templates e placeholders permitidos

### Entrada típica

- Comandos administrativos do front-end de gestão

### Saída típica

- Versão de pesquisa publicada para outros domínios materializarem

### Não deve fazer

- Decidir elegibilidade por respondente
- Criar instância de pesquisa
- Receber resposta do canal
- Calcular resultado

## 2) Ingestão de Origens

### O que este serviço representa

É a porta técnica de entrada para fatos que podem originar pesquisa: atendimento, integração externa ou lista proativa.

### Quando o dev mexe nele

- Criar endpoint de recepção
- Processar arquivo de entrada
- Validar esquema técnico
- Normalizar payload de origem

### Entrada típica

- API HTTP
- Arquivo batch

### Saída típica

- Origem registrada e normalizada para a Seleção

### Não deve fazer

- Tomar decisão de elegibilidade
- Aplicar blocklist/quarentena/amostragem/prioridade
- Criar oportunidade ou instância

## 3) Seleção e Elegibilidade

### O que este serviço representa

É o domínio de decisão. Aqui mora a regra de quem pode ou não receber pesquisa.

### Quando o dev mexe nele

- Implementar regra de elegibilidade
- Aplicar blocklist
- Aplicar quarentena
- Aplicar amostragem e prioridade
- Expor decisão síncrona para jornada digital

### Entrada típica

- Origem registrada da Ingestão
- Consulta síncrona da Execução em jornada digital

### Saída típica

- Decisão de elegibilidade
- Oportunidade criada quando elegível

### Não deve fazer

- Criar instância
- Gerar token
- Materializar mensagem
- Receber resposta

## 4) Execução de Pesquisas

### O que este serviço representa

É o runtime da pesquisa. Ele transforma a decisão positiva em experiência real para o respondente.

### Quando o dev mexe nele

- Criar/recuperar instância
- Validar token, status e expiração
- Materializar snapshot e template
- Consultar SICLI/SIISO/LDAP para contexto autorizado
- Receber e persistir resposta bruta
- Publicar fatos operacionais de resposta e encerramento

### Entrada típica

- Chamada do canal digital
- Oportunidade criada no fluxo assíncrono

### Saída típica

- Pesquisa materializada para o canal
- Resposta registrada
- Fatos de não-resposta/encerramento

### Não deve fazer

- Assumir regra de elegibilidade como dona
- Calcular resultado de métrica
- Expor dados sensíveis em logs

## 5) Entrega de Convites

### O que este serviço representa

É o operador do envio técnico de convites nos canais de comunicação.

### Quando o dev mexe nele

- Integrar com provedor de comunicação
- Controlar tentativas e retry
- Registrar sucesso/falha de envio

### Entrada típica

- Informação operacional para envio de convite

### Saída típica

- Acionamento do provedor de comunicação
- Estado do ciclo de envio

### Não deve fazer

- Decidir elegibilidade
- Criar instância
- Alterar questionário
- Calcular resultado

## 6) Apuração de Resultados

### O que este serviço representa

É o domínio que transforma resposta em indicador.

### Quando o dev mexe nele

- Consumir resposta registrada
- Aplicar política de cálculo da métrica
- Persistir resultado e agregados
- Expor API de resultados consolidados

### Entrada típica

- Eventos de resposta e não-resposta

### Saída típica

- Resultado por item
- Agregados e indicadores

### Não deve fazer

- Receber resposta direto do canal
- Tomar decisão de elegibilidade
- Ler base interna de outro serviço

## Decisão rápida para implementação

Use esta regra de bolso:

- Definição da pesquisa: Gestão
- Entrada técnica de origem: Ingestão
- Decisão de pode/não pode: Seleção
- Experiência do respondente: Execução
- Envio técnico de convite: Entrega
- Cálculo e consolidação de indicadores: Apuração

## Erros comuns que geram retrabalho

- Colocar regra de elegibilidade no canal ou na Execução
- Fazer apuração dentro da Execução
- Ler banco de outro domínio
- Tratar evento como detalhe privado sem versionamento
- Registrar PII, token ou URL sensível em telemetria
