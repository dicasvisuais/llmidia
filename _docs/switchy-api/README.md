# Switchy API — Documentação interna (LL Mídia)

API do **Switchy** (encurtador de links com retargeting, pixels, deep links e cloaking).

> Documento de referência interno LL Mídia, em português (pt-PT), fiel à documentação oficial da Switchy e enriquecido com a nossa inteligência prática de uso. Nomes de campos, parâmetros, valores de enum, headers e paths ficam **em inglês**, exactamente como na fonte.

---

## O ponto mais importante antes de começar

A API do Switchy é **híbrida e assimétrica**:

| | |
|---|---|
| **Leitura** | GraphQL (Hasura) em `https://graphql.switchy.io/v1/graphql` |
| **Escrita** | REST em `https://api.switchy.io/v1` — mas **só existem 2 endpoints públicos**: criar link e actualizar link |
| **Mutations GraphQL** | **Não existem.** `mutationType` é `null` no schema. |
| **Subscriptions** | **Não existem.** `subscriptionType` é `null`. |
| **Apagar link** | **Não documentado** — não há endpoint público |
| **Estatísticas/cliques** | ❌ Só um contador cumulativo `links.clicks` (Int). **Não há cliques por data em nenhuma API.** Ver [`05-schema-graphql.md`](./05-schema-graphql.md) |

Ou seja: tudo o que for **ler** faz-se em GraphQL; tudo o que for **criar/editar links** faz-se nos dois endpoints REST. Não há mais nada exposto publicamente.

---

## Proveniência

| | |
|---|---|
| **Fonte primária** | <https://developers.switchy.io/docs/overview/index> |
| **Repositório da doc** | <https://github.com/Switchy-io/api-docs> (Docusaurus, branch `master`) |
| **Cópia fiel do upstream** | [`./_upstream/`](./_upstream/) |
| **Data do clone** | 2026-09-17 (upstream `last-modified` 2026-09-10) |
| **Versão da API** | v1 |

Regra de ouro: **não inventamos** campos, parâmetros, endpoints ou respostas. Tudo o que consta destes documentos provém da doc oficial ou de uma verificação feita por nós contra o endpoint; qualquer acréscimo nosso vem marcado como **inteligência LL Mídia** ou `(inferência)`.

### Verificações que nós próprios fizemos (inteligência LL Mídia)

Em 2026-09-17 corremos uma introspecção sem token contra `https://graphql.switchy.io/v1/graphql`:

```
{"data":{"__schema":{"queryType":{"name":"query_root"},"mutationType":null}}}
```

Conclusões práticas:

1. **É Hasura.** O nome `query_root` é a assinatura do Hasura. Isso determina toda a sintaxe de filtros, ordenação e paginação — ver [`02-graphql.md`](./02-graphql.md).
2. **Não há mutations nem subscriptions** no schema exposto.
3. **A introspecção responde sem token, mas devolve zero campos.** O schema é filtrado por *role*: sem `Api-Authorization` o role anónimo não vê nenhuma tabela. Para conhecer os tipos reais é **obrigatório** ter um token de workspace.

---

## Índice

| Documento | Conteúdo |
|---|---|
| [`01-autenticacao.md`](./01-autenticacao.md) | Header `Api-Authorization`, como gerar o token, tokens de parceiro |
| [`02-graphql.md`](./02-graphql.md) | Endpoint raiz, introspecção, sintaxe Hasura (`where`, `order_by`, `limit`, `_is_null`), Altair |
| [`03-modelo-de-dados.md`](./03-modelo-de-dados.md) | Tipos `Link`, `Pixel`, `LinkExpiration`, `PasswordProtect`, enum `Platform` |
| [`04-limites-e-regras.md`](./04-limites-e-regras.md) | Rate limits, domínios restritos, regra de uso pessoal vs. parceria |
| [`05-schema-graphql.md`](./05-schema-graphql.md) | **Schema real** por introspecção autenticada: as 8 tabelas e todos os campos |
| [`06-extrair-analitica-report.md`](./06-extrair-analitica-report.md) | **Como obter cliques por dia** raspando a página pública de report (a API não os dá) |
| [`endpoints/criar-link.md`](./endpoints/criar-link.md) | `POST /v1/links/create` |
| [`endpoints/atualizar-link.md`](./endpoints/atualizar-link.md) | `PUT /v1/links/:uniqId` e `PUT /v1/links/by-domain/:domain/:id` |
| [`_upstream/`](./_upstream/) | Markdown original da Switchy, sem alterações |

---

## Arranque rápido

### 1. Gerar o token

Login em [switchy.io](https://switchy.io/) → escolher o **workspace** → **Settings** → separador **Integrations** → **Generate a token**.

> O token é **por workspace**. Um token só dá acesso aos dados do workspace onde foi gerado. Com vários workspaces, são vários tokens.

### 2. Primeira leitura (GraphQL)

```bash
curl -X POST 'https://graphql.switchy.io/v1/graphql' \
  -H 'Content-Type: application/json' \
  -H 'Api-Authorization: SEU_TOKEN' \
  -d '{"query":"query { workspaces { id name companyName createdDate } }"}'
```

### 3. Primeira escrita (REST)

```bash
curl -X POST 'https://api.switchy.io/v1/links/create' \
  -H 'Content-Type: application/json' \
  -H 'Api-Authorization: SEU_TOKEN' \
  -d '{"link":{"url":"https://exemplo.com/","domain":"hi.switchy.io","tags":[]}}'
```

### 4. Descobrir o que existe no seu workspace

Como o schema depende do token, o passo obrigatório a seguir é correr a introspecção **com** o token e guardar o resultado. Ver [`02-graphql.md`](./02-graphql.md#introspecção).

---

## Notas de manutenção

- A doc oficial é um site Docusaurus alojado na Vercel; o conteúdo vive em `docs/**.md` no repo `Switchy-io/api-docs`.
- Para refrescar este clone: baixar `https://codeload.github.com/Switchy-io/api-docs/tar.gz/refs/heads/master`, comparar `docs/` com [`./_upstream/`](./_upstream/) e propagar as diferenças.
- Ficheiros presentes no upstream mas **vazios ou placeholder** (não copiados): `overview/about-graphql.md` (só front-matter), `overview/rate-limits.md` (contém apenas `## Coucou`), `guides/doc3.md` (lorem ipsum do template Docusaurus). Nenhum deles está no sidebar.
- A Switchy avisa que a API "está sujeita a alterações" e que tentam manter retrocompatibilidade entre versões "tanto quanto possível". Não há versionamento formal para lá do `v1` no path.
