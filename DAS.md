Documento de Arquitetura de Software — Sistema de Métricas de Percepção¶
1. Introdução e Objetivos¶
O Sistema de Métricas de Percepção tem como objetivo prover uma plataforma corporativa para configuração, execução, coleta e apuração de pesquisas de percepção aplicáveis a clientes e funcionários. A solução deve suportar CSAT, NPS, NES e futuras métricas sem redesenhar o núcleo da plataforma.

A arquitetura será organizada em microsserviços cuja separação acompanha os domínios da aplicação: Gestão de Pesquisas, Ingestão de Origens de Pesquisa, Seleção de Público e Elegibilidade, Execução de Pesquisas, Entrega de Convites e Apuração de Métricas e Resultados. A decomposição está descrita na seção 5 e formalizada na ADR-001.

1.1 Objetivos Funcionais em Nível Arquitetural¶
O sistema deve permitir a gestão governada e versionada das pesquisas de percepção, preservando a separação entre o que é definido administrativamente e o que é executado em runtime. Detalhes funcionais de operação da gestão não pertencem ao DAS principal e devem ser tratados em artefatos acessórios.

A solução deve suportar origens de pesquisa distintas, sem deslocar a decisão de elegibilidade para os canais ou para a Ingestão. As responsabilidades dos blocos e os fluxos representativos são detalhados nas seções 5.3 e 6.

A solução deve permitir materialização contextual de mensagens por meio de placeholders controlados, mantendo a definição na Gestão e a resolução na Execução. O conceito é tratado na seção 8.4 e formalizado na ADR-012.

A solução deve preservar rastreabilidade ponta a ponta entre os fatos relevantes da pesquisa. Os conceitos de auditoria, eventos e idempotência estão consolidados nas seções 8.2 e 8.3.

2. Restrições Arquiteturais¶
2.1 Restrições organizacionais e tecnológicas¶
A organização possui preferência explícita por microsserviços. A solução deve ser decomposta conforme os domínios da aplicação, evitando microsserviços por entidade, tabela ou operação CRUD. A decisão correspondente está registrada na ADR-001.

A solução deverá ser implantada em contêineres orquestrados por Kubernetes, com configuração externalizada por ambiente, health checks, heart beat ou sinais operacionais equivalentes e persistência fora do filesystem efêmero dos pods.

Cada serviço deve possuir persistência lógica própria. Mesmo que a infraestrutura física seja compartilhada, a arquitetura não permitirá acesso direto de um serviço às estruturas internas de outro. A decisão correspondente está registrada na ADR-004.

2.2 Restrições de comunicação e integração¶
A comunicação será híbrida. APIs síncronas serão usadas quando houver usuário, canal ou consumidor aguardando resposta imediata. Eventos assíncronos serão usados para propagação de fatos de negócio, processamento desacoplado, retry e isolamento de falhas. A decisão correspondente está registrada na ADR-002.

Esta seção registra apenas a restrição arquitetural: chamadas síncronas devem ser reservadas a interações que exigem resposta imediata, e eventos devem ser usados para desacoplamento, reprocessamento e isolamento de falhas. A definição de quais blocos se comunicam por APIs ou eventos está consolidada na seção 5.4, e os fluxos representativos aparecem na seção 6.

Eventos de integração devem ser versionados e conter envelope mínimo de identificação, versionamento, rastreabilidade e idempotência. A decisão correspondente está registrada na ADR-005; o payload completo deve ficar no catálogo de eventos.

Chamadas digitais para consulta de pesquisa devem possuir mecanismo de proteção contra excesso de tráfego, chamadas repetitivas ou uso indevido. A aplicação de rate limit deve ocorrer preferencialmente na borda corporativa, como APIM ou gateway equivalente, usando critérios tecnicamente disponíveis e confiáveis no contrato de entrada. Limites numéricos, política APIM, resposta HTTP e parâmetros operacionais devem ser definidos em documento técnico ou runbook.

A solução poderá utilizar cache e projeções locais para resiliência dos fluxos digitais síncronos, conforme seção 8.5 e ADR-015. Esse mecanismo não substitui a Seleção como domínio de decisão nem deve substituir controles consistentes da Execução para limite de respostas.

2.3 Restrições de segurança, privacidade e operação¶
Dados pessoais devem ser minimizados em eventos, logs, traces, cache e mensagens de erro. Sempre que possível, os serviços devem operar com respondenteRef ou referência pseudonimizada, evitando CPF, e-mail, telefone e nome completo em payloads internos.

Tokens e links de pesquisa devem ser opacos, expiráveis e tratados como credenciais sensíveis, conforme ADR-011. Logs e traces não devem registrar token completo nem URL completa quando ela contiver credencial de acesso.

APIs devem expor readiness e liveness. Workers devem disponibilizar hearth beat ou sinal operacional equivalente, permitindo identificar processo ativo, falha de consumo, dependência indisponível, backlog, outbox acumulada ou incapacidade de processamento.

3. Contexto e Escopo¶
3.1 Contexto de Negócio¶
O Sistema de Métricas de Percepção se posiciona como uma plataforma corporativa para operacionalizar pesquisas de percepção a partir de diferentes origens de negócio. Ele atua entre as áreas responsáveis pela definição das pesquisas, os canais que apresentam a experiência ao respondente, as origens corporativas que indicam oportunidades de pesquisa, os meios de comunicação utilizados para convites e os consumidores analíticos que acompanham os resultados.

O papel de negócio da solução é garantir que a coleta de percepção ocorra de forma governada, rastreável e consistente. O sistema não define a estratégia corporativa de CX, nem substitui os canais, cadastros, sistemas de atendimento ou plataformas analíticas. Ele fornece a capacidade arquitetural para transformar definições de pesquisa e fatos originadores em experiências controladas de resposta e resultados consolidados.

Fronteira de negócio¶
A fronteira de negócio da solução começa na definição governada da pesquisa e termina na disponibilização dos resultados consolidados. Dentro dessa fronteira estão a publicação das definições, a recepção de origens de pesquisa, a decisão de elegibilidade, a criação da instância, a entrega ou apresentação da pesquisa, o registro da resposta e a apuração dos resultados.

Ficam fora dessa fronteira a operação própria dos canais digitais, a gestão de clientes, unidades e funcionários, a execução dos atendimentos que originam pesquisas, a estratégia corporativa de relacionamento e a exploração analítica avançada fora dos resultados consolidados disponibilizados pela solução.

Escopo incluído¶
O escopo inclui as capacidades necessárias para gerir, executar e apurar pesquisas de percepção em nível corporativo. Isso abrange a gestão governada das definições de pesquisa, a recepção de origens autorizadas, a aplicação das regras de elegibilidade, a materialização da experiência de resposta, a coleta das respostas, o tratamento de não-respostas e a disponibilização de resultados consolidados.

Também fazem parte do escopo as regras transversais que impedem ou limitam a geração de pesquisas, como blocklist, quarentena, amostragem, expiração e limite de respostas. Essas regras não são tratadas como funcionalidades isoladas, mas como controles de governança da experiência de pesquisa.

A solução deve suportar tanto pesquisas motivadas por fatos operacionais, como atendimentos, quanto pesquisas originadas por listas proativas autorizadas. Em todos os casos, a geração de oportunidade ou instância permanece condicionada às regras de elegibilidade definidas no domínio de Seleção.

Escopo excluído¶
A solução não substitui sistemas que são fonte de dados ou canais de interação. Sistemas de atendimento continuam responsáveis pela execução e registro dos atendimentos; canais digitais continuam responsáveis pela jornada principal do usuário; cadastros corporativos continuam responsáveis pelos dados mestres; plataformas de comunicação continuam responsáveis pelo envio técnico; e ambientes analíticos externos continuam responsáveis por análises avançadas que ultrapassem os resultados consolidados expostos pela solução.

Também não faz parte do escopo do DAS principal detalhar telas, layouts de arquivo, contratos completos de APIs, payloads de eventos, regras de apresentação visual da pesquisa ou políticas operacionais específicas. Esses elementos devem ser descritos em documentos funcionais, catálogos técnicos ou runbooks.

Premissas de negócio¶
A pesquisa é tratada como uma experiência versionada de coleta de percepção. Ela não se confunde com a métrica, com o convite, com a oportunidade ou com o resultado. Essa distinção é importante para permitir que uma mesma pesquisa utilize diferentes métricas, que uma oportunidade não gere necessariamente resposta e que os resultados possam ser apurados de forma independente da coleta.

A elegibilidade é uma decisão de negócio centralizada. Canais, origens de pesquisa e módulos de ingestão não devem replicar regras de público, bloqueio, quarentena ou amostragem. Essa centralização preserva consistência entre fluxos físicos, proativos e digitais.

