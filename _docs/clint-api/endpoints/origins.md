# Origins — Clint API

Recurso de leitura para consultar as **origens** (funis/pipelines) da sua conta Clint, em lista paginada ou individualmente por ID.

> Fonte: OpenAPI oficial da Clint (`@clint-api/v1.0`) · Base `https://api.clint.digital` · Auth: header `api-token` · Documento interno LL Mídia.

## Visão geral

A categoria **Origins** expõe endpoints de consulta (apenas leitura) sobre as origens existentes na conta. Uma `Origin` é o objecto que representa um funil/pipeline, identificado por `id` (UUID), com um `name`, uma referência ao `group` a que pertence (objecto com `id` e `name`) e a sua lista de `stages` (etapas do funil). Inclui ainda informação opcional de arquivamento (`archived_at` e `archived_by`) — quando a origem está arquivada, estes campos vêm preenchidos com a data/hora do arquivamento e o UUID de quem arquivou.

Cada elemento de `stages` descreve uma etapa do funil com `id` (UUID), `label` (rótulo), `order` (posição inteira na ordenação) e `type` (tipo da etapa). É por aqui que se percebe a estrutura do pipeline associado a cada origem.

Ambos os endpoints vivem na **v1** do CRM (`/v1/origins`), pelo que seguem as convenções gerais da v1: autenticação por header `api-token`, respostas em JSON e, na listagem, o envelope de paginação `Paginated` (ver [Convenções](../02-convencoes.md)). A listagem aceita ainda um filtro `group_id` para restringir os resultados a um único grupo.

Esta fatia disponibiliza **2 operações**, ambas `GET`: listar origens com paginação (e filtro opcional por grupo) e obter uma origem específica pelo seu `id`. Não há, nesta categoria, operações de criação, actualização ou remoção. O `tag_description` oficial desta categoria diz literalmente *"Operations related to managing origins"* — é um texto descritivo genérico e **não** um guia passo-a-passo do tipo SMS/VOICE/Webhooks, pelo que não há walkthrough a reproduzir aqui.

## Índice de endpoints

| Método | Caminho | O que faz |
|---|---|---|
| GET | `/v1/origins` | Lista origens, de forma paginada, com filtro opcional por grupo. |
| GET | `/v1/origins/{id}` | Obtém uma origem única pelo seu `id`. |

---

## `GET` `/v1/origins` — List origins

**O que faz.** Devolve uma lista paginada de origens da conta (`Retrieve a paginated list of origins`).

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.** Nenhum.

**Parâmetros de query.**

| Nome | Tipo | Obrigatório | Default | Descrição |
|---|---|---|---|---|
| `limit` | integer | Não | `200` | Número máximo de linhas devolvidas. Mínimo `1`, máximo `1000`. |
| `offset` | integer | Não | `0` | Número de linhas ignoradas no início do resultado. Mínimo `0`. |
| `page` | integer | Não | `1` | Selecciona a página do resultado. Mínimo `1`. |
| `group_id` | string (uuid) | Não | — | Filtra as origens por ID de grupo. |

**Cabeçalhos específicos.**

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `api-token` | string | Sim | API Token da conta. |

**Corpo da requisição.** Nenhum.

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `200` | A list of origins | `Paginated` + `data: Origin[]` |

