Tenho uma aplicação .NET Worker responsável por processar atendimentos de clientes e gerar pesquisas de satisfação. Os dados chegam por meio de uma carga ETL contendo registros de atendimentos realizados em agências bancárias.

Cada registro precisa passar por várias etapas: validações de regras de negócio, chamadas a APIs externas para enriquecimento de dados (cliente e unidade), criação da pesquisa de satisfação em uma API externa e envio de push ao cliente. A cada etapa, o sistema atualiza o status do registro e grava o histórico em uma tabela separada, tudo dentro de transações usando EF Core.

O processamento atual é feito por meio de um pipeline sequencial. Os registros são buscados no banco e cada um percorre todas as etapas em ordem. Um registro só começa a ser processado quando o anterior finaliza completamente. Apesar de a aplicação rodar em múltiplos pods e já existir controle de lock/claim no banco, cada pod ainda processa poucos registros por vez. Como o fluxo depende fortemente de chamadas a APIs externas, o worker passa muito tempo aguardando respostas, o que gera baixo throughput e acúmulo de registros pendentes.

O problema principal não está nas regras de negócio nem no modelo de dados, mas na forma como o fluxo é orquestrado. O pipeline sequencial impede que diferentes etapas sejam executadas em paralelo, mesmo sendo majoritariamente I/O bound.

A solução proposta é substituir o pipeline sequencial por um modelo baseado em Channels internos do .NET. Cada etapa do processo passa a ter seu próprio Channel, com múltiplos consumidores concorrentes. Um registro é consumido por uma etapa, processado, tem seu status e histórico atualizados e, em caso de sucesso, é enviado para o próximo Channel. Em caso de falha, o fluxo daquele registro é encerrado.

Com isso, vários registros podem estar em etapas diferentes ao mesmo tempo, aumentando significativamente o throughput, melhor aproveitando o tempo de espera das APIs externas e permitindo escalabilidade tanto horizontal (mais pods) quanto vertical (mais consumidores por etapa), sem necessidade de filas externas e sem alterar regras de negócio ou persistência existentes.