A coleta de percepção deve respeitar limites de exposição do respondente. Por isso, regras como blocklist, quarentena, expiração e limite de respostas fazem parte do escopo de governança da solução, ainda que detalhes de configuração e operação pertençam a artefatos acessórios.

3.2 Contexto Técnico¶
A solução será composta por serviços de aplicação, processamento assíncrono, persistência por domínio, mensageria e integrações externas. As interfaces síncronas serão expostas por mecanismos corporativos de entrada e autorização.

No contexto técnico, devem aparecer apenas os sistemas vizinhos e interfaces externas relevantes para delimitar a solução. A decomposição interna dos serviços, seus bancos e suas relações detalhadas são tratadas na seção 5.

Interfaces técnicas externas¶
Interface	Tipo	Relação com a solução	Observação arquitetural
API de Gestão	HTTP	Interface administrativa	Expõe capacidades de gestão governada das pesquisas.
API de Ingestão	HTTP/arquivo	Entrada de origens autorizadas	Recebe atendimentos, integrações externas ou listas proativas sem decidir elegibilidade.
API de Execução	HTTP	Interface dos canais digitais	Suporta consulta/materialização de pesquisa e submissão de resposta.
API de Resultados	HTTP	Interface de consumidores analíticos autorizados	Expõe resultados consolidados sem acesso direto às bases internas.
Broker de eventos	Mensageria	Integração entre domínios internos	Propaga fatos de negócio entre microsserviços.
Plataforma de comunicação	API ou integração corporativa	Dependência da Entrega de Convites	Executa envio técnico de convites.
IAM	Integração corporativa	Proteção das APIs expostas	Autenticação e autorização.
Fontes corporativas de contexto	API/LDAP	Dependência da Execução de Pesquisas	SICLI, SIISO e LDAP fornecem dados contextuais autorizados para materialização.
C4 — Contexto Técnico¶
Contexto

4. Estratégia de Solução¶
A estratégia é preservar fronteiras claras entre os domínios da aplicação. As responsabilidades estão consolidadas na seção 5.3; aqui são destacados apenas os direcionadores estruturantes.

4.1 Decomposição por domínios da aplicação¶
A decomposição segue os domínios descritos na seção 5.1. A decisão arquitetural associada a essa divisão está registrada na ADR-001.

4.2 Publicação e materialização¶
A Gestão publica versões imutáveis. Seleção, Execução e Apuração materializam localmente as partes da definição necessárias ao seu domínio. A decisão evita consulta síncrona à Gestão durante fluxos operacionais, mas exige monitoramento de materialização e consistência eventual. A decisão correspondente está registrada na ADR-003.

4.3 Elegibilidade síncrona para jornadas digitais¶
Para fluxos assíncronos, a elegibilidade é avaliada a partir de origens recebidas pela Ingestão. Para jornadas digitais, a Execução consulta a Seleção por API ou gRPC antes de materializar a pesquisa. A decisão preserva a Seleção como dona das regras e está registrada na ADR-013.

4.4 Templates e placeholders¶
A estratégia de solução permite mensagens parametrizadas por placeholders controlados, com definição na Gestão e resolução pela Execução. O detalhamento conceitual fica concentrado na seção 8.4, e a decisão arquitetural correspondente está registrada na ADR-012.

Esta seção registra apenas o posicionamento estratégico: templates não devem acoplar a Gestão às fontes corporativas de contexto, e canais não devem assumir responsabilidade por materialização de mensagens.

4.5 Persistência, eventos e resiliência¶
A solução adota persistência própria por domínio, eventos versionados e mecanismos de idempotência nos fluxos críticos. As responsabilidades por persistência aparecem na seção 5.1; os conceitos transversais de eventos, idempotência e consistência estão consolidados na seção 8.3; as decisões formais correspondentes estão registradas nas ADR-004, ADR-005 e ADR-010.

5. Visão de Blocos de Construção¶
Esta seção apresenta a decomposição estática da solução e as fronteiras entre seus principais blocos. O nível de detalhe é suficiente para orientar responsabilidades e dependências arquiteturais, sem antecipar especificações de implementação.

5.1 Visão Sistêmica — Caixa Branca¶
Bloco	Responsabilidade arquitetural	Persistência própria	Principal forma de integração
Gestão de Pesquisas	Define, valida, versiona e publica pesquisas, regras, métricas, templates e blocklists.	Sim	API administrativa e evento de publicação.
Ingestão de Origens de Pesquisa	Recebe, valida tecnicamente e normaliza atendimentos ou listas proativas vindas de origens autorizadas.	Sim	API/arquivo de entrada e evento de origem registrada.
Seleção de Público e Elegibilidade	Avalia elegibilidade, blocklist, quarentena, amostragem e prioridade.	Sim	Eventos assíncronos e API/gRPC para jornada digital.
Execução de Pesquisas	Controla instância, token, snapshot, materialização contextual, resposta, não-resposta e expiração.	Sim	API de execução, eventos e consultas a fontes de contexto.
Entrega de Convites	Envia convites e controla tentativas, sucesso, falha e retry.	Sim	Evento de instância criada e integração com plataforma de comunicação.
Apuração de Métricas e Resultados	Calcula resultados por política de métrica e expõe agregados.	Sim	Eventos de resposta/não-resposta e API de resultados.
A persistência própria representa ownership lógico dos dados. A separação física das bases pode depender da plataforma, mas o acesso direto entre persistências de domínios diferentes não é permitido.

5.2 C4 — Containers¶
O diagrama de containers mostra os elementos executáveis e as dependências estruturais da solução. A topologia física de implantação é tratada na seção 7.

Para preservar legibilidade, o diagrama evita representar todas as conexões transversais. Temas como segurança, segredos e observabilidade são tratados nas seções próprias.

Containers

5.3 Responsabilidades e limites dos blocos¶
Esta seção define o ownership de cada bloco da solução. O objetivo é deixar claro qual domínio toma decisões, quais dados estão sob sua responsabilidade e quais responsabilidades ficam explicitamente fora de sua fronteira. Os fluxos de execução são tratados na seção 6, e as decisões formais aparecem no item 9.

5.3.1 Gestão de Pesquisas¶
A Gestão de Pesquisas é o domínio responsável pela definição governada da pesquisa. Ela mantém a versão autoritativa das configurações que orientam a execução, a elegibilidade e a apuração, sem participar dos fluxos operacionais de decisão por respondente.

Sua responsabilidade central é permitir que uma pesquisa seja definida, validada, versionada e publicada de forma controlada. Uma versão publicada deve ser tratada como referência imutável para os demais domínios. A partir dela, a Seleção materializa regras de elegibilidade, a Execução materializa a experiência de resposta e a Apuração materializa critérios necessários ao cálculo dos resultados.

A Gestão também é responsável por manter configurações governadas que afetam a geração de pesquisas, como regras gerais, templates, critérios de expiração e blocklists. No entanto, manter a configuração não significa executar a decisão operacional. A blocklist, por exemplo, é cadastrada ou governada na Gestão, mas aplicada pela Seleção no momento da decisão de elegibilidade.

A Gestão não deve resolver placeholders para respondentes específicos, não deve consultar SICLI, SIISO ou LDAP, não deve decidir se uma origem ou jornada gera pesquisa, não deve criar instâncias, não deve receber respostas e não deve calcular resultados. Essas exclusões preservam a Gestão como fonte de definição, e não como participante dos fluxos de runtime.

A decisão de manter a Gestão como fonte de verdade da definição está registrada na ADR-003.

5.3.2 Ingestão de Origens de Pesquisa¶
A Ingestão de Origens de Pesquisa é o domínio de entrada técnica das origens que podem motivar uma pesquisa. Ela atua como fronteira anticorrupção entre sistemas externos e o modelo interno da solução, recebendo informações em formatos autorizados e convertendo essas entradas em fatos normalizados.

Uma origem de pesquisa pode representar um atendimento realizado, uma integração externa ou uma lista proativa de respondentes. A diferença entre esses tipos é relevante para rastreabilidade e para a aplicação posterior das regras pela Seleção, mas não muda a responsabilidade principal da Ingestão: validar tecnicamente a entrada, registrar sua recepção e publicar o fato normalizado para processamento assíncrono.

A Ingestão deve preservar rastreabilidade sobre a origem recebida, inclusive em cenários de rejeição técnica, reprocessamento ou duplicidade. Esse controle é importante para auditoria operacional e para evitar que falhas de integração sejam confundidas com decisões de elegibilidade.

A Ingestão não decide se haverá pesquisa, não aplica blocklist, não avalia quarentena, não executa amostragem e não escolhe qual pesquisa será apresentada. Também não cria oportunidade, instância, convite ou resultado. Sua fronteira termina quando a origem foi tecnicamente aceita, normalizada e disponibilizada para a Seleção.

