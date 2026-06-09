# Sistema de Métricas de Percepção

Este repositório concentra o material de arquitetura do DAS da solução de métricas de percepção.
O objetivo deste índice é servir como porta de entrada humana e como mapa de leitura para o GitHub Copilot.

## Como usar este material

Leia primeiro as regras duras em [copilot-instructions.md](copilot-instructions.md).
Depois navegue pelo contexto global e, por fim, abra o documento do serviço que você estiver trabalhando.

## Ordem recomendada de leitura

1. [Visão Geral do DAS](.github/ai/00-visao-geral-das.md)
2. [Objetivos e Escopo](.github/ai/01-objetivos-e-escopo.md)
3. [Domínios e Responsabilidades](.github/ai/02-dominios-e-responsabilidades.md)
4. [Integrações e Comunicação](.github/ai/03-integracoes-e-comunicacao.md)
5. [Eventos, Idempotência e Consistência](.github/ai/04-eventos-idempotencia-e-consistencia.md)
6. [Segurança, Privacidade e Tokens](.github/ai/05-seguranca-privacidade-e-tokens.md)
7. [Runtime e Fluxos](.github/ai/06-runtime-e-fluxos.md)
8. [Infraestrutura, Operação e Observabilidade](.github/ai/07-infra-operacao-e-observabilidade.md)
9. [Riscos, Lacunas e Decisões Pendentes](.github/ai/08-riscos-lacunas-e-decisoes-pendentes.md)
10. [Glossário](.github/ai/09-glossario.md)
11. [ADRs em Resumo](.github/ai/10-adrs-resumo.md)
12. [Regras de Implementação](.github/ai/11-regras-de-implementacao.md)
13. [Checklist de Arquitetura](.github/ai/12-checklist-de-arquitetura.md)
14. [Microserviços Explicados para Dev](.github/ai/13-microservicos-explicado-para-dev.md)

## Diagramação de apoio

O DAS tem apoio visual nos diagramas abaixo:

- [Contexto técnico](IMGS/c4_contexto.svg)
- [Containers](IMGS/c4_containers.svg)
- [Publicação e materialização](IMGS/seq_publicacao_pesquisa.svg)
- [Origem assíncrona com elegibilidade](IMGS/seq_origem_assincrona_com_elegibilidade.svg)
- [Jornada digital com elegibilidade síncrona](IMGS/seq_origem_sincrona.svg)
- [Submissão de resposta e apuração assíncrona](IMGS/seq_resposta_e_apuracao.svg)
- [Expiração ou não-resposta](IMGS/seq_nao_resposta.svg)
- [Falha e reentrega de evento](IMGS/seq_falha_e_reentrega.svg)

## Kits por serviço

Cada microserviço deve carregar o mesmo contexto global e o seu próprio recorte local.

- [Gestão de Pesquisas](.github/ai/servicos/01-gestao-pesquisas.md)
- [Ingestão de Origens](.github/ai/servicos/02-ingestao-origens.md)
- [Seleção e Elegibilidade](.github/ai/servicos/03-selecao-elegibilidade.md)
- [Execução de Pesquisas](.github/ai/servicos/04-execucao-pesquisas.md)
- [Entrega de Convites](.github/ai/servicos/05-entrega-convites.md)
- [Apuração de Resultados](.github/ai/servicos/06-apuracao-resultados.md)

## Resumo operacional

Este DAS separa claramente definição, decisão e execução.
Gestão define e publica.
Seleção decide elegibilidade.
Execução instancia, materializa e coleta respostas.
Entrega aciona comunicação.
Apuração calcula resultados.

Se alguma implementação cruzar essas fronteiras, ela precisa ser revista antes de seguir adiante.
