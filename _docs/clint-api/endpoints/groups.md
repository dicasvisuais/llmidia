# Groups — Clint API

Recurso de leitura para consultar os **grupos** (agrupamentos organizacionais) da sua conta Clint, em lista paginada ou individualmente por ID.

> Fonte: OpenAPI oficial da Clint (`@clint-api/v1.0`) · Base `https://api.clint.digital` · Auth: header `api-token` · Documento interno LL Mídia.

## Visão geral

A categoria **Groups** expõe endpoints de consulta (apenas leitura) sobre os grupos existentes na conta. Um `Group` é um objecto simples, identificado por `id` (UUID), com um `name` e informação opcional de arquivamento (`archived_at` e `archived_by`). Segundo a spec, um grupo pode estar activo ou arquivado — quando arquivado, os campos `archived_at` (data/hora do arquivamento) e `archived_by` (UUID de quem arquivou) vêm preenchidos.

Ambos os endpoints vivem na **v1** do CRM (`/v1/groups`), pelo que seguem as convenções gerais da v1: autenticação por header `api-token`, respostas em JSON e, na listagem, o envelope de paginação `Paginated` (ver [Convenções](../02-convencoes.md)).

Esta fatia disponibiliza **2 operações**, ambas `GET`: listar grupos com paginação e obter um grupo específico pelo seu `id`. Não há, nesta categoria, operações de criação, actualização ou remoção.

Nota sobre a spec: o `tag_description` oficial desta categoria diz literalmente *"Operations related to managing deals"*. Trata-se de um texto genérico herdado da spec (aparentemente copiado da categoria de negócios) e **não** de um guia passo-a-passo do tipo SMS/VOICE/Webhooks — logo, não há walkthrough a reproduzir aqui. O recurso efectivamente documentado são **grupos**, conforme o schema `Group` e os paths `/v1/groups`.

## Índice de endpoints

| Método | Caminho | O que faz |
|---|---|---|
| GET | `/v1/groups` | Lista grupos, de forma paginada. |
| GET | `/v1/groups/{id}` | Obtém um grupo único pelo seu `id`. |

---

## `GET` `/v1/groups` — List groups

**O que faz.** Devolve uma lista paginada de grupos da conta (`Retrieve a paginated list of groups`).

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.** Nenhum.

**Parâmetros de query.**

| Nome | Tipo | Obrigatório | Default | Descrição |
|---|---|---|---|---|
| `limit` | integer | Não | `200` | Número máximo de linhas devolvidas. Mínimo `1`, máximo `1000`. |
| `offset` | integer | Não | `0` | Número de linhas ignoradas no início do resultado. Mínimo `0`. |
| `page` | integer | Não | `1` | Selecciona a página do resultado. Mínimo `1`. |

**Cabeçalhos específicos.**

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `api-token` | string | Sim | API Token da conta. |

**Corpo da requisição.** Nenhum.

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `200` | A list of groups | `Paginated` + `data: Group[]` |