Essa separação garante que listas proativas e atendimentos sigam o mesmo princípio arquitetural: nenhuma origem deve gerar pesquisa sem passar pelo domínio de decisão.

5.3.3 Seleção de Público e Elegibilidade¶
A Seleção de Público e Elegibilidade é o domínio responsável por decidir se uma origem de pesquisa ou uma jornada digital pode gerar pesquisa. Ela concentra as regras que determinam quem pode ser pesquisado, em qual contexto, sob quais restrições e com qual prioridade.

A Seleção materializa as definições publicadas pela Gestão que são necessárias para decisão. A partir dessas definições, aplica regras de elegibilidade, blocklist, quarentena, amostragem e priorização. Quando houver quarentena definida em mais de um nível, como pesquisa e canal, a Seleção deve aplicar o menor prazo vigente e registrar a regra considerada na decisão.

Nos fluxos assíncronos, a Seleção consome origens normalizadas pela Ingestão e decide se elas geram oportunidade de pesquisa. Nas jornadas digitais, a Seleção é consultada de forma síncrona pela Execução para que a decisão ocorra durante a interação do canal. Em ambos os casos, a responsabilidade de decisão permanece no mesmo domínio, evitando que canais ou outros módulos repliquem regras de elegibilidade.

A Seleção deve registrar decisões positivas e negativas de forma auditável. Essa rastreabilidade é necessária para explicar por que uma pesquisa foi gerada, descartada, bloqueada por blocklist, impedida por quarentena ou preterida por alguma regra de prioridade.

A Seleção não cria instância de pesquisa, não gera token, não renderiza questionário, não resolve placeholders, não envia convite, não recebe resposta e não calcula resultado. Ela produz decisão ou oportunidade; a execução da experiência pertence à Execução.

A centralização da elegibilidade na Seleção está formalizada na ADR-007.

5.3.4 Execução de Pesquisas¶
A Execução de Pesquisas é o domínio responsável pela vida operacional da pesquisa para um respondente. Ela transforma uma decisão positiva de elegibilidade em uma experiência concreta de resposta, preservando a versão publicada usada, o estado da instância e a resposta bruta recebida.

A Execução mantém o snapshot renderizável da pesquisa, controla instâncias, tokens, status, expiração, não-respostas e submissões. Também é responsável por impedir respostas após encerramento por prazo ou por limite de respostas. Esse controle é parte da integridade da coleta e não deve ser delegado ao canal.

Nas jornadas digitais, a Execução orquestra a interação com o canal, mas não decide elegibilidade. Antes de materializar a pesquisa, consulta a Seleção. Essa separação evita que regras de público, blocklist, quarentena ou amostragem sejam duplicadas no runtime da experiência.

Quando a pesquisa usa mensagens parametrizadas, a Execução resolve os placeholders no contexto da instância. Para isso, pode consultar fontes corporativas autorizadas, como SICLI, SIISO e LDAP. Essa resolução deve respeitar minimização de dados, fallback e proteção das informações pessoais. O conceito transversal de placeholders está descrito na seção 8.4.

A Execução publica fatos relevantes para os demais domínios, especialmente criação de instância, resposta registrada, não-resposta e encerramento. A resposta bruta permanece sob sua responsabilidade; a interpretação métrica pertence à Apuração.

A Execução não define pesquisa, não governa regras de elegibilidade, não envia convite como responsabilidade primária, não calcula métricas e não deve ser usada como fonte analítica direta por consumidores externos. Sua fronteira é a experiência de resposta e o registro fiel do que ocorreu nessa experiência.

A decisão de manter a Execução como dona da instância, resposta bruta e materialização contextual está registrada na ADR-008.

5.3.5 Entrega de Convites¶
A Entrega de Convites é o domínio operacional responsável por acionar os meios de comunicação configurados para convidar respondentes. Ela existe para isolar a solução das particularidades de provedores, falhas transitórias, tentativas de envio e rastreabilidade operacional dos convites.

A Entrega consome instâncias criadas pela Execução e registra o ciclo operacional do convite. Seu foco é controlar se o envio foi solicitado, aceito, rejeitado, concluído ou falhou tecnicamente. Essa rastreabilidade é importante para diferenciar problemas de comunicação de problemas de elegibilidade ou execução da pesquisa.

A Entrega não decide público, não escolhe pesquisa, não cria instância, não gera token como autoridade de domínio, não altera questionário e não calcula resultado. Também não deve incorporar regras de elegibilidade para decidir se alguém deve ou não receber pesquisa. Se houver necessidade de não enviar, a decisão deve ter ocorrido antes, na Seleção ou na Execução, conforme o caso.

Fallback multicanal, preferências de contato e estratégias avançadas de comunicação devem ser tratados como evolução governada, caso se tornem requisitos formais. O DAS principal mantém apenas a fronteira arquitetural da Entrega.

A decisão de manter a Entrega como serviço operacional especializado está registrada na ADR-009.

5.3.6 Apuração de Métricas e Resultados¶
A Apuração de Métricas e Resultados é o domínio responsável por transformar respostas e não-respostas em resultados interpretáveis. Ela materializa as definições de métrica publicadas pela Gestão e aplica políticas de cálculo compatíveis com a versão da pesquisa executada.

A Apuração deve preservar a rastreabilidade entre resultado, resposta, instância e versão da pesquisa. Essa rastreabilidade permite explicar indicadores consolidados e sustentar reprocessamentos controlados quando necessário.

A Apuração opera de forma assíncrona em relação à submissão da resposta. Isso permite que a Execução confirme o recebimento ao respondente sem aguardar cálculo de métrica. A consistência entre resposta registrada e resultado apurado é eventual e deve ser observável.

A Apuração não recebe respostas diretamente dos canais, não decide elegibilidade, não cria instância, não envia convite e não acessa diretamente a persistência interna da Execução. Ela deve consumir fatos publicados e manter sua própria persistência de resultados e agregados.

Reprocessamentos de resultados devem ser autorizados, auditáveis e limitados por escopo. Alterações cadastrais ou mudanças posteriores em regras não devem alterar automaticamente a interpretação histórica de resultados já apurados. A decisão de reprocessamento controlado está registrada na ADR-016.

5.4 Interfaces arquiteturais principais¶
Esta seção resume apenas interfaces que definem dependências arquiteturais entre blocos. Contratos completos de APIs, payloads de eventos, schemas, códigos de erro e versionamento detalhado devem ficar em documentos acessórios.

Origem	Destino	Contrato arquitetural	Tipo
Gestão	Seleção, Execução e Apuração	Publicação de versão de pesquisa para materialização local	Evento
Ingestão	Seleção	Comunicação de origem registrada	Evento
Execução	Seleção	Avaliação de elegibilidade em jornada digital	API/gRPC síncrono
Seleção	Execução	Comunicação de oportunidade para origens assíncronas	Evento
Canal Digital	Execução	Consulta/materialização de pesquisa e submissão de resposta	API síncrona
Execução	SICLI, SIISO e LDAP	Resolução de dados contextuais para placeholders autorizados	API/LDAP síncrono
Execução	Entrega	Comunicação de instância criada para envio de convite	Evento
Execução	Apuração	Comunicação de resposta ou não-resposta registrada	Evento
Entrega	Plataforma de Comunicação	Solicitação técnica de envio	Integração corporativa
Consumidores autorizados	Apuração	Consulta de resultados consolidados	API síncrona
5.5 Observações sobre nível de detalhamento¶
O DAS principal não deve absorver especificações voláteis de desenvolvimento ou operação. Modelos detalhados, contratos, payloads, parâmetros operacionais e artefatos de implantação devem permanecer em documentos acessórios.

O detalhamento interno de cada módulo pode ser produzido em documentos específicos de módulo quando necessário para orientar construção. Esses documentos devem derivar das fronteiras definidas neste DAS, sem redefinir responsabilidades ou alterar decisões arquiteturais sem ADR correspondente.

O C4 nível 3 fica reservado para documentos acessórios dos módulos que exigirem detalhamento interno. No DAS principal, a prioridade é manter a visão sistêmica legível.

6. Visão de Tempo de Execução¶
Esta seção descreve os fluxos de runtime necessários para compreender as decisões arquiteturais. Os diagramas permanecem no nível de interação entre blocos, sem entrar em detalhes internos de implementação.

6.1 Publicação de pesquisa e materialização¶
Publicação de pesquisa e materialização

A regra de ownership da Gestão está descrita na seção 5.3.1 e na ADR-003. A publicação confirma a versão e o fato de negócio; a materialização nos consumidores é eventual e observável.

6.2 Origem assíncrona com elegibilidade¶
Origem assíncrona com elegibilidade

