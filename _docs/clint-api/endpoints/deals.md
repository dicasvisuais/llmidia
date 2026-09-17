# Deals — Clint API

Gestão de negócios (deals) do CRM: listagem paginada com filtros ricos, criação, leitura, atualização, remoção e consulta do histórico/timeline de um negócio.

> Fonte: OpenAPI oficial da Clint (`@clint-api/v1.0`) · Base `https://api.clint.digital` · Auth: header `api-token` · Documento interno LL Mídia.

## Visão geral

Os **deals** (negócios) são as oportunidades comerciais do CRM da Clint. Cada negócio pertence a uma **origem** (`origin_id`), está posicionado numa **etapa** (`stage_id`) do funil, está ligado a um **contacto** (`contact`) e a um **utilizador/responsável** (`user`), e transita entre os estados `OPEN`, `WON` e `LOST` (schema [`DealStatus`](../03-modelo-de-dados.md#dealstatus)). A par dos campos-padrão, cada negócio pode carregar dados personalizados no objeto livre `fields` (com subestruturas `contact` e `organization`).

A maior parte das operações vive em **`/v1`** (CRM base): `GET /v1/deals`, `POST /v1/deals`, `GET /v1/deals/{id}`, `POST /v1/deals/{id}` (atualização — repare que a atualização usa `POST`, não `PATCH`/`PUT`) e `DELETE /v1/deals/{id}`. Estas seguem as convenções v1 de paginação (envelope [`Paginated`](../03-modelo-de-dados.md#paginated): `limit`/`offset`/`page`) e de erros comuns (401/403/404/422). Ver [Convenções](../02-convencoes.md).

A operação de **histórico** vive em **`/v2`**: `GET /v2/deals/{id}/history`. Devolve a timeline do negócio — os mesmos itens do separador *Histórico* no painel do CRM (mudanças de etapa/estado/responsável, notas, tags, e-mails/SMS/chamadas, atividades concluídas e eventos de integração), para o negócio e o seu contacto. Cada item traz o `body` bruto do evento (formato varia por `category`) e um `summary` pré-renderizado em pt-BR (`null` nas categorias sem sumarizador). Os itens vêm ordenados por data descendente e a resposta usa o envelope `PaginatedV2`.

**Feature flag / escopo.** O endpoint de histórico exige que a API key tenha o escopo **`deals:read`**; sem ele a chamada devolve `403`. As restantes operações `/v1` exigem apenas o header `api-token` válido.

## Índice de endpoints

| Método | Caminho | O que faz |
|---|---|---|
| GET | `/v1/deals` | Lista negócios de forma paginada, com filtros por origem, datas, utilizador, contacto, tags, estado e etapa. |
| POST | `/v1/deals` | Cria um novo negócio. |
| GET | `/v1/deals/{id}` | Obtém um negócio pelo ID. |
| POST | `/v1/deals/{id}` | Atualiza um negócio existente. |
| DELETE | `/v1/deals/{id}` | Remove um negócio pelo ID. |
| GET | `/v2/deals/{id}/history` | Devolve o histórico/timeline do negócio (e do seu contacto). |

---

## `GET` `/v1/deals` — Lista negócios

**O que faz.** Devolve uma lista paginada de negócios. Aceita um conjunto alargado de filtros combináveis (por origem, intervalos de datas, utilizador, contacto, tags, estado e etapa).

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.** Nenhum.

**Parâmetros de query.**

| Nome | Tipo | Obrigatório | Default | Descrição |
|---|---|---|---|---|
| `limit` | integer (1–1000) | Não | `200` | Número máximo de linhas devolvidas. |
| `offset` | integer (>= 0) | Não | `0` | Número de linhas ignoradas no resultado. |
| `page` | integer (>= 1) | Não | `1` | Página do resultado a devolver. |
| `origin_id` | string (UUID) | Não | — | Filtra pelo ID da origem. |
| `created_at_start` | string (date-time) | Não | — | Filtra por `created_at` com operador GTE (>=). |
| `created_at_end` | string (date-time) | Não | — | Filtra por `created_at` com operador LTE (<=). |
| `updated_at_start` | string (date-time) | Não | — | Filtra por `updated_at` com operador GTE (>=). |
| `updated_at_end` | string (date-time) | Não | — | Filtra por `updated_at` com operador LTE (<=). |
| `user_id` | string (UUID) | Não | — | Filtra pelo ID do utilizador/responsável. |
| `user_email` | string | Não | — | Filtra pelo e-mail do utilizador (ex.: `user@email.com`). |
| `contact_id` | string (UUID) | Não | — | Filtra pelo ID do contacto. |
| `phone` | string | Não | — | Filtra pelo telefone do contacto (ex.: `999999999`). |
| `email` | string | Não | — | Filtra pelo e-mail do contacto (ex.: `contact@email.com`). |
| `tag_ids` | string | Não | — | Filtra por IDs de tags, com operador OR. Separados por `,`. |
| `tag_names` | string | Não | — | Filtra por nomes de tags, com operador OR. Separados por `,`. |
| `status` | string (enum) | Não | — | Filtra pelo estado do negócio: `OPEN`, `WON` ou `LOST` (ver [`DealStatus`](../03-modelo-de-dados.md#dealstatus)). |
| `stage_id` | string (UUID) | Não | — | Filtra pelo ID da etapa. |
| `updated_stage_at_start` | string (date-time) | Não | — | Filtra por `updated_stage_at` com operador GTE (>=). |
| `updated_stage_at_end` | string (date-time) | Não | — | Filtra por `updated_stage_at` com operador LTE (<=). |
| `fields` | object (deepObject) | Não | — | Filtra por campos personalizados do negócio. Pode ser usado várias vezes, um por campo (estilo `deepObject`, ex.: `fields[cidade]=Lisboa`). |
| `won_at_start` | string (date-time) | Não | — | Filtra por `won_at` com operador GTE (>=). |
| `won_at_end` | string (date-time) | Não | — | Filtra por `won_at` com operador LTE (<=). |
| `lost_at_start` | string (date-time) | Não | — | Filtra por `lost_at` com operador GTE (>=). |
| `lost_at_end` | string (date-time) | Não | — | Filtra por `lost_at` com operador LTE (<=). |

**Corpo da requisição.** Nenhum.

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `200` | Lista de negócios. | Envelope [`Paginated`](../03-modelo-de-dados.md#paginated) + `data`: array de [`Deal`](../03-modelo-de-dados.md#deal). |

**Exemplo — requisição**
```bash
curl -X GET "https://api.clint.digital/v1/deals?status=OPEN&limit=2&page=1" \
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
      "origin_id": "99999999-d77b-4e8b-9d35-fd43e972b999",
      "user": {
        "id": "11111111-2222-3333-4444-555555555555",
        "full_name": "User full name"
      },
      "contact": {
        "id": "66666666-7777-8888-9999-000000000000",
        "name": "Contact name",
        "email": "contact@email.com",
        "phone": "+5548999999999"
      },
      "created_at": "2020-01-01T14:15:00.000000+00:00",
      "stage_id": "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee",
      "updated_stage_at": "2020-01-02T09:30:00.000000+00:00",
      "status": "OPEN",
      "won_at": null,
      "won_by": null,
      "lost_status_id": null,
      "lost_at": null,
      "lost_by": null,
      "fields": {}
    }
  ]
}
```

**Notas (inteligência LL Mídia).** Os filtros de intervalo funcionam aos pares `*_start` (GTE) e `*_end` (LTE) e podem combinar-se (ex.: `created_at_start` + `created_at_end` para uma janela). `tag_ids` e `tag_names` usam OR interno (qualquer tag), por isso separe múltiplos valores por vírgula sem espaços. Para paginar grandes volumes prefira `page` (mais previsível) e mantenha `limit` abaixo do teto de 1000. O parâmetro `fields` é um `deepObject`: passe cada campo personalizado como `fields[chave]=valor`.

---

## `POST` `/v1/deals` — Cria negócio

**O que faz.** Cria um novo negócio.

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.** Nenhum.

**Parâmetros de query.** Nenhum.

**Corpo da requisição.** Schema [`DealCreateSchema`](../03-modelo-de-dados.md#dealcreateschema).

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `origin_id` | string (UUID) | **Sim** | ID da origem à qual o negócio pertence. |
| `name` | string | Não | Nome do contacto associado ao negócio (ex.: `Contact name`). |
| `phone` | string | Não | Telefone do contacto (ex.: `48999999999`). |
| `email` | string | Não | E-mail do contacto (ex.: `contact@email.com`). |
| `username` | string | Não | Identificador de rede (ex.: `Instagram ID`). |
| `value` | number | Não | Valor do negócio (ex.: `200.5`). |
| `stage_id` | string (UUID) | Não | ID da etapa em que o negócio é criado. |
| `user_id` | string (UUID) | Não | ID do utilizador/responsável. |
| `contact_id` | string (UUID) | Não | ID de um contacto existente a associar. |
| `fields` | object | Não | Campos personalizados (pares chave/valor string). Suporta subobjetos `contact` e `organization`, também de pares chave/valor string. |

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `201` | Negócio criado. | Objeto `{ "id": ID }` com o UUID do negócio criado. |

**Exemplo — requisição**
```bash
curl -X POST "https://api.clint.digital/v1/deals" \
  -H "api-token: SUA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "origin_id": "99999999-d77b-4e8b-9d35-fd43e972b999",
    "name": "Contact name",
    "phone": "48999999999",
    "email": "contact@email.com",
    "username": "Instagram ID",
    "value": 200.5,
    "stage_id": "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee",
    "user_id": "11111111-2222-3333-4444-555555555555",
    "fields": {
      "contact": { "cidade": "Lisboa" },
      "organization": { "setor": "Educação" }
    }
  }'
```

**Exemplo — resposta (`201`)**
```json
{
  "id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8"
}
```

**Notas (inteligência LL Mídia).** `origin_id` é o único campo obrigatório — sem ele a criação falha (`422`). Para criar o negócio já ligado a um contacto existente use `contact_id`; em alternativa, passe `name`/`phone`/`email`/`username` para que a Clint resolva/crie o contacto. Os valores de `fields` são strings (incluindo os subobjetos `contact` e `organization`); serialize números e datas como texto. A resposta devolve apenas o `id`: para obter o objeto completo, faça em seguida `GET /v1/deals/{id}`.

---

## `GET` `/v1/deals/{id}` — Obtém negócio

**O que faz.** Devolve um único negócio pelo seu ID.

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.**

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string (UUID) | **Sim** | UUID do negócio. |

**Parâmetros de query.** Nenhum.

**Corpo da requisição.** Nenhum.

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `200` | Objeto do negócio. | `{ "status": integer, "data": `[`Deal`](../03-modelo-de-dados.md#deal)` }` |

**Exemplo — requisição**
```bash
curl -X GET "https://api.clint.digital/v1/deals/8feade82-d77b-4e8b-9d35-fd43e972b5c8" \
  -H "api-token: SUA_API_KEY"
```

**Exemplo — resposta (`200`)**
```json
{
  "status": 200,
  "data": {
    "id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8",
    "origin_id": "99999999-d77b-4e8b-9d35-fd43e972b999",
    "user": {
      "id": "11111111-2222-3333-4444-555555555555",
      "full_name": "User full name"
    },
    "contact": {
      "id": "66666666-7777-8888-9999-000000000000",
      "name": "Contact name",
      "email": "contact@email.com",
      "phone": "+5548999999999"
    },
    "created_at": "2020-01-01T14:15:00.000000+00:00",
    "stage_id": "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee",
    "updated_stage_at": "2020-01-02T09:30:00.000000+00:00",
    "status": "OPEN",
    "won_at": null,
    "won_by": null,
    "lost_status_id": null,
    "lost_at": null,
    "lost_by": null,
    "fields": {}
  }
}
```

**Notas (inteligência LL Mídia).** Ao contrário do endpoint de lista (que envolve os itens em `Paginated`), aqui o envelope é `{ status, data }`, com o negócio em `data`. Os campos de fecho (`won_at`/`won_by`) ou de perda (`lost_status_id`/`lost_at`/`lost_by`) só ficam preenchidos consoante o `status` do negócio (`WON`/`LOST`). ID inexistente devolve `404`.

---

## `POST` `/v1/deals/{id}` — Atualiza negócio

**O que faz.** Atualiza um único negócio existente.

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.**

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string (UUID) | **Sim** | UUID do negócio a atualizar. |

**Parâmetros de query.** Nenhum.

**Corpo da requisição.** Schema [`DealUpdateSchema`](../03-modelo-de-dados.md#dealupdateschema).

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `name` | string | Não | Nome do contacto associado (ex.: `Contact name`). |
| `phone` | string | Não | Telefone do contacto (ex.: `48999999999`). |
| `email` | string | Não | E-mail do contacto (ex.: `contact@email.com`). |
| `value` | number | Não | Valor do negócio (ex.: `200.5`). |
| `stage_id` | string (UUID) | Não | ID da etapa (mover o negócio no funil). |
| `status` | string (enum) | Não | Novo estado: `OPEN`, `WON` ou `LOST` (ver [`DealStatus`](../03-modelo-de-dados.md#dealstatus)). |
| `user_id` | string (UUID) | Não | ID do utilizador/responsável. |
| `origin_id` | string (UUID) | Não | ID da origem. |
| `fields` | object | Não | Campos personalizados (pares chave/valor string). Suporta subobjetos `contact` e `organization`. |

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `200` | Negócio atualizado. | Objeto `{ "id": ID }` com o UUID do negócio. |

**Exemplo — requisição**
```bash
curl -X POST "https://api.clint.digital/v1/deals/8feade82-d77b-4e8b-9d35-fd43e972b5c8" \
  -H "api-token: SUA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "value": 350.0,
    "stage_id": "bbbbbbbb-cccc-dddd-eeee-ffffffffffff",
    "status": "WON"
  }'
```

**Exemplo — resposta (`200`)**
```json
{
  "id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8"
}
```

**Notas (inteligência LL Mídia).** A atualização usa **`POST`** (não `PATCH`/`PUT`) e é parcial: envie apenas os campos que quer alterar. Mudar `stage_id` move o negócio de etapa; alterar `status` para `WON`/`LOST` marca-o como ganho/perdido (os respetivos carimbos de data ficam refletidos no objeto `Deal` ao consultar). Note que `DealUpdateSchema` inclui `status` (que `DealCreateSchema` não tem) mas não inclui `username`/`contact_id` — para associar um contacto na criação use `POST /v1/deals`. A resposta devolve apenas o `id`.

---

## `DELETE` `/v1/deals/{id}` — Remove negócio

**O que faz.** Remove um único negócio pelo seu ID.

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.**

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string (UUID) | **Sim** | UUID do negócio a remover. |

**Parâmetros de query.** Nenhum.

**Corpo da requisição.** Nenhum.

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `204` | No Content — negócio removido, sem corpo de resposta. | — |

**Exemplo — requisição**
```bash
curl -X DELETE "https://api.clint.digital/v1/deals/8feade82-d77b-4e8b-9d35-fd43e972b5c8" \
  -H "api-token: SUA_API_KEY"
```

**Exemplo — resposta (`204`)**
```
(sem corpo)
```

**Notas (inteligência LL Mídia).** Uma remoção bem-sucedida devolve `204 No Content` — sem corpo. A operação é destrutiva; confirme o `id` antes de chamar. ID inexistente devolve `404`.

---

## `GET` `/v2/deals/{id}/history` — Histórico (timeline)

**O que faz.** Devolve a timeline do negócio — os mesmos itens do separador *Histórico* no painel do CRM: mudanças de etapa/estado/responsável, notas, tags, e-mails/SMS/chamadas, atividades concluídas e eventos de integração, para o negócio e o seu contacto. Cada item traz o `body` bruto do evento (formato varia por `category`) e um `summary` pré-renderizado em pt-BR (`null` nas categorias sem sumarizador). Os itens vêm ordenados por data do evento, de forma descendente.

**Autenticação.** Header `api-token` (obrigatório). Escopo **`deals:read`** requerido.

**Parâmetros de caminho.**

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string (UUID) | **Sim** | ID do negócio. |

**Parâmetros de query.**

| Nome | Tipo | Obrigatório | Default | Descrição |
|---|---|---|---|---|
| `limit` | integer (máx. 200) | Não | `200` | Número máximo de itens por página. |
| `page` | integer | Não | `1` | Página do resultado a devolver. |

**Corpo da requisição.** Nenhum.

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `200` | Envelope `PaginatedV2` com itens de histórico. Cada item: `{ id, category, date (UTC ISO), system, user: { id, name } \| null, body (objeto específico do evento) \| null, summary (string pt-BR) \| null }`. | Envelope `PaginatedV2` (ver [Convenções](../02-convencoes.md)). |
| `400` | ID do negócio inválido. | — |
| `403` | A API key não tem o escopo `deals:read`. | — |
| `404` | Negócio não encontrado para esta conta. | — |

**Campos de cada item de histórico.**

| Nome | Tipo | Descrição |
|---|---|---|
| `id` | string | Identificador do item de histórico. |
| `category` | string | Categoria do evento (determina o formato de `body`). |
| `date` | string (UTC ISO) | Data/hora do evento. |
| `system` | — | Indica se o evento foi gerado pelo sistema. |
| `user` | object \| null | Autor do evento: `{ id, name }`, ou `null`. |
| `body` | object \| null | Corpo bruto do evento (formato específico por `category`), ou `null`. |
| `summary` | string \| null | Resumo pré-renderizado em pt-BR, ou `null` para categorias sem sumarizador. |

**Exemplo — requisição**
```bash
curl -X GET "https://api.clint.digital/v2/deals/8feade82-d77b-4e8b-9d35-fd43e972b5c8/history?limit=200&page=1" \
  -H "api-token: SUA_API_KEY"
```

**Exemplo — resposta (`200`)**
```json
{
  "data": [
    {
      "id": "c1a2b3c4-d5e6-7788-99aa-bbccddeeff00",
      "category": "stage_changed",
      "date": "2020-01-02T09:30:00.000Z",
      "system": false,
      "user": {
        "id": "11111111-2222-3333-4444-555555555555",
        "name": "User full name"
      },
      "body": {
        "from_stage_id": "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee",
        "to_stage_id": "bbbbbbbb-cccc-dddd-eeee-ffffffffffff"
      },
      "summary": "Negócio movido de etapa."
    },
    {
      "id": "d2b3c4d5-e6f7-8899-aabb-ccddeeff0011",
      "category": "note",
      "date": "2020-01-01T14:15:00.000Z",
      "system": false,
      "user": {
        "id": "11111111-2222-3333-4444-555555555555",
        "name": "User full name"
      },
      "body": null,
      "summary": null
    }
  ]
}
```

**Notas (inteligência LL Mídia).** Este é um endpoint **`/v2`** e distingue-se das operações `/v1`: exige o escopo **`deals:read`** (senão `403`) e usa o envelope `PaginatedV2`. O `limit` está limitado a 200 (teto mais baixo que o dos endpoints `/v1`). O `body` muda de forma consoante a `category`, por isso trate-o de modo defensivo e não assuma um esquema fixo entre categorias; use `summary` (pt-BR) quando disponível para exibição direta, sabendo que pode vir `null`. Um `id` malformado devolve `400` (diferente do `404` para negócio inexistente). O histórico cobre eventos do negócio **e do seu contacto**.

## Objetos relacionados

- [`Deal`](../03-modelo-de-dados.md#deal) — o negócio completo devolvido nas leituras (lista e detalhe), com `user`, `contact`, etapa, estado e campos de fecho/perda.
- [`DealCreateSchema`](../03-modelo-de-dados.md#dealcreateschema) — corpo de criação; `origin_id` obrigatório.
- [`DealUpdateSchema`](../03-modelo-de-dados.md#dealupdateschema) — corpo de atualização parcial; inclui `status`.
- [`DealStatus`](../03-modelo-de-dados.md#dealstatus) — enum do estado do negócio: `OPEN` (default), `WON`, `LOST`.
- [`Fields`](../03-modelo-de-dados.md#fields) — objeto livre de campos personalizados do negócio.
- [`ID`](../03-modelo-de-dados.md#id) — identificador UUID usado em caminhos, filtros e corpos.
- [`DateTime`](../03-modelo-de-dados.md#datetime) — string ISO-8601 usada nos filtros de data e nos carimbos do negócio.
- [`Paginated`](../03-modelo-de-dados.md#paginated) — envelope de paginação v1 do endpoint de lista.