O corpo de `200` combina (`allOf`) o envelope [`Paginated`](../03-modelo-de-dados.md#paginated) com uma propriedade `data`, que é um array de [`Group`](../03-modelo-de-dados.md#group). Campos do envelope:

| Campo | Tipo | Descrição |
|---|---|---|
| `status` | integer | Código de estado da resposta. |
| `totalCount` | integer | Total de itens de acordo com os filtros actuais. |
| `page` | integer | Página actual. |
| `totalPages` | integer | Total de páginas de acordo com os filtros actuais. |
| `hasNext` | boolean | Indica se existe página seguinte. |
| `hasPrevious` | boolean | Indica se existe página anterior. |
| `data` | array de `Group` | Lista de grupos desta página. |

**Exemplo — requisição**
```bash
curl -X GET "https://api.clint.digital/v1/groups?limit=200&offset=0&page=1" \
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
  "hasPrevious": false,
  "data": [
    {
      "id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8",
      "name": "Group name",
      "archived_at": null,
      "archived_by": null
    },
    {
      "id": "123e4567-e89b-12d3-a456-426614174000",
      "name": "Equipa Comercial",
      "archived_at": "2020-01-01T14:15:00.000000+00:00",
      "archived_by": "8feade82-d77b-4e8b-9d35-fd43e972b5c8"
    }
  ]
}
```

**Notas (inteligência LL Mídia).**
- Endpoint de **leitura apenas**. Não existe, nesta fatia, forma de criar, editar ou arquivar grupos via API — o arquivamento reflecte-se nos campos `archived_at`/`archived_by`, mas é gerido fora destes endpoints.
- Para percorrer todos os grupos, itere com `page` (ou `offset`) até `hasNext` ser `false`; use `totalCount`/`totalPages` para dimensionar o loop.
- `limit` tem tecto rígido de `1000`; valores acima disso são inválidos. Em contas com muitos grupos, prefira `limit` alto e paginação por `page` para reduzir chamadas.
- `offset` e `page` são ambos aceites; combine-os com cuidado para não saltar ou repetir registos. Em geral, escolha **uma** estratégia de paginação (`page` ou `offset`) e mantenha-a.

---

## `GET` `/v1/groups/{id}` — Get group

**O que faz.** Devolve um único grupo pelo seu identificador (`Retrieve a single group by ID`).

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.**

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string (uuid) | Sim | UUID do grupo a obter. |

**Parâmetros de query.** Nenhum.

**Cabeçalhos específicos.**

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `api-token` | string | Sim | API Token da conta. |

**Corpo da requisição.** Nenhum.

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `200` | A group object | `{ status: integer, data: Group }` |

O corpo de `200` combina (`allOf`) um objecto com `status` e um objecto com `data`, sendo `data` um único [`Group`](../03-modelo-de-dados.md#group):

| Campo | Tipo | Descrição |
|---|---|---|
| `status` | integer | Código de estado da resposta. |
| `data` | `Group` | O objecto do grupo pedido. |

Campos do objecto [`Group`](../03-modelo-de-dados.md#group):

| Campo | Tipo | Descrição |
|---|---|---|
| `id` | string (uuid) | Identificador único do grupo. |
| `name` | string | Nome do grupo. |
| `archived_at` | string (date-time) | Data/hora de arquivamento (ISO-8601), quando arquivado. |
| `archived_by` | string (uuid) | UUID de quem arquivou o grupo, quando aplicável. |

**Exemplo — requisição**
```bash
curl -X GET "https://api.clint.digital/v1/groups/8feade82-d77b-4e8b-9d35-fd43e972b5c8" \
  -H "api-token: SUA_API_KEY"
```

**Exemplo — resposta (`200`)**
```json
{
  "status": 200,
  "data": {
    "id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8",
    "name": "Group name",
    "archived_at": null,
    "archived_by": null
  }
}
```

**Notas (inteligência LL Mídia).**
- O `id` no caminho tem de ser um **UUID** válido; um valor mal formado ou inexistente resulta tipicamente em `404` (não encontrado) — ver [erros comuns](../02-convencoes.md).
- Ao contrário da listagem, esta resposta **não** usa o envelope `Paginated`: o corpo traz apenas `status` e `data` (um único `Group`).
- Para grupos activos, espere `archived_at` e `archived_by` a `null`; quando preenchidos, indicam que o grupo foi arquivado (e por quem).

---

## Objetos relacionados

- [`Group`](../03-modelo-de-dados.md#group) — grupo da conta: `id`, `name`, `archived_at`, `archived_by`.
- [`Paginated`](../03-modelo-de-dados.md#paginated) — envelope de paginação v1 (`status`, `totalCount`, `page`, `totalPages`, `hasNext`, `hasPrevious`).
- [`ID`](../03-modelo-de-dados.md#id) — identificador UUID (string) usado em `id` e `archived_by`.
- [`DateTime`](../03-modelo-de-dados.md#datetime) — data/hora em ISO-8601, usado em `archived_at`.
