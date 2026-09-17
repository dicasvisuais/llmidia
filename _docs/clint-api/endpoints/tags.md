# Tags — Clint API

Gestão de etiquetas (tags) do CRM: rótulos coloridos que se aplicam a contactos e negócios para segmentação e organização.

> Fonte: OpenAPI oficial da Clint (`@clint-api/v1.0`) · Base `https://api.clint.digital` · Auth: header `api-token` · Documento interno LL Mídia.

## Visão geral

As **Tags** são etiquetas do CRM (recurso `/v1`) compostas por um nome e uma cor. Servem para classificar e segmentar registos (contactos, negócios, etc.), permitindo filtrar e organizar a base de forma visual. Cada tag tem um `id` (UUID), um `name` livre, uma `color` escolhida de uma paleta fechada e um `created_at` (data de criação em ISO-8601).

Esta categoria expõe um CRUD parcial sobre `/v1/tags`: **listar** (com paginação e filtro por nome), **criar**, **obter uma** por ID e **remover** por ID. Não há operação de atualização (`PUT`/`PATCH`) na fatia — para "renomear" ou "recorar" na prática, a via disponível é criar uma nova tag e remover a antiga.

A cor (`color`) é obrigatória na criação e **tem de pertencer à paleta fixa** definida no schema `TagColor` (16 valores hexadecimais). Qualquer valor fora dessa lista deve ser rejeitado na validação (`422`). O valor por defeito da paleta é `#f44336`.

O `tag_description` oficial desta categoria é apenas *"Operations related to managing tags"* — não traz guia passo-a-passo (SMS/VOICE/Webhooks), pelo que não há fluxo adicional a reproduzir aqui.

Autenticação: todas as chamadas exigem o header `api-token`. Erros comuns transversais: `401` (sem/`api-token` inválido), `403` (feature flag/escopo em falta), `404` (tag não encontrada) e `422` (validação — ex.: cor fora da paleta ou `name`/`color` em falta na criação).

## Índice de endpoints

| Método | Caminho | O que faz |
|---|---|---|
| GET | `/v1/tags` | Lista tags de forma paginada, com filtro opcional por nome. |
| POST | `/v1/tags` | Cria uma nova tag (nome + cor). |
| GET | `/v1/tags/{id}` | Obtém uma tag pelo seu ID. |
| DELETE | `/v1/tags/{id}` | Remove uma tag pelo seu ID. |

---

## `GET` `/v1/tags` — Listar tags

**O que faz.** Devolve uma lista paginada de tags. Suporta paginação por `limit`/`offset`/`page` e filtro opcional por `name`.

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.** Nenhum.

**Parâmetros de query.**

| Nome | Tipo | Obrigatório | Default | Descrição |
|---|---|---|---|---|
| `limit` | integer | Não | `200` | Número máximo de linhas devolvidas (mín. `1`, máx. `1000`). |
| `offset` | integer | Não | `0` | Número de linhas ignoradas no resultado (mín. `0`). |
| `page` | integer | Não | `1` | Seleciona a página do resultado (mín. `1`). |
| `name` | string | Não | — | Filtra por nome da tag (ex.: `Tag name`). |

**Corpo da requisição.** Nenhum.

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `200` | Lista de tags | `Paginated` + `data: array<Tag>` |

O envelope `Paginated` inclui os campos:

| Nome | Tipo | Descrição |
|---|---|---|
| `status` | integer | Estado da resposta (ex.: `200`). |
| `totalCount` | integer | Total de itens de acordo com os filtros atuais. |
| `page` | integer | Página atual. |
| `totalPages` | integer | Total de páginas de acordo com os filtros atuais. |
| `hasNext` | boolean | Indica se existe página seguinte. |
| `hasPrevious` | boolean | Indica se existe página anterior. |
| `data` | array\<`Tag`\> | Lista de tags (ver schema `Tag`). |

**Exemplo — requisição**
```bash
curl -X GET "https://api.clint.digital/v1/tags?limit=200&offset=0&page=1&name=Tag%20name" \
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
      "name": "Tag name",
      "color": "#f44336",
      "created_at": "2026-01-15T10:30:00.000Z"
    }
  ]
}
```

**Notas (inteligência LL Mídia).** A paginação segue o padrão `Paginated` do `/v1`: por defeito devolve `limit=200`, com o teto rígido de `1000` por página — para bases grandes de tags, itere sobre `page` ou avance `offset`. O filtro `name` é o único filtro disponível; use-o para verificar a existência de uma tag antes de a criar (evita duplicados, já que não há restrição de unicidade documentada na spec). Combine `hasNext` com incrementos de `page` para percorrer o conjunto completo sem depender de `totalCount`.

---

## `POST` `/v1/tags` — Criar tag

**O que faz.** Cria uma nova tag com nome e cor.

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.** Nenhum.

**Parâmetros de query.** Nenhum.