A Ingestão retorna apenas aceite técnico da origem recebida. A decisão de gerar ou não pesquisa pertence à Seleção, conforme seções 5.3.2 e 5.3.3.

6.3 Jornada digital com elegibilidade síncrona e materialização contextual¶
Jornada digital com elegibilidade síncrona e materialização contextual

A Execução orquestra a jornada digital, mas a Seleção permanece como domínio de decisão, conforme ADR-013. SICLI, SIISO e LDAP são fontes de contexto para materialização, conforme seção 8.4, não fontes de regra de elegibilidade.

Falhas na Seleção devem seguir política de timeout e degradação definida para o canal. Cache de decisão digital, quando usado, deve respeitar as restrições da seção 8.5 e da ADR-015.

6.4 Submissão de resposta e apuração assíncrona¶
Submissão de resposta e apuração assíncrona

A confirmação ao respondente depende da persistência da resposta na Execução, não da conclusão da Apuração, conforme ADR-006.

6.5 Expiração ou não-resposta¶
Submissão de resposta e apuração assíncrona

A não-resposta, a expiração temporal e o encerramento por limite são fatos operacionais distintos, conforme seção 8.3. A regra de ownership da Execução está descrita na seção 5.3.4.

6.6 Falha e reentrega de evento¶
Submissão de resposta e apuração assíncrona

Esse cenário se aplica aos consumidores críticos da solução. Os conceitos de idempotência e consistência estão consolidados na seção 8.3 e na ADR-010.

7. Visão de Implantação¶
A solução será implantada em Kubernetes. APIs e workers serão empacotados como imagens de contêiner e promovidos por pipeline corporativo. A mesma imagem deve ser promovida entre ambientes sempre que possível, variando apenas configuração externa.

A solução deve ter ambientes PRD e NPRD, esse último dividido em DES e TQS, com recursos compartilhados.

7.1 Diretrizes operacionais¶
APIs devem expor readiness e liveness. Workers devem possuir endpoint técnico, heartbeat, métrica ou mecanismo equivalente. Readiness não deve depender de integrações não essenciais para todas as rotas.

A API de Execução possui criticidade especial, pois atende canais digitais e pode depender da Seleção no fluxo síncrono digital e de fontes corporativas para materialização contextual. Essas dependências devem possuir timeout, circuit-breaker, métricas próprias, tratamento de falha e política de resposta degradada.

Segredos devem ser obtidos por cofre corporativo ou mecanismo equivalente. Connection strings, tokens, credenciais de provedor e chaves não devem existir em código, imagem, manifesto aberto ou logs.

Detalhes físicos de plataforma e parâmetros operacionais pertencem aos documentos de implantação e operação.

8. Conceitos Transversais¶
8.1 Segurança e privacidade¶
A solução deve aplicar autenticação corporativa, autorização por perfil, menor privilégio por workload e segregação de persistência. Respostas individuais, tokens, dados resolvidos por placeholders e trilhas por respondente devem ser tratados como sensíveis.

Eventos, logs e traces devem priorizar identificadores técnicos ou pseudonimizados. Dados pessoais só devem trafegar quando houver necessidade explícita, aprovada e documentada.

8.2 Auditoria e rastreabilidade¶
A solução deve permitir rastreabilidade ponta-a-ponta, utilizando identificadores de correlação, para os eventos, entre as etapas.

8.3 Eventos, idempotência e consistência¶
Eventos são contratos públicos versionados. Produtores de eventos críticos devem usar Outbox ou mecanismo equivalente. Consumidores críticos devem usar Inbox, deduplicação e controle por chave de negócio quando necessário.

A solução não usará transações distribuídas. A consistência entre Gestão, Seleção, Execução, Entrega e Apuração será eventual. A operação deve monitorar backlog, dead-letter, outbox pendente, versões materializadas e divergências entre etapas.

Atendimentos e listas proativas são tipos distintos de origem de pesquisa. Essa distinção deve existir no contrato arquitetural do evento de origem registrada, para que a Seleção aplique regras adequadas sem que a Ingestão assuma decisão de elegibilidade.

Eventos relacionados a expiração, não-resposta e encerramento por limite de respostas devem ser suficientemente distinguíveis para permitir auditoria e apuração correta. A expiração por tempo e o encerramento por atingimento de quantidade máxima de respostas não devem ser tratados como o mesmo fato operacional.

8.4 Templates e placeholders¶
Templates de mensagem devem ser versionados junto com a pesquisa publicada. A Gestão valida sintaxe e placeholders permitidos; a Execução resolve valores no contexto da instância, conforme responsabilidades definidas na seção 5.3.

A inclusão de novos placeholders deve ser governada, pois pode introduzir novos dados pessoais, novas dependências externas, novos riscos de privacidade e novas regras de fallback. A resolução de placeholders não deve registrar mensagem materializada completa em logs, eventos ou traces.

Exemplos possíveis de placeholders incluem {{cliente.nome}}, {{atendimento.unidade.nome}} e {{atendimento.funcionario.nome}}, sem que isso defina catálogo definitivo. Catálogo completo, preview, mensagens de erro e validações funcionais pertencem a documento funcional ou histórias de usuário.

Falhas de resolução devem ter comportamento previsível por política de fallback. As opções exatas de fallback por placeholder pertencem a documento funcional ou histórias de usuário.

8.5 Cache e projeções para resiliência digital¶
A solução poderá utilizar caches e projeções locais para aumentar resiliência e reduzir latência nos fluxos síncronos dos canais digitais. O uso desse recurso deve respeitar governança, rastreabilidade, privacidade e consistência das regras de negócio. A estratégia técnica de cache deve ser definida em artefato próprio.

Definições publicadas podem ser materializadas localmente pelos domínios que delas dependem. Dados contextuais obtidos de SICLI, SIISO e LDAP podem ser cacheados pela Execução quando houver autorização e proteção compatível com sua sensibilidade.

Decisões finais de elegibilidade digital só devem ser cacheadas de forma restrita, contextual e invalidável. Esse cache não substitui a Seleção como domínio de decisão. Configurações de limite de respostas podem ser materializadas por versão, mas o contador de respostas aceitas não deve depender apenas de cache local.

8.6 Quarentena, blocklist e expiração¶
Blocklist, quarentena e expiração são regras transversais com ownership distinto. A blocklist e a quarentena são aplicadas pela Seleção, conforme ADR-007. A expiração por tempo ou limite de respostas é controlada pela Execução, conforme ADR-014.

Quando houver quarentena no nível da pesquisa e do canal, a Seleção aplica o menor prazo vigente e registra a decisão. Quando houver expiração temporal ou limite de respostas, a Execução impede novas respostas após o encerramento e publica fatos suficientes para Apuração.

8.7 Observabilidade e operação¶
A observabilidade deve cobrir os fluxos críticos e permitir diagnóstico de ponta a ponta. Métricas exatas, limiares de alerta e dashboards pertencem ao runbook ou documento operacional.

8.8 Reprocessamento e governança de dados¶
Reprocessamentos devem ser autorizados, auditáveis e limitados por escopo, conforme ADR-016. Resultados históricos não devem mudar automaticamente por alteração cadastral. Consultas analíticas devem ocorrer por API, projeções autorizadas ou cargas governadas, não por acesso direto às bases internas dos serviços.

9. Decisões Arquiteturais¶
Esta seção registra as decisões arquiteturais que sustentam a solução. As decisões foram mantidas no DAS principal porque afetam fronteiras de domínio, integração, operação, segurança, evolução, custo ou governança. Detalhes de implementação, contratos completos, schemas, parâmetros operacionais e modelos físicos devem ser mantidos em documentos acessórios.

ADR-001 — Microsserviços orientados aos domínios da aplicação¶
Status: Proposto

Decisão:
A solução será decomposta em microsserviços cuja separação acompanha os domínios da aplicação: Gestão de Pesquisas, Ingestão de Origens de Pesquisa, Seleção de Público e Elegibilidade, Execução de Pesquisas, Entrega de Convites e Apuração de Métricas e Resultados.

Contexto:
A solução precisa atender diferentes públicos, canais, métricas e origens de pesquisa. A decomposição deve preservar autonomia por domínio da aplicação, não por entidade ou tabela. A visão dos blocos está consolidada na seção 5.

Alternativas Consideradas:
Foram consideradas as alternativas de monólito modular, microsserviços por entidade e microsserviços orientados aos domínios da aplicação. A última foi escolhida por oferecer melhor equilíbrio entre autonomia, coesão e governança.

Consequências:
A decisão melhora coesão e evolução independente, mas aumenta custo operacional, contratos, observabilidade e governança distribuída.

