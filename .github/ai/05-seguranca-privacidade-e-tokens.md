# Segurança, Privacidade e Tokens

## Diretrizes

- Minimizar dados pessoais em eventos, logs, traces, cache e mensagens de erro.
- Preferir respondenteRef ou identificadores pseudonimizados.
- Tratar token e link de pesquisa como credenciais sensíveis.
- Não registrar token completo nem URL sensível em telemetria.

## Execução e placeholders

A Execução pode consultar SICLI, SIISO e LDAP para materialização contextual.
Essa resolução precisa obedecer minimização, proteção e fallback controlado.

## Operação

- APIs expostas devem ter autenticação corporativa.
- Segredos devem vir de cofre corporativo ou mecanismo equivalente.
- Rate limit deve proteger a borda, preferencialmente em APIM ou gateway.
