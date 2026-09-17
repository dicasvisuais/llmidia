# Lost Status — Clint API

Os *lost status* (motivos de perda) representam as razões catalogadas pelas quais um negócio pode ser marcado como perdido no CRM da Clint.

> Fonte: OpenAPI oficial da Clint (`@clint-api/v1.0`) · Base `https://api.clint.digital` · Auth: header `api-token` · Documento interno LL Mídia.

## Visão geral

Esta categoria (`Lost Status`) expõe operações de leitura sobre os motivos de perda configurados na conta. Um *lost status* é uma entidade simples do CRM, composta apenas por um `id` (UUID) e um `name` (texto), que serve para classificar por que razão um negócio (deal) foi perdido — por exemplo, "Sem orçamento", "Escolheu concorrente" ou "Fora do perfil". A `tag_description` da spec resume o âmbito como "Operations related to managing lost status".

São endpoints da versão **v1** (CRM base) e, como todo o v1, seguem as convenções gerais: autenticação por header `api-token`, paginação por envelope `Paginated` e identificadores em UUID. A fatia expõe **apenas leitura** — uma listagem paginada e a obtenção de um registo individual por `id`. Não há, nesta categoria, operações de criação, atualização ou eliminação.

Na prática, estes registos são geridos na interface da Clint (definições de pipeline/CRM) e consumidos por integrações que precisam de resolver o motivo de perda de um negócio. O uso típico é: listar os *lost status* uma vez para construir um mapa `id → name` local, e depois usar esse mapa para traduzir o motivo de perda associado a cada deal.

Não é indicada qualquer feature flag ou escopo especial para esta categoria na fatia; aplicam-se os erros comuns da API (ver notas por endpoint).

## Índice de endpoints

| Método | Caminho | O que faz |
|---|---|---|
| GET | `/v1/lost-status` | Lista os motivos de perda de forma paginada. |
| GET | `/v1/lost-status/{id}` | Obtém um motivo de perda específico pelo seu `id`. |

---

## `GET` `/v1/lost-status` — List lost status

**O que faz.** Devolve uma lista paginada dos *lost status* (motivos de perda) configurados na conta.

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.** Nenhum.

**Parâmetros de query.**

| Nome | Tipo | Obrigatório | Default | Descrição |
|---|---|---|---|---|
| `limit` | integer | Não | `200` | Número máximo de linhas devolvidas. Mínimo `1`, máximo `1000`. |
| `offset` | integer | Não | `0` | Número de linhas ignoradas no resultado. Mínimo `0`. |
| `page` | integer | Não | `1` | Seleciona a página do resultado. Mínimo `1`. |

**Corpo da requisição.** Nenhum.

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `200` | Uma lista de *lost status*. | Envelope [`Paginated`](../03-modelo-de-dados.md#paginated) com `data` = array de [`LostStatus`](../03-modelo-de-dados.md#loststatus). |

Campos do envelope de resposta (`Paginated` + `data`):

| Campo | Tipo | Descrição |
|---|---|---|
| `status` | integer | Estado da resposta (ex.: `200`). |
| `totalCount` | integer | Total de itens com base nos filtros atuais. |
| `page` | integer | Página atual. |
| `totalPages` | integer | Total de páginas com base nos filtros atuais. |
| `hasNext` | boolean | Indica se existe página seguinte. |
| `hasPrevious` | boolean | Indica se existe página anterior. |
| `data` | array de `LostStatus` | Lista de motivos de perda (ver campos abaixo). |

Campos de cada item `LostStatus`:

| Campo | Tipo | Descrição |
|---|---|---|
| `id` | string (uuid) | Identificador único do motivo de perda. |
| `name` | string | Nome do motivo de perda. |

**Exemplo — requisição**
```bash
curl -X GET "https://api.clint.digital/v1/lost-status?limit=200&offset=0&page=1" \
  -H "api-token: SUA_API_KEY"
```

**Exemplo — resposta (`200`)**
```json
{
  "status": 200,
  "totalCount": 50,
  "page": 1,
  "totalPages": 10,
  "hasNext": true,
  "hasPrevious": true,
  "data": [
    {
      "id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8",
      "name": "Lost status name"
    }
  ]
}
```

**Notas (inteligência LL Mídia).**
- A paginação segue o envelope `Paginated` do v1: pode combinar `limit`/`offset` ou navegar por `page`. Para varrer tudo, itere enquanto `hasNext` for `true`.
- O `limit` está limitado a `1000`; o default `200` costuma bastar, já que o número de motivos de perda por conta é tipicamente pequeno.
- Como os *lost status* mudam raramente, faz sentido pré-carregar esta lista e mantê-la em cache local (mapa `id → name`) em vez de a pedir por cada deal.
- Erros comuns: `401` (sem/inválido `api-token`), `403` (escopo/feature em falta), `422` (validação de query, ex.: `limit` fora do intervalo `1..1000`).

---

## `GET` `/v1/lost-status/{id}` — Get lost status

**O que faz.** Devolve um único *lost status* (motivo de perda) identificado pelo seu `id`.

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.**

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string (uuid) | Sim | UUID do motivo de perda a obter. |

**Parâmetros de query.** Nenhum.

**Corpo da requisição.** Nenhum.

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `200` | Um objeto de *lost status*. | Objeto com `status` (integer) + `data` = [`LostStatus`](../03-modelo-de-dados.md#loststatus). |

Campos do envelope de resposta:

| Campo | Tipo | Descrição |
|---|---|---|
| `status` | integer | Estado da resposta (ex.: `200`). |
| `data` | `LostStatus` | O motivo de perda encontrado (ver campos abaixo). |

Campos de `LostStatus`:

| Campo | Tipo | Descrição |
|---|---|---|
| `id` | string (uuid) | Identificador único do motivo de perda. |
| `name` | string | Nome do motivo de perda. |

**Exemplo — requisição**
```bash
curl -X GET "https://api.clint.digital/v1/lost-status/8feade82-d77b-4e8b-9d35-fd43e972b5c8" \
  -H "api-token: SUA_API_KEY"
```

**Exemplo — resposta (`200`)**
```json
{
  "status": 200,
  "data": {
    "id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8",
    "name": "Lost status name"
  }
}
```

**Notas (inteligência LL Mídia).**
- Ao contrário da listagem, a resposta individual **não** usa o envelope `Paginated`: traz apenas `status` e `data`.
- Use este endpoint quando já tem o `id` (por exemplo, o motivo de perda associado a um deal) e precisa de resolver o `name` correspondente.
- Erros comuns: `401` (sem/inválido `api-token`), `404` (nenhum *lost status* com esse `id`), `422` (o `id` não é um UUID válido).

---

## Objetos relacionados

- [`LostStatus`](../03-modelo-de-dados.md#loststatus) — motivo de perda (`id` + `name`) usado para classificar negócios perdidos.
- [`Paginated`](../03-modelo-de-dados.md#paginated) — envelope de paginação padrão do v1 (`status`, `totalCount`, `page`, `totalPages`, `hasNext`, `hasPrevious`), usado na listagem.
- [`ID`](../03-modelo-de-dados.md#id) — identificador UUID reutilizado como `id` do `LostStatus`.