Critérios de Revisão:
Reavaliar se a operação dos serviços se tornar desproporcional, se dois serviços exigirem deploy coordenado com frequência, ou se algum serviço não justificar ownership funcional próprio.

ADR-002 — Comunicação híbrida síncrona e assíncrona¶
Status: Proposto

Decisão:
A solução usará comunicação síncrona quando houver usuário, canal ou consumidor aguardando resposta imediata, e comunicação assíncrona por eventos quando houver propagação de fatos de negócio, processamento desacoplado, retry ou isolamento de falhas.

Contexto:
A solução possui fluxos com naturezas diferentes. A Gestão precisa responder ao usuário administrativo. O canal digital precisa saber se há pesquisa disponível ao final da jornada. A submissão de resposta precisa confirmar recebimento ao usuário. Por outro lado, publicação de pesquisa, ingestão, seleção assíncrona, entrega de convite e apuração podem ocorrer de forma desacoplada.

Alternativas Consideradas:
Foram consideradas as alternativas de tudo síncrono, tudo assíncrono e modelo híbrido. O modelo híbrido foi escolhido por preservar experiência síncrona onde necessário e usar eventos para processamento desacoplado.

Consequências:
A decisão reduz cadeias síncronas longas e preserva autonomia dos serviços. O trade-off aceito é a consistência eventual entre os domínios.

Critérios de Revisão:
Reavaliar se algum fluxo assíncrono passar a exigir resposta imediata, se a latência da elegibilidade digital afetar a experiência do canal, ou se a plataforma de mensageria não oferecer confiabilidade operacional adequada.

ADR-003 — Gestão de Pesquisas como fonte de verdade da definição¶
Status: Proposto

Decisão:
A Gestão de Pesquisas será a fonte de verdade da definição da pesquisa. Ela publicará versões imutáveis para que Seleção, Execução e Apuração materializem localmente apenas as partes necessárias ao seu domínio.

Contexto:
A pesquisa publicada contém elementos usados por vários domínios: regras de elegibilidade para Seleção, snapshot renderizável para Execução, critérios de métrica para Apuração e templates de mensagem para materialização contextual. Se todos os serviços consultarem a Gestão em runtime, a Gestão se tornará dependência síncrona crítica de fluxos operacionais.

Alternativas Consideradas:
Foram consideradas consulta direta à Gestão em runtime, replicação integral da base da Gestão e publicação versionada com materialização local. A última foi escolhida por preservar ownership e reduzir acoplamento síncrono.

Consequências:
A decisão reduz acoplamento e preserva histórico de versões. O trade-off aceito é a necessidade de controlar materialização e consistência eventual.

Critérios de Revisão:
Reavaliar se houver exigência de consistência forte imediata entre publicação e execução, se os snapshots publicados se tornarem grandes demais para evento, ou se houver necessidade de estratégia de publicação por referência a artefato versionado externo.

ADR-004 — Persistência própria por domínio¶
Status: Proposto

Decisão:
Cada serviço terá persistência lógica própria. Nenhum serviço poderá acessar diretamente tabelas, coleções ou estruturas internas de outro serviço.

Contexto:
Microsserviços só preservam autonomia se os serviços também tiverem ownership sobre seus dados. Compartilhar banco ou permitir leitura cruzada enfraquece fronteiras, dificulta evolução de modelo e cria acoplamento oculto.

Alternativas Consideradas:
Foram consideradas banco único compartilhado, banco físico separado por serviço e persistência lógica própria por domínio. A última foi escolhida por equilibrar ownership e viabilidade operacional.

Consequências:
A decisão preserva autonomia, mas exige duplicação controlada de dados, materializações locais, eventos e consistência eventual.

Critérios de Revisão:
Reavaliar se a plataforma corporativa impuser restrições de persistência, se o custo de isolamento físico for incompatível com o benefício, ou se algum domínio não justificar persistência própria.

ADR-005 — Eventos versionados como contratos públicos¶
Status: Proposto

Decisão:
Eventos que cruzam fronteiras de domínio serão tratados como contratos públicos versionados, com envelope mínimo para identificação, versionamento, rastreabilidade e idempotência.

Contexto:
Eventos entre domínios não são detalhes internos do produtor; eles orientam comportamento de consumidores. A nomenclatura e o payload final devem ser definidos no catálogo de eventos.

Alternativas Consideradas:
Foram considerados eventos sem contrato formal, contrato implícito no código e eventos versionados/documentados. A última alternativa foi escolhida por viabilizar evolução independente.

Consequências:
A decisão melhora compatibilidade, rastreabilidade e governança, mas aumenta disciplina de documentação, testes de contrato e gestão de evolução.

Critérios de Revisão:
Reavaliar se houver schema registry corporativo com padrão próprio, se os eventos se tornarem grandes demais, ou se um evento precisar ser decomposto por consumidores muito distintos.

ADR-006 — Separação entre definição, execução e apuração¶
Status: Proposto

Decisão:
A solução separará definição da pesquisa, execução da experiência e apuração de resultados. A Gestão define e publica; a Execução instancia, materializa e coleta respostas; a Apuração calcula resultados e agregados.

Contexto:
A plataforma deve suportar múltiplas métricas em uma mesma pesquisa e futuras métricas além de CSAT, NPS e NES. Acoplar cálculo à submissão de resposta dificultaria evolução e aumentaria latência percebida pelo respondente.

Alternativas Consideradas:
Foram considerados cálculo na Execução, serviço específico por métrica e Apuração central com políticas de cálculo. A última alternativa foi escolhida por desacoplar coleta e cálculo.

Consequências:
A decisão permite confirmação da resposta após persistência, sem aguardar cálculo. O trade-off é que resultados serão eventualmente consistentes.

Critérios de Revisão:
Reavaliar se surgir requisito de cálculo síncrono, se novas métricas exigirem payload incompatível com o modelo de resposta, ou se a Apuração exigir engine de regras mais flexível.

ADR-007 — Seleção como domínio de decisão de elegibilidade¶
Status: Proposto

Decisão:
A Seleção de Público e Elegibilidade será o domínio responsável por aplicar regras de vigência, público, canal, serviço, produto, blocklist, quarentena, amostragem e prioridade. Quando houver quarentena no nível da pesquisa e do canal, deverá considerar o menor prazo vigente.

Contexto:
A decisão de gerar pesquisa precisa ser consistente entre fluxos físicos, digitais e proativos. Espalhar regras nos canais, na Ingestão ou na Execução dificultaria auditoria e evolução.

Alternativas Consideradas:
Foram consideradas decisão no canal, na Ingestão, na Execução e em domínio dedicado. A Seleção dedicada foi escolhida para centralizar decisão e explicabilidade.

Consequências:
A decisão preserva consistência entre canais e concentra auditabilidade. O trade-off é manter a Seleção disponível e performática, especialmente nos fluxos digitais síncronos.

Critérios de Revisão:
Reavaliar se a Seleção se tornar gargalo, se as regras não justificarem serviço dedicado, ou se a política de quarentena gerar conflito funcional futuro.

ADR-008 — Execução como dona da instância, resposta bruta e materialização contextual¶
Status: Proposto

Decisão:
A Execução será dona da instância da pesquisa, token/link, status, snapshot renderizável, resposta bruta, não-resposta, materialização contextual e controle de expiração por tempo ou limite de respostas.

Contexto:
A experiência apresentada ao respondente precisa ser estável e vinculada a uma versão publicada. A personalização por placeholders só pode ser resolvida no contexto de uma instância específica. A Execução também precisa impedir respostas após expiração ou atingimento de limite.

Alternativas Consideradas:
Foram consideradas materialização pelo canal, resolução de placeholders pela Gestão e resolução pela Execução. A última alternativa foi escolhida por preservar separação entre definição e runtime.

Consequências:
A decisão mantém a Gestão independente das fontes corporativas de contexto e centraliza o controle operacional da instância. O trade-off é que a Execução passa a depender de fontes externas para personalização e precisa tratar falhas, privacidade e observabilidade.

Critérios de Revisão:
Reavaliar se a materialização contextual aumentar muito a latência, se houver restrições de privacidade, se a quantidade de placeholders crescer muito, ou se o limite de respostas exigir mecanismo transacional específico.

ADR-009 — Entrega como serviço operacional especializado¶
Status: Proposto

Decisão:
A Entrega de Convites será responsável por registrar convites, acionar a plataforma de comunicação e controlar tentativas, sucesso, falha e retry.

Contexto:
O envio de convites depende de provedores externos e está sujeito a falhas transitórias, rejeições, limites e comportamento específico por meio de envio.

Alternativas Consideradas:
Foram consideradas entrega pela Execução, entrega pela Seleção e serviço próprio de Entrega. A última alternativa foi escolhida para isolar falhas operacionais de envio.

