# GraphQL — endpoint, introspecção e sintaxe

> Fontes: <https://developers.switchy.io/docs/overview/root-endpoint>, <https://developers.switchy.io/docs/overview/schema-introspection>, <https://developers.switchy.io/docs/guides/how-to-query> · Documento interno LL Mídia.

## Endpoint raiz

```
POST https://graphql.switchy.io/v1/graphql
```

Regras da doc oficial:

- O verbo é sempre **POST**, tanto para queries como (teoricamente) mutations — em GraphQL não são os verbos HTTP que determinam a operação, é o corpo JSON.
- **Excepção:** uma query de introspecção pode ser feita com um simples **GET** ao endpoint.
- O corpo é JSON e tem de conter uma string chamada `query`.
- ⚠️ A string de `query` tem de escapar os newlines, senão o schema não é interpretado correctamente. No corpo POST, usar aspas duplas exteriores e aspas duplas internas escapadas.

A Switchy recomenda um cliente GraphQL como o [Altair](https://altair.sirmuel.design/) em vez de cURL, precisamente por causa do JSON multi-linha.

## Por que é que isto importa: é Hasura

**Inteligência LL Mídia.** Verificámos por introspecção que o `queryType` se chama `query_root` — a assinatura inconfundível do **Hasura**. A doc oficial nunca o diz, mas o exemplo oficial de query confirma-o:

```graphql
domains(where: {removedDate: {_is_null: true}}) { ... }
```

`_is_null` é um operador Hasura. Isto significa que, apesar de a doc ser mínima, **conhecemos toda a gramática de consulta**, porque o Hasura gera sempre os mesmos argumentos por tabela:

| Argumento | Para quê |
|---|---|
| `where` | Filtro booleano (`_eq`, `_neq`, `_gt`, `_gte`, `_lt`, `_lte`, `_in`, `_nin`, `_like`, `_ilike`, `_is_null`, `_and`, `_or`, `_not`) |
| `order_by` | Ordenação: `{campo: asc}` / `{campo: desc}` / `asc_nulls_last` etc. |
| `limit` | Número máximo de linhas |
| `offset` | Deslocamento para paginação |
| `distinct_on` | Desduplicação por coluna |

E, por convenção Hasura, para uma tabela `X` existem tipicamente:

- `X` — lista de linhas
- `X_by_pk(...)` — uma linha por chave primária
- `X_aggregate` — contagens e agregações (`aggregate { count }`)

> `(inferência)` A existência concreta de `_by_pk` e `_aggregate` em cada tabela do Switchy depende das permissões do role do token. **Confirmar sempre com a introspecção autenticada** antes de assumir.

## Introspecção

A doc dá três níveis de introspecção.

### 1. Listar todos os tipos

```graphql
query {
  __schema {
    types {
      name
      kind
      description
      fields { name }
    }
  }
}
```

### 2. Inspeccionar um tipo concreto

```graphql
query {
  __type(name: "links") {
    name
    kind
    description
    fields { name }
  }
}
```

> Nota: o exemplo oficial usa `"links"` em minúsculas e no plural — o que reforça que os tipos seguem os nomes das tabelas Hasura, não PascalCase.

### 3. Introspecção completa

A query completa (`IntrospectionQuery` com os fragmentos `FullType`, `InputValue` e `TypeRef`) está no upstream, em [`_upstream/overview/schema-introspection.md`](./_upstream/overview/schema-introspection.md). A Switchy recomenda pretty-print do JSON resultante para o conseguir ler.

### Receita LL Mídia: gravar o schema do workspace

```bash
curl -s 'https://graphql.switchy.io/v1/graphql' \
  -H 'Content-Type: application/json' \
  -H "Api-Authorization: $SWITCHY_TOKEN" \
  -d @introspection.json \
  | python3 -m json.tool > switchy-schema.json

# listar as tabelas visíveis pelo token
python3 -c "
import json
s = json.load(open('switchy-schema.json'))['data']['__schema']
qr = [t for t in s['types'] if t['name'] == 'query_root'][0]
print('\n'.join(f['name'] for f in qr['fields']))
"
```

⚠️ Sem token, este comando devolve `query_root` com **zero campos** — não é um erro, é o filtro de permissões do Hasura. Ver [`01-autenticacao.md`](./01-autenticacao.md).

## O que sabemos que existe

Da doc oficial, com nomes confirmados:

| Tipo | Campos confirmados na doc |
|---|---|
| `workspaces` | `id`, `name`, `companyName`, `createdDate`, `activateRgpdEverywhere` |
| `domains` | `name`, `createdDate`, `removedDate` |
| `links` | mencionado como tipo introspectável; campos não listados na doc |

Query oficial combinando dois recursos numa só chamada (exactamente o argumento de venda do GraphQL segundo a Switchy):

```graphql
query SomeDatas {
  workspaces {
    companyName
    activateRgpdEverywhere
    createdDate
    id
    name
  }
  domains(where: {removedDate: {_is_null: true}}) {
    createdDate
    name
  }
}
```

> O filtro `removedDate: {_is_null: true}` revela um padrão de **soft delete**: os domínios apagados ficam na tabela com `removedDate` preenchido. `(inferência)` É provável que outras tabelas sigam a mesma convenção — filtrar por `removedDate` nulo ao listar links é uma precaução sensata até haver confirmação.

## O que NÃO existe no GraphQL

- **Mutations.** `mutationType` é `null`. A doc é explícita: "we don't expose our GraphQL endpoint for creating link. We are working hard to allow it." O mesmo para actualização. Toda a escrita passa pelo REST — ver [`endpoints/`](./endpoints/).
- **Subscriptions.** `subscriptionType` é `null`. O changelog oficial diz apenas que "não sabemos se no futuro vamos gerir o versionamento para Subscriptions".

## Configurar o Altair (guia oficial)

1. Instalar o [Altair GraphQL Client](https://altair.sirmuel.design/)
2. URL do GraphQL: `https://graphql.switchy.io/v1/graphql`
3. Adicionar o header `Api-Authorization` com o token
4. Recarregar os docs no Altair, para obter o schema correspondente ao acesso do token
5. (Opcional, recomendado pela Switchy) Activar o modo experimental nas definições e instalar o plugin `altair-graphql-plugin-graphql-explorer`, que permite construir queries com cliques

> O passo 4 é a confirmação prática, na UI, do que descrevemos acima: **o schema visível depende do token**.
