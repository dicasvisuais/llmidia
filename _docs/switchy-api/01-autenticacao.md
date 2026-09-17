# Autenticação — Switchy API

> Fonte: <https://developers.switchy.io/docs/overview/authentication> · Documento interno LL Mídia.

## Resumo

Autenticação por **bearer token em header próprio**. Não há OAuth, não há `Authorization: Bearer`, não há tokens de sessão. É uma string única enviada em **todas** as requisições — tanto no GraphQL como no REST.

```
Api-Authorization: SEU_TOKEN
```

> ⚠️ Atenção ao nome do header: é `Api-Authorization`, **não** `Authorization`. Um erro comum é usar `Authorization: Bearer <token>`, que a Switchy ignora.

## Escopo do token: por workspace

Citação da doc oficial: cada token é dedicado a um workspace, pelo que com um token só se acede aos dados do workspace correspondente.

Implicação prática (**inteligência LL Mídia**): se a LL Mídia usar mais do que um workspace no Switchy (por exemplo, um por marca ou por cliente), é preciso guardar **um token por workspace** e escolher o token certo antes de cada chamada. Não existe forma de trocar de workspace dentro de uma chamada.

## Como gerar o token

1. Login em [switchy.io](https://switchy.io/)
2. Ir para o **workspace** que se quer usar com a API
3. Abrir a página de **settings** e escolher o separador **integrations**
4. Clicar no botão **Generate a token**

## Exemplo oficial (GraphQL)

```bash
curl -H "Api-Authorization: your-token" -X POST -d " \
 { \
   \"query\": \"query {  workspaces {     companyName     createdDate     id     name   }}\" \
 } \
" https://graphql.switchy.io/v1/graphql
```

Versão mais legível, com `Content-Type` explícito (**inteligência LL Mídia** — o exemplo oficial omite-o):

```bash
curl -X POST 'https://graphql.switchy.io/v1/graphql' \
  -H 'Content-Type: application/json' \
  -H 'Api-Authorization: SEU_TOKEN' \
  -d '{"query":"query { workspaces { companyName createdDate id name } }"}'
```

## O token também é o que define o schema que se vê

Verificação nossa (**inteligência LL Mídia**, 2026-09-17): o endpoint GraphQL aceita introspecção **sem** token e responde `200`, mas o `query_root` vem com **zero campos**. O Hasura filtra o schema por *role*, e o role anónimo não tem acesso a nenhuma tabela.

Consequência: **não é possível mapear o modelo de dados do Switchy sem um token válido.** Qualquer trabalho sério de integração começa por gerar o token e correr a introspecção autenticada.

## Integrações oficiais / parceiro

Para integrações oficiais existe um segundo nível de acesso, com mais permissões. Nesse caso enviam-se **dois** headers em cada requisição:

```
Partner-Authorization: TOKEN_DE_PARCEIRO
Api-Authorization: TOKEN_DO_UTILIZADOR
```

O `Partner-Authorization` identifica o parceiro; o `Api-Authorization` continua a identificar o utilizador/workspace final. O token de parceiro obtém-se pelo live chat da Switchy (no site ou dentro da app).

O que o estatuto de parceiro desbloqueia, segundo a doc:

- Acesso alargado à API ("you have more access to our API")
- Uso dos domínios `hi.switchy.io` e `swiy.io` na criação de links — ver [`04-limites-e-regras.md`](./04-limites-e-regras.md)
- Integração em white-label dentro de um SaaS próprio

Sem parceria, a API é explicitamente **só para uso pessoal na própria conta**. Detalhes e a advertência formal em [`04-limites-e-regras.md`](./04-limites-e-regras.md).

## Boas práticas (inteligência LL Mídia)

- Guardar o token em variável de ambiente (`SWITCHY_TOKEN`), nunca em código versionado.
- Com múltiplos workspaces, nomear as variáveis pelo workspace (`SWITCHY_TOKEN_LLMIDIA`, etc.).
- O token não tem prazo de validade documentado nem endpoint de revogação pela API — a regeneração faz-se na UI, no mesmo sítio onde foi gerado.
- A doc não documenta os códigos de erro de autenticação. `(inferência)` Espera-se `401`/`403` num token inválido, mas isto não está confirmado pela fonte.