Consequências:
A decisão permite retry e controle de falhas sem recriar instâncias ou reexecutar elegibilidade. O trade-off é incluir mais uma unidade implantável e mais um fluxo assíncrono a monitorar.

Critérios de Revisão:
Reavaliar se a plataforma corporativa de comunicação oferecer orquestração completa, se todos os convites forem resolvidos por canais digitais síncronos, ou se fallback multicanal passar a ser requisito formal.

ADR-010 — Inbox/Outbox nos fluxos críticos¶
Status: Proposto

Decisão:
Produtores de eventos críticos deverão usar Outbox ou mecanismo equivalente. Consumidores críticos deverão usar Inbox, deduplicação ou mecanismo equivalente para controle de processamento.

Contexto:
A solução precisa tolerar falha entre persistir dado de negócio e publicar evento, além de tolerar reentrega de mensagens sem gerar duplicidade funcional.

Alternativas Consideradas:
Foram consideradas publicação direta, dependência exclusiva do broker e Inbox/Outbox. A última alternativa foi escolhida por aumentar robustez nos fluxos críticos.

Consequências:
A decisão aumenta confiabilidade dos fluxos assíncronos e melhora rastreabilidade, mas adiciona elementos operacionais a monitorar.

Critérios de Revisão:
Reavaliar se a plataforma oferecer mecanismo transacional equivalente entre banco e broker, ou se determinado evento puder aceitar perda sem impacto de negócio.

ADR-011 — Tokens e links como credenciais sensíveis¶
Status: Proposto

Decisão:
Tokens e links de pesquisa serão opacos, expiráveis, escopados à instância e protegidos contra exposição em logs, traces, mensagens de erro e URLs persistidas indevidamente.

Contexto:
Links de pesquisa podem permitir acesso à instância sem autenticação interativa adicional, dependendo do canal. Portanto, o token funciona como credencial temporária de acesso.

Alternativas Consideradas:
Foram consideradas URL com dados legíveis, token simples sem escopo forte e token opaco/expirável/escopado. A última alternativa foi escolhida por reduzir exposição.

Consequências:
A decisão reduz risco de exposição e reforça segurança da resposta, mas aumenta responsabilidade da Execução sobre geração, validação, expiração e revogação.

Critérios de Revisão:
Reavaliar se todos os acessos passarem a ocorrer por canais autenticados, se houver padrão corporativo de deep link seguro, ou se for exigida autenticação forte antes da resposta.

ADR-012 — Templates com placeholders controlados¶
Status: Proposto

Decisão:
Mensagens associadas à pesquisa poderão usar templates com placeholders controlados. A Gestão validará sintaxe e placeholders permitidos; a Execução resolverá os valores no momento da materialização da instância.

Contexto:
Mensagens apresentadas ao cliente podem exigir personalização com dados contextuais. Esses dados não pertencem à Gestão e dependem de fontes corporativas externas.

Alternativas Consideradas:
Foram consideradas mensagens fixas, substituição manual por canal, placeholders livres e placeholders controlados. A última alternativa foi escolhida por equilibrar personalização, governança e segurança.

Consequências:
A decisão permite mensagens contextualizadas sem acoplar a Gestão a sistemas de cliente, unidade ou funcionário. O trade-off é que a Execução passa a depender de fontes corporativas durante a materialização.

Critérios de Revisão:
Reavaliar se a quantidade de placeholders crescer muito, se houver demanda por lógica condicional no template, se as integrações degradarem a experiência, ou se requisitos de privacidade restringirem certos dados.

ADR-013 — Elegibilidade síncrona para jornadas digitais¶
Status: Proposto

Decisão:
Para jornadas digitais, a Execução consultará a Seleção de forma síncrona, por API ou gRPC, antes de materializar e retornar a pesquisa ao canal.

Contexto:
Nos fluxos assíncronos, origens autorizadas podem ser recebidas e avaliadas posteriormente. Nas jornadas digitais, o usuário espera receber a pesquisa logo após concluir a interação. As regras de elegibilidade não devem ser duplicadas nos canais nem na Execução.

Alternativas Consideradas:
Foram considerados apenas fluxo assíncrono, replicação de regras na Execução, chamada direta do canal à Seleção e orquestração pela Execução. A última alternativa foi escolhida por preservar o domínio da Seleção e centralizar a experiência na Execução.

Consequências:
A decisão permite pesquisa imediata no canal digital e preserva a Seleção como dona das regras. O trade-off é o acoplamento temporal entre Execução e Seleção.

Critérios de Revisão:
Reavaliar se o volume de chamadas digitais tornar a Seleção gargalo, se a latência impactar a jornada, ou se houver estratégia segura de pré-materialização/cache sem duplicar responsabilidade.

ADR-014 — Expiração de pesquisas por tempo e por quantidade de respostas¶
Status: Proposto

Decisão:
As pesquisas poderão possuir expiração por prazo temporal, por quantidade máxima de respostas ou por ambos. A Execução deverá impedir novas respostas após o encerramento por qualquer um desses critérios.

Contexto:
Algumas pesquisas devem ficar disponíveis apenas por uma janela de tempo. Outras podem ser encerradas quando atingirem uma quantidade de respostas suficiente para o objetivo definido.

Alternativas Consideradas:
Foram consideradas expiração apenas por tempo, apenas por quantidade e por tempo e/ou quantidade. A última alternativa foi escolhida por oferecer maior flexibilidade.

Consequências:
A decisão permite maior controle da coleta. O trade-off é a necessidade de cuidado técnico com concorrência, rejeição de submissões tardias e rastreabilidade do motivo de encerramento.

Critérios de Revisão:
Reavaliar se limites de respostas exigirem precisão forte em alta concorrência, se a apuração precisar recalcular limites retroativamente, ou se houver necessidade de cotas por segmento, canal, unidade ou público.

ADR-015 — Cache resiliente para fluxos digitais síncronos¶
Status: Proposto

Decisão:
A solução poderá utilizar cache e projeções locais para aumentar resiliência e reduzir latência nos fluxos digitais síncronos. Regras publicadas, snapshots, templates e blocklists podem ser materializados por versão. Dados contextuais de SICLI, SIISO e LDAP podem ser cacheados pela Execução quando houver autorização, temporalidade compatível, minimização de dados e proteção adequada. Decisões finais de elegibilidade digital só podem ser cacheadas de forma restrita, contextual e invalidável. Contadores de limite de respostas devem usar controle consistente na Execução, não apenas cache local.

Contexto:
A jornada digital depende de chamadas síncronas entre Canal, Execução, Seleção e, quando houver placeholders, fontes corporativas de contexto. Essa cadeia pode sofrer com latência, indisponibilidade transitória e chamadas repetidas. O cache pode melhorar a experiência do canal, mas também pode gerar decisões incorretas se usado sobre regras sensíveis como blocklist, quarentena, expiração ou limite de respostas.

Alternativas Consideradas:
Foram consideradas as alternativas de não usar cache, cache amplo de decisão final e cache governado por tipo de dado. A última alternativa foi escolhida por permitir resiliência sem substituir regras autoritativas.

Consequências:
A decisão aumenta resiliência e reduz latência percebida nos canais digitais. O trade-off é a necessidade de governar temporalidade, chave de cache, invalidação, minimização de dados e observabilidade.

Critérios de Revisão:
Reavaliar se houver inconsistência causada por decisão cacheada, se dados pessoais em cache aumentarem risco de privacidade, se a frequência de invalidação tornar o cache pouco útil, ou se o limite de respostas exigir precisão forte em cenários de alta concorrência.

ADR-016 — Reprocessamento de resultados como operação controlada¶
Status: Proposto

Decisão:
Reprocessamentos de resultados deverão ser autorizados, auditáveis e limitados por escopo. Alterações cadastrais ou mudanças em política de cálculo não devem recalcular automaticamente resultados históricos.

Contexto:
Resultados de percepção podem ser usados em indicadores executivos, acompanhamento operacional e decisões de negócio. Alterações históricas sem rastreabilidade reduzem confiança.

Alternativas Consideradas:
Foram considerados recálculo automático, proibição absoluta de reprocessamento e reprocessamento controlado. A última alternativa foi escolhida por equilibrar correção e governança.

Consequências:
A decisão permite corrigir falhas e recomputar cenários específicos sem perder governança. O trade-off é a necessidade de mecanismos operacionais e controles de autorização.

Critérios de Revisão:
Reavaliar se houver exigência de congelamento absoluto de resultados, se reprocessamentos se tornarem frequentes, ou se plataforma analítica externa assumir recomputações históricas.

