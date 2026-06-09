# Domínios e Responsabilidades

## Gestão de Pesquisas

Define, valida, versiona e publica pesquisas, regras, métricas, templates e blocklists.
É a fonte de verdade da definição.

Não decide elegibilidade, não instancia pesquisa e não materializa placeholders para respondentes.

## Ingestão de Origens de Pesquisa

Recebe, valida tecnicamente e normaliza origens autorizadas vindas de sistemas externos ou arquivos.
Publica o fato normalizado para a Seleção.

Não aplica blocklist, quarentena, amostragem ou prioridade.

## Seleção de Público e Elegibilidade

Aplica as regras que decidem se uma origem ou jornada digital pode gerar pesquisa.
É o domínio autoritativo de elegibilidade.

Não cria instância, não gera token e não coleta resposta.

## Execução de Pesquisas

Controla instância, token, snapshot, materialização contextual, resposta, não-resposta e expiração.
Consulta a Seleção no fluxo digital síncrono.

Não governa elegibilidade e não calcula métrica.

## Entrega de Convites

Executa o envio técnico dos convites e controla tentativas, sucesso, falha e retry.

Não decide público nem reavalia regras de negócio.

## Apuração de Métricas e Resultados

Calcula resultados a partir de respostas e não-respostas, seguindo a política de métrica da versão executada.

Não recebe resposta do canal e não participa da experiência de resposta.