O corpo de `200` combina (`allOf`) o envelope [`Paginated`](../03-modelo-de-dados.md#paginated) com uma propriedade `data`, que é um array de [`Origin`](../03-modelo-de-dados.md#origin). Campos do envelope:

| Campo | Tipo | Descrição |
|---|---|---|
| `status` | integer | Código de estado da resposta. |
| `totalCount` | integer | Total de itens de acordo com os filtros actuais. |
| `page` | integer | Página actual. |
| `totalPages` | integer | Total de páginas de acordo com os filtros actuais. |
| `hasNext` | boolean | Indica se existe página seguinte. |
| `hasPrevious` | boolean | Indica se existe página anterior. |
| `data` | array de `Origin` | Lista de origens desta página. |

**Exemplo — requisição**
```bash
curl -X GET "https://api.clint.digital/v1/origins?limit=200&offset=0&page=1&group_id=8feade82-d77b-4e8b-9d35-fd43e972b5c8" \
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
      "name": "Origin name",
      "group": {
        "id": "123e4567-e89b-12d3-a456-426614174000",
        "name": "Group name"
      },
      "stages": [
        {
          "id": "a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d",
          "label": "Novo lead",
          "order": 1,
          "type": "open"
        },
        {
          "id": "b2c3d4e5-f6a7-4b8c-9d0e-1f2a3b4c5d6e",
          "label": "Ganho",
          "order": 2,
          "type": "won"
        }
      ],
      "archived_at": null,
      "archived_by": null
    }
  ]
}
```

**Notas (inteligência LL Mídia).**
- Endpoint de **leitura apenas**. Não existe, nesta fatia, forma de criar, editar ou arquivar origens via API — o arquivamento reflecte-se nos campos `archived_at`/`archived_by`, mas é gerido fora destes endpoints.
- O filtro `group_id` é útil para listar apenas as origens (funis) de um grupo específico. Sem ele, são devolvidas todas as origens visíveis para o `api-token`.
- Para percorrer todas as origens, itere com `page` (ou `offset`) até `hasNext` ser `false`; use `totalCount`/`totalPages` para dimensionar o loop.
- `limit` tem tecto rígido de `1000`; valores acima disso são inválidos. Em contas com muitas origens, prefira `limit` alto e paginação por `page` para reduzir chamadas.
- `offset` e `page` são ambos aceites; combine-os com cuidado para não saltar ou repetir registos. Em geral, escolha **uma** estratégia de paginação (`page` ou `offset`) e mantenha-a.
- A lista `stages` de cada origem descreve o pipeline associado; é a partir daqui que se mapeiam as etapas de um funil para cruzar com negócios (deals).

---

## `GET` `/v1/origins/{id}` — Get origin

**O que faz.** Devolve uma única origem pelo seu identificador (`Retrieve a single origin by ID`).

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.**

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string (uuid) | Sim | UUID da origem a obter. |

**Parâmetros de query.** Nenhum.

**Cabeçalhos específicos.**

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `api-token` | string | Sim | API Token da conta. |

**Corpo da requisição.** Nenhum.

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `200` | A origin object | `{ status: integer, data: Origin }` |

O corpo de `200` combina (`allOf`) um objecto com `status` e um objecto com `data`, sendo `data` uma única [`Origin`](../03-modelo-de-dados.md#origin):

| Campo | Tipo | Descrição |
|---|---|---|
| `status` | integer | Código de estado da resposta. |
| `data` | `Origin` | O objecto da origem pedida. |

Campos do objecto [`Origin`](../03-modelo-de-dados.md#origin):

| Campo | Tipo | Descrição |
|---|---|---|
| `id` | string (uuid) | Identificador único da origem. |
| `name` | string | Nome da origem (funil). |
| `group` | object | Grupo a que a origem pertence. |
| `group.id` | string (uuid) | UUID do grupo. |
| `group.name` | string | Nome do grupo. |
| `stages` | array de object | Etapas do funil associadas à origem. |
| `stages[].id` | string (uuid) | UUID da etapa. |
| `stages[].label` | string | Rótulo da etapa. |
| `stages[].order` | integer | Posição/ordem da etapa no funil. |
| `stages[].type` | string | Tipo da etapa. |
| `archived_at` | string (date-time) | Data/hora de arquivamento (ISO-8601), quando arquivada. |
| `archived_by` | string (uuid) | UUID de quem arquivou a origem, quando aplicável. |

**Exemplo — requisição**
```bash
curl -X GET "https://api.clint.digital/v1/origins/8feade82-d77b-4e8b-9d35-fd43e972b5c8" \
  -H "api-token: SUA_API_KEY"
```

**Exemplo — resposta (`200`)**
```json
{
  "status": 200,
  "data": {
    "id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8",
    "name": "Origin name",
    "group": {
      "id": "123e4567-e89b-12d3-a456-426614174000",
      "name": "Group name"
    },
    "stages": [
      {
        "id": "a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d",
        "label": "Novo lead",
        "order": 1,
        "type": "open"
      },
      {
        "id": "b2c3d4e5-f6a7-4b8c-9d0e-1f2a3b4c5d6e",
        "label": "Ganho",
        "order": 2,
        "type": "won"
      }
    ],
    "archived_at": null,
    "archived_by": null
  }
}
```

**Notas (inteligência LL Mídia).**
- O `id` no caminho tem de ser um **UUID** válido; um valor mal formado ou inexistente resulta tipicamente em `404` (não encontrado) — ver [erros comuns](../02-convencoes.md).
- Ao contrário da listagem, esta resposta **não** usa o envelope `Paginated`: o corpo traz apenas `status` e `data` (uma única `Origin`).
- Para origens activas, espere `archived_at` e `archived_by` a `null`; quando preenchidos, indicam que a origem foi arquivada (e por quem).
- O objecto `group` embutido evita uma chamada adicional a [Groups](./groups.md) só para descobrir a que grupo pertence a origem; ainda assim, apenas `id` e `name` do grupo são devolvidos aqui.

---

## Objetos relacionados

- [`Origin`](../03-modelo-de-dados.md#origin) — origem/funil: `id`, `name`, `group` (`id`, `name`), `stages[]` (`id`, `label`, `order`, `type`), `archived_at`, `archived_by`.
- [`Paginated`](../03-modelo-de-dados.md#paginated) — envelope de paginação v1 (`status`, `totalCount`, `page`, `totalPages`, `hasNext`, `hasPrevious`).
- [`ID`](../03-modelo-de-dados.md#id) — identificador UUID (string) usado em `id`, `group.id`, `stages[].id`, `group_id` e `archived_by`.
- [`DateTime`](../03-modelo-de-dados.md#datetime) — data/hora em ISO-8601, usado em `archived_at`.