10. Requisitos de Qualidade¶
Esta seção organiza os requisitos de qualidade em formato compatível com ATAM. A árvore de qualidade apresenta os atributos arquiteturais relevantes e seus refinamentos. Os cenários de qualidade descrevem situações avaliáveis usando apenas estímulo, ambiente, resposta e métrica.

10.1 Árvore de Qualidade¶
Qualidade da Arquitetura
├── Modificabilidade
│   ├── Evolução de métricas
│   ├── Evolução de origens de pesquisa
│   ├── Evolução de placeholders
│   └── Baixo acoplamento entre domínios
├── Confiabilidade
│   ├── Idempotência
│   ├── Publicação confiável de eventos
│   ├── Controle de duplicidade
│   └── Encerramento consistente por limite de respostas
├── Disponibilidade
│   ├── Jornada digital síncrona
│   ├── Degradação controlada
│   ├── Timeouts explícitos
│   └── Proteção contra dependências indisponíveis
├── Segurança e Privacidade
│   ├── Minimização de dados pessoais
│   ├── Proteção de tokens e links
│   ├── Controle de acesso
│   └── Proteção contra uso indevido
├── Auditabilidade
│   ├── Rastreabilidade ponta a ponta
│   ├── Elegibilidade explicável
│   ├── Versões materializadas
│   └── Reprocessamento controlado
├── Operabilidade
│   ├── Logs estruturados
│   ├── Métricas operacionais
│   ├── Tracing distribuído
│   ├── Alertas
│   └── Reconciliação operacional
├── Escalabilidade
│   ├── APIs escaláveis
│   ├── Workers escaláveis
│   ├── Mensageria dimensionável
│   └── Persistência dimensionável
└── Proteção de Capacidade
    ├── Rate limit em canais digitais
    ├── Backpressure operacional
    └── Isolamento de fluxos críticos
10.2 Cenários de Qualidade¶
10.2.1 Cenário de Qualidade — Inclusão de Nova Métrica¶
Estímulo:
Uma área de negócio solicita uma métrica de percepção além das métricas inicialmente suportadas.

Ambiente:
Evolução funcional, com pesquisas existentes preservadas.

Resposta:
A solução deve permitir a inclusão da nova métrica por configuração governada e política de cálculo específica, sem remodelar os fluxos estruturais de origem, elegibilidade, convite ou submissão de resposta.

Métrica:
A alteração não deve exigir mudança estrutural em Ingestão, Seleção ou Entrega, salvo quando houver novo tipo de pergunta ou nova semântica de resposta.

10.2.2 Cenário de Qualidade — Inclusão de Nova Origem de Pesquisa¶
Estímulo:
Uma nova origem corporativa passa a indicar potenciais pesquisas.

Ambiente:
Evolução da integração, com regras de elegibilidade já existentes.

Resposta:
A origem deve ser recebida e normalizada pela Ingestão e submetida à Seleção, sem criar caminho paralelo direto para Execução.

Métrica:
Toda nova origem deve gerar fato de origem registrada e passar pela decisão de elegibilidade antes de gerar oportunidade ou instância.

10.2.3 Cenário de Qualidade — Reentrega de Evento¶
Estímulo:
Um evento já processado é entregue novamente ao consumidor.

Ambiente:
PRD, após falha transitória, timeout, reinício de consumidor ou reprocessamento.

Resposta:
O consumidor deve reconhecer a duplicidade e concluir o processamento sem novo efeito funcional.

Métrica:
A reentrega não deve gerar duplicidade de oportunidade, instância, convite, resposta ou resultado para a mesma chave de negócio.

10.2.4 Cenário de Qualidade — Falha de Publicação de Evento¶
Estímulo:
Um dado de negócio é persistido, mas a publicação do evento correspondente falha.

Ambiente:
PRD, com indisponibilidade parcial do broker ou falha transitória de rede.

Resposta:
O evento deve permanecer recuperável por Outbox ou mecanismo equivalente.

Métrica:
Eventos pendentes devem ser observáveis e republicáveis sem intervenção manual direta na base de negócio.

10.2.5 Cenário de Qualidade — Jornada Digital Síncrona¶
Estímulo:
Um canal digital solicita pesquisa disponível ao final de uma jornada.

Ambiente:
PRD, operação normal, com usuário aguardando resposta no canal.

Resposta:
A Execução deve obter decisão da Seleção e retornar pesquisa materializada ou ausência de pesquisa dentro do SLO definido.

Métrica:
Latência, taxa de timeout e taxa de degradação devem ser mensuráveis por canal e jornada.

10.2.6 Cenário de Qualidade — Falha de Dependência Contextual¶
Estímulo:
Uma fonte corporativa de contexto não responde ou retorna indisponibilidade.

Ambiente:
PRD, durante materialização de mensagem com placeholders.

Resposta:
A Execução deve aplicar política de fallback arquitetural e preservar a experiência de forma controlada.

Métrica:
Falhas devem ser rastreáveis por fonte, sem registrar dados pessoais ou mensagem final completa em logs.

10.2.7 Cenário de Qualidade — Minimização de Dados Pessoais¶
Estímulo:
Um dado pessoal é necessário para elegibilidade, materialização, auditoria ou operação.

Ambiente:
PRD, em tráfego normal, contingência ou reprocessamento.

Resposta:
A solução deve usar apenas os dados necessários, preferindo referências técnicas ou pseudonimizadas quando possível.

Métrica:
Logs, eventos e traces não devem conter tokens completos, URLs sensíveis, mensagens materializadas completas ou dados pessoais sem necessidade arquitetural explícita.

10.2.8 Cenário de Qualidade — Proteção de Tokens e Links¶
Estímulo:
Um link ou token de pesquisa é gerado, transportado ou validado.

Ambiente:
PRD, durante convite, acesso à pesquisa ou submissão de resposta.

Resposta:
Tokens devem ser opacos, expiráveis, escopados e protegidos contra exposição.

Métrica:
Token completo não deve aparecer em logs, traces, mensagens de erro ou relatórios operacionais.

10.2.9 Cenário de Qualidade — Decisão de Elegibilidade Explicável¶
Estímulo:
Uma área de negócio, operação ou auditoria solicita explicação sobre uma pesquisa gerada, bloqueada ou descartada.

Ambiente:
PRD, após processamento síncrono ou assíncrono.

Resposta:
A Seleção deve permitir recuperar a decisão, a versão das regras e o motivo aplicável.

Métrica:
A investigação deve ser possível sem acesso direto às bases internas de outros domínios.

10.2.10 Cenário de Qualidade — Encerramento por Limite de Respostas¶
Estímulo:
A pesquisa atinge o limite de respostas enquanto novas submissões chegam.

Ambiente:
PRD, sob concorrência.

Resposta:
A Execução deve impedir respostas válidas após o encerramento por limite.

Métrica:
O mecanismo de controle deve atender à precisão definida para o limite de respostas no plano de capacidade ou ADR técnico.

10.2.11 Cenário de Qualidade — Observabilidade Ponta a Ponta¶
Estímulo:
Uma falha ou divergência é percebida em algum ponto da jornada de pesquisa.

Ambiente:
PRD, com fluxos distribuídos e consistência eventual.

Resposta:
A solução deve permitir rastrear a cadeia por correlação técnica e por identificadores de negócio.

Métrica:
Deve ser possível identificar a etapa da falha sem consultar manualmente bases internas de múltiplos domínios.

10.2.12 Cenário de Qualidade — Reprocessamento Controlado¶
Estímulo:
Há solicitação de recomputação de resultados em escopo definido.

Ambiente:
PRD ou ambiente controlado, com autorização apropriada.

Resposta:
O reprocessamento deve ser autorizado, auditável e limitado por escopo.

Métrica:
Deve haver rastreabilidade de solicitante, motivo, período, escopo e efeito nos resultados.

10.3 Lacunas quantitativas¶
Antes da entrada em PRD, devem ser definidos os parâmetros quantitativos que afetam decisões arquiteturais, como SLOs, volumetria, retenção, throughput, concorrência, precisão esperada para limite de respostas e padrão corporativo de pseudonimização. Valores numéricos específicos devem ser mantidos em documento técnico, plano de capacidade ou runbook operacional.

11. Riscos e Débitos Técnicos¶
Esta seção registra riscos e débitos relevantes para governança arquitetural. O acompanhamento operacional desses itens deve ocorrer no backlog técnico, no plano de implantação ou no runbook.