**Corpo da requisição.** Schema `TagCreateSchema` (`application/json`, obrigatório).

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `name` | string | Sim | Nome da tag (ex.: `Tag name`). |
| `color` | string (`TagColor`) | Sim | Cor da tag; tem de ser um dos valores da paleta `TagColor`. Default da paleta: `#f44336`. |

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `200` | OK | Resposta genérica `OK` (sem schema de corpo definido na fatia). |

**Exemplo — requisição**
```bash
curl -X POST "https://api.clint.digital/v1/tags" \
  -H "api-token: SUA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Tag name",
    "color": "#f44336"
  }'
```

**Exemplo — resposta (`200`)**
```json
{
  "status": 200
}
```

> Nota: a fatia define esta resposta apenas como `OK` (componente `responses/200`), sem schema de corpo. O JSON acima é ilustrativo do envelope de estado; não assuma um corpo mais rico do que o que a spec garante.

**Notas (inteligência LL Mídia).** Ambos os campos são obrigatórios — omitir `name` ou `color` deve devolver `422`. A `color` é validada contra a paleta fechada `TagColor` (16 valores); não envie hex arbitrário. A spec não documenta unicidade de `name`, por isso é possível criar tags com nomes repetidos; se quiser evitar duplicados, faça primeiro um `GET /v1/tags?name=...`. Para "editar" uma tag (não há `PUT`/`PATCH`), a alternativa prática é criar a nova e remover a antiga com `DELETE /v1/tags/{id}`.

---

## `GET` `/v1/tags/{id}` — Obter tag

**O que faz.** Devolve uma única tag pelo seu identificador.

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.**

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string (uuid) | Sim | UUID da tag (ex.: `8feade82-d77b-4e8b-9d35-fd43e972b5c8`). |

**Parâmetros de query.** Nenhum.

**Corpo da requisição.** Nenhum.

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `200` | Objeto de tag | `{ status: integer, data: Tag }` |

Campos do corpo:

| Nome | Tipo | Descrição |
|---|---|---|
| `status` | integer | Estado da resposta (ex.: `200`). |
| `data` | `Tag` | O objeto da tag (ver schema `Tag`). |

**Exemplo — requisição**
```bash
curl -X GET "https://api.clint.digital/v1/tags/8feade82-d77b-4e8b-9d35-fd43e972b5c8" \
  -H "api-token: SUA_API_KEY"
```

**Exemplo — resposta (`200`)**
```json
{
  "status": 200,
  "data": {
    "id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8",
    "name": "Tag name",
    "color": "#f44336",
    "created_at": "2026-01-15T10:30:00.000Z"
  }
}
```

**Notas (inteligência LL Mídia).** Diferentemente da listagem (que devolve um envelope `Paginated`), aqui o corpo é `{ status, data }` com um único objeto `Tag`. Um `id` inexistente devolve `404`. Guarde o `id` retornado ao criar/listar tags para poder obter ou remover mais tarde.

---

## `DELETE` `/v1/tags/{id}` — Remover tag

**O que faz.** Remove uma única tag pelo seu identificador.

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.**

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string (uuid) | Sim | UUID da tag (ex.: `8feade82-d77b-4e8b-9d35-fd43e972b5c8`). |

**Parâmetros de query.** Nenhum.

**Corpo da requisição.** Nenhum.

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `204` | No Content | Sem corpo. |

**Exemplo — requisição**
```bash
curl -X DELETE "https://api.clint.digital/v1/tags/8feade82-d77b-4e8b-9d35-fd43e972b5c8" \
  -H "api-token: SUA_API_KEY"
```

**Exemplo — resposta (`204`)**
```
HTTP/1.1 204 No Content
```

**Notas (inteligência LL Mídia).** A remoção bem-sucedida devolve `204` sem corpo — não espere JSON de confirmação. Um `id` inexistente tende a devolver `404`. A operação é destrutiva e não há endpoint de atualização; confirme o `id` (via `GET /v1/tags/{id}`) antes de remover, sobretudo se a tag estiver aplicada a contactos/negócios.

---

## Objetos relacionados

- [`Tag`](../03-modelo-de-dados.md#tag) — a etiqueta em si: `id` (UUID), `name`, `color` e `created_at`.
- [`TagCreateSchema`](../03-modelo-de-dados.md#tagcreateschema) — corpo de criação: `name` e `color` (ambos obrigatórios).
- [`TagColor`](../03-modelo-de-dados.md#tagcolor) — enum de cores (16 valores hex; default `#f44336`): `#f44336`, `#e91e63`, `#9c27b0`, `#673ab7`, `#3f51b5`, `#2196f3`, `#03a9f4`, `#00bcd4`, `#009688`, `#4caf50`, `#8bc34a`, `#faa200`, `#ff9800`, `#ff5722`, `#795548`, `#607d8b`.
- [`ID`](../03-modelo-de-dados.md#id) — string UUID usada como identificador de tag.
- [`Paginated`](../03-modelo-de-dados.md#paginated) — envelope de paginação `/v1` usado na listagem (`status`, `totalCount`, `page`, `totalPages`, `hasNext`, `hasPrevious`).