Risco ou débito	Impacto	Mitigação	Critério de acompanhamento
Fragmentação excessiva dos microsserviços	Aumento de custo operacional e dependências distribuídas sem ganho proporcional.	Manter decomposição por domínio da aplicação e exigir ADR para novos serviços.	Revisar quando houver proposta de novo serviço ou dependência operacional recorrente entre serviços existentes.
Eventos sem governança de contrato	Quebra de consumidores e necessidade de deploy coordenado.	Tratar eventos como contratos versionados e manter catálogo de eventos acessório.	Acompanhar mudanças incompatíveis, consumidores impactados e aderência ao catálogo de eventos.
Reentrega de eventos sem idempotência	Oportunidades, instâncias, convites, respostas ou resultados duplicados.	Usar Inbox, Outbox e deduplicação por chave de negócio nos fluxos críticos.	Monitorar duplicidades, eventos reprocessados e falhas de deduplicação.
Exposição de dados pessoais	Risco de privacidade, LGPD e auditoria.	Minimizar dados em eventos, logs, traces, cache e mensagens de erro.	Revisar payloads, logs e traces durante desenho técnico, homologação e auditorias.
Observabilidade insuficiente	Dificuldade de diagnosticar fluxos distribuídos e decisões de elegibilidade.	Definir correlação, logs, métricas, traces e reconciliação por etapa crítica.	Acompanhar cobertura de telemetria e capacidade de rastrear incidentes ponta a ponta.
Materialização divergente de versões publicadas	Seleção, Execução ou Apuração usando versões inconsistentes.	Monitorar versão materializada por consumidor e prever reprocessamento controlado.	Monitorar divergência de versão e atrasos de materialização.
Persistência física da Execução indefinida	Baixa performance, custo elevado ou perda de rastreabilidade.	Fechar ADR técnico com volumetria, consultas, concorrência e retenção.	Resolver antes da implementação detalhada da Execução.
Seleção como gargalo em jornada digital	Atraso ou indisponibilidade da pesquisa no canal.	Definir SLO, timeout, autoscaling, cache restrito e comportamento degradado.	Acompanhar latência, timeout, saturação e taxa de degradação por canal.
Lista proativa bypassar elegibilidade	Envio indevido ou fora da governança.	Tratar lista como origem de pesquisa e sempre submeter à Seleção.	Validar que todo fluxo de lista proativa gera origem registrada e passa pela decisão de elegibilidade.
Blocklist desatualizada ou não aplicada em todos os fluxos	Envio ou apresentação indevida para respondente bloqueado.	Materializar blocklist na Seleção e monitorar versão aplicada.	Acompanhar versão materializada e ocorrências de bloqueio por fluxo.
Quarentena aplicada de forma inconsistente entre pesquisa e canal	Bloqueio indevido ou envio excessivo de pesquisas.	Aplicar menor prazo vigente e registrar regra considerada na decisão.	Auditar decisões de elegibilidade com sobreposição de quarentena.
Limite de respostas sem controle consistente	Aceite de respostas acima do máximo configurado.	Controlar encerramento na Execução com mecanismo adequado à concorrência esperada.	Acompanhar respostas recusadas após limite e eventuais estouros em teste de concorrência.
Cache de decisão digital desatualizado	Apresentação indevida após mudança de regra, blocklist, expiração ou limite.	Usar cache apenas de forma restrita, contextual, observável e invalidável.	Monitorar uso de decisão cacheada, taxa de invalidação e incidentes associados.
Cache de dados contextuais com excesso de dados pessoais	Aumento do risco de privacidade.	Aplicar minimização, proteção do armazenamento, temporalidade compatível e não registrar valores em logs.	Revisar política de cache, dados armazenados e evidências de descarte.
Rate limit insuficiente ou mal posicionado	Saturação da Execução ou Seleção em chamadas digitais.	Aplicar controle preferencialmente no APIM/gateway e monitorar consumo por canal.	Acompanhar volume de chamadas, respostas limitadas e saturação dos serviços internos.
Crescimento desgovernado de placeholders	Aumento de dependências externas e complexidade da Execução.	Manter catálogo governado e exigir avaliação arquitetural para novos placeholders.	Revisar novos placeholders quanto a fonte, privacidade, fallback e impacto de latência.
Acesso analítico direto às bases internas	Acoplamento, vazamento semântico e risco de privacidade.	Expor resultados por API, projeções autorizadas ou cargas governadas.	Acompanhar solicitações de acesso direto e direcioná-las para interfaces governadas.
Fallback multicanal limitado	Entrega pode não atender cenários futuros de preferência ou redundância de canal.	Manter desenho simples inicialmente e evoluir apenas mediante requisito formal.	Revisar quando houver requisito de múltiplos canais, preferência de contato ou custo por canal.
Eventos negativos não obrigatórios	Pode haver lacuna para BI, auditoria ou análise de funil completo.	Não criar contratos sem consumidor real; formalizar quando houver demanda.	Revisar se operação, BI ou auditoria exigirem visibilidade completa dos descartes.
Política de fallback de placeholders pendente	Comportamento inconsistente diante de falhas em SICLI, SIISO ou LDAP.	Definir política funcional e arquitetural antes da entrada em PRD dos templates personalizados.	Acompanhar cenários de falha por fonte e decisão de produto sobre mensagem degradada.
Estratégia técnica de cache pendente	Risco de cache inseguro, ineficaz ou inconsistente.	Formalizar desenho técnico ou ADR complementar antes da entrada em PRD do fluxo digital.	Acompanhar definição de dados cacheáveis, invalidação, temporalidade e proteção.
Política de rate limit pendente	Risco de exposição produtiva sem proteção adequada de capacidade.	Definir política técnica no APIM/gateway antes da exposição produtiva.	Acompanhar limites por canal, critérios de aplicação e comportamento observado em produção.
12. Glossário¶
Termo	Definição
Pesquisa	Experiência de coleta de percepção apresentada ao respondente. Não é sinônimo de métrica.
Questionário	Estrutura versionada de blocos, perguntas, opções e regras condicionais.
Métrica	Indicador de percepção, como CSAT, NPS ou NES.
Campanha	Agrupamento ou estratégia de aplicação de pesquisas por período, público ou objetivo.
Respondente	Pessoa que pode responder uma pesquisa, como cliente ou funcionário.
Atendimento	Fato originador que pode motivar uma pesquisa.
Origem de pesquisa	Entrada normalizada pela Ingestão que pode representar atendimento, lista proativa ou outra origem autorizada para avaliação pela Seleção.
Lista proativa	Lista de respondentes ou públicos indicados para geração de pesquisa, sempre submetida à Seleção antes de gerar oportunidade ou instância.
Elegibilidade	Decisão que determina se uma origem de pesquisa ou jornada digital deve gerar oportunidade, instância ou apresentação de pesquisa.
Oportunidade de pesquisa	Decisão positiva da Seleção para criação de pesquisa.
Instância de pesquisa	Ocorrência concreta de uma pesquisa para um respondente.
Resposta bruta	Registro original persistido pela Execução.
Resultado	Valor calculado pela Apuração a partir de resposta e métrica.
Não-resposta	Expiração, recusa, abandono, cancelamento ou encerramento sem resposta válida.
Quarentena	Intervalo mínimo para evitar nova pesquisa ao mesmo respondente. Pode existir no nível da pesquisa e do canal; quando houver sobreposição, aplica-se o menor prazo vigente.
Expiração temporal	Critério de encerramento da pesquisa ou instância por prazo de disponibilidade.
Limite de respostas	Quantidade máxima de respostas válidas permitida para uma pesquisa, após a qual novas submissões não devem ser aceitas como válidas.
Template de mensagem	Texto cadastrado na Gestão com placeholders controlados.
Placeholder	Marcador declarativo substituído na materialização, como {{cliente.nome}}.
Materialização contextual	Resolução de snapshot, template e placeholders para uma instância específica.
Elegibilidade síncrona digital	Avaliação feita em tempo de interação entre Execução e Seleção.
SICLI	Sistema corporativo de cadastro de clientes, usado como fonte autorizada para dados cadastrais necessários à materialização contextual.
SIISO	Fonte corporativa para dados de unidade de atendimento.
LDAP	Diretório corporativo usado como fonte de dados de funcionário.
Blocklist	Lista governada de respondentes bloqueados para envio ou apresentação de pesquisas, aplicada prioritariamente pela Seleção.
Rate limit	Controle de limitação de chamadas, preferencialmente aplicado no APIM ou gateway, para proteger fluxos digitais contra excesso de tráfego.
Cache resiliente	Uso controlado de cache ou projeção local para reduzir latência e tolerar falhas transitórias sem substituir regras autoritativas de elegibilidade, blocklist, quarentena ou limite de respostas.
Outbox	Padrão para persistir eventos pendentes de publicação.
Inbox	Padrão para controlar eventos recebidos e evitar duplicidade.
Consistência eventual	Modelo em que serviços convergem por eventos, sem transação distribuída.
correlationId	Identificador usado para rastrear a cadeia entre APIs, eventos e workers.
Token opaco	Credencial temporária não interpretável pelo cliente ou canal.