# Dashboards — Clint API

Consulta de dashboards de análise (analytics) e dos dados dos respetivos gráficos (charts), na versão `/v2` da API.

> Fonte: OpenAPI oficial da Clint (`@clint-api/v1.0`) · Base `https://api.clint.digital` · Auth: header `api-token` · Documento interno LL Mídia.

## Visão geral

A categoria **Dashboards** expõe os painéis de análise da conta e os dados dos gráficos que os compõem. É uma família de endpoints exclusivamente de leitura (`GET`), toda ela na versão `/v2` da API. Serve para replicar num sistema externo aquilo que se vê no ecrã de dashboards da Clint: listar os painéis existentes, obter a estrutura de um painel (com os seus gráficos e o posicionamento em grelha) e, por fim, executar as consultas de cada gráfico para obter os valores calculados.

O fluxo típico tem três passos. Primeiro, `GET /v2/dashboards` devolve a lista paginada de painéis, cada um com um `charts_count` (quantos gráficos tem). Segundo, `GET /v2/dashboards/{id}` devolve o painel individual com o array `charts` — cada gráfico traz `id`, `name`, `type` e `layout` (posição `x`/`y` e dimensão `w`/`h` na grelha), mas **ainda sem dados**. Terceiro, para obter os valores, chama-se `GET /v2/dashboards/{id}/data` (todos os gráficos do painel de uma vez, paginados a 10 por página) ou `GET /v2/charts/{id}/data` (um único gráfico).

Os dois endpoints de dados aceitam os mesmos filtros analíticos — intervalo de datas (`date_start`/`date_end`), `user_id`, `origin_id`, `origin_group_id`, `tag_id`, `timezone` e um `limit` de linhas (para gráficos do tipo tabela/lista) — que são aplicados às consultas antes da execução. O resultado de cada gráfico vem no schema [`ChartData`](../03-modelo-de-dados.md#chartdata), cujo campo `result` muda de formato consoante o `type` do gráfico: os tipos numéricos (`number`, `kpiPanel`) devolvem `{value}`, os tipos de tabela (`table`) devolvem `{columns, rows}` e os tipos de série (`line`, `area`, `bar`, `stackedBar`, `pie`, `donut`, `funnel`) devolvem um array de objetos de série. Quando a consulta de um gráfico falha, o objeto traz um campo `error` (texto) **em vez de** `result` — no endpoint de painel completo isto é tratado com tolerância a falhas, ou seja, um gráfico que rebente não faz falhar o pedido inteiro.

Notas de autenticação e erros: todas as chamadas exigem o header `api-token`. A ausência ou invalidez do token devolve `401`. Um `id` inexistente devolve `404` (painel ou gráfico não encontrado). Os endpoints de dados podem ainda devolver `400` por erro de validação — tipicamente um formato de data inválido em `date_start`/`date_end` (o padrão esperado é `AAAA-MM-DD`).

## Índice de endpoints

| Método | Caminho | O que faz |
|---|---|---|
| GET | `/v2/dashboards` | Lista paginada de dashboards. |
| GET | `/v2/dashboards/{id}` | Devolve um dashboard individual com os seus gráficos. |
| GET | `/v2/dashboards/{id}/data` | Executa as consultas dos gráficos do dashboard e devolve os dados (paginado a 10 gráficos/página). |
| GET | `/v2/charts/{id}/data` | Executa a consulta de um único gráfico e devolve os seus dados. |

---

## `GET` `/v2/dashboards` — Listar dashboards

**O que faz.** Devolve uma lista paginada de dashboards da conta. Cada item é um resumo ([`DashboardSummary`](../03-modelo-de-dados.md#dashboardsummary)) com identificação, datas e a contagem de gráficos — sem o detalhe dos gráficos.

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.** Nenhum.

**Parâmetros de query.**

| Nome | Tipo | Obrigatório | Default | Descrição |
|---|---|---|---|---|
| `limit` | integer | Não | `200` | Número máximo de linhas devolvidas (mín. `1`, máx. `1000`). |
| `offset` | integer | Não | `0` | Número de linhas ignoradas no início do resultado (mín. `0`). |
| `page` | integer | Não | `1` | Seleciona a página do resultado (mín. `1`). |

**Corpo da requisição.** Nenhum.

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `200` | Lista paginada de dashboards. | [`PaginatedV2`](../03-modelo-de-dados.md#paginatedv2) + `data`: array de [`DashboardSummary`](../03-modelo-de-dados.md#dashboardsummary). |
| `401` | Erro de autenticação — `api-token` inválido ou em falta. | — |

O envelope de paginação `PaginatedV2` traz os campos: `status` (integer), `total_count` (integer — total de itens segundo os filtros atuais), `page` (integer — página atual), `total_pages` (integer), `has_next` (boolean) e `has_previous` (boolean). O campo `data` acrescenta o array de dashboards.

Cada item `DashboardSummary`:

| Campo | Tipo | Descrição |
|---|---|---|
| `id` | string (uuid) | Identificador do dashboard. |
| `name` | string | Nome do dashboard. |
| `created_at` | string (date-time) | Data de criação (ISO-8601). |
| `updated_at` | string (date-time) | Data da última atualização (ISO-8601). |
| `charts_count` | integer | Número de gráficos no dashboard. |

**Exemplo — requisição**
```bash
curl -X GET "https://api.clint.digital/v2/dashboards?limit=200&page=1" \
  -H "api-token: SUA_API_KEY"
```

**Exemplo — resposta (`200`)**
```json
{
  "status": 200,
  "total_count": 50,
  "page": 1,
  "total_pages": 10,
  "has_next": true,
  "has_previous": false,
  "data": [
    {
      "id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8",
      "name": "Sales Dashboard",
      "created_at": "2020-01-01T14:15:00.000000+00:00",
      "updated_at": "2020-01-01T14:15:00.000000+00:00",
      "charts_count": 12
    }
  ]
}
```

**Notas (inteligência LL Mídia).** Endpoint `/v2`, por isso a paginação usa o envelope `PaginatedV2` (chaves em snake_case: `total_count`, `total_pages`, `has_next`, `has_previous`) — diferente do envelope `Paginated` dos endpoints `/v1`. Repare que a spec expõe simultaneamente `offset` e `page`; para navegar por páginas de forma previsível, use `page` + `limit`. O `charts_count` é útil para estimar o custo antes de puxar os dados: um painel com muitos gráficos vai paginar (10 por página) no endpoint `/data`.

---

## `GET` `/v2/dashboards/{id}` — Obter um dashboard

**O que faz.** Devolve um único dashboard com a sua lista de gráficos. Cada gráfico traz `id`, `name`, `type` e `layout` (posição e dimensão na grelha do painel), mas **não** traz dados calculados — para os dados, use os endpoints `/data`.

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.**

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string (uuid) | Sim | UUID do dashboard. |

**Parâmetros de query.** Nenhum.

**Corpo da requisição.** Nenhum.

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `200` | Detalhes do dashboard com gráficos. | Objeto `{ status, data }`, onde `data` é [`Dashboard`](../03-modelo-de-dados.md#dashboard). |
| `401` | Erro de autenticação — `api-token` inválido ou em falta. | — |
| `404` | Dashboard não encontrado. | — |

Estrutura de [`Dashboard`](../03-modelo-de-dados.md#dashboard):

| Campo | Tipo | Descrição |
|---|---|---|
| `id` | string (uuid) | Identificador do dashboard. |
| `name` | string | Nome do dashboard. |
| `created_at` | string (date-time) | Data de criação (ISO-8601). |
| `updated_at` | string (date-time) | Data da última atualização (ISO-8601). |
| `charts` | array de objetos | Lista de gráficos do dashboard (ver abaixo). |

Cada item de `charts`:

| Campo | Tipo | Descrição |
|---|---|---|
| `id` | string (uuid) | Identificador do gráfico. |
| `name` | string | Nome do gráfico. |
| `type` | string (enum) | Tipo do gráfico. Valores: `line`, `area`, `bar`, `stackedBar`, `table`, `number`, `kpiPanel`, `pie`, `donut`, `funnel`. |
| `layout` | object | Posição do gráfico na grelha do dashboard: `x` (integer), `y` (integer), `w` (integer — largura), `h` (integer — altura). |

**Exemplo — requisição**
```bash
curl -X GET "https://api.clint.digital/v2/dashboards/8feade82-d77b-4e8b-9d35-fd43e972b5c8" \
  -H "api-token: SUA_API_KEY"
```

**Exemplo — resposta (`200`)**
```json
{
  "status": 200,
  "data": {
    "id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8",
    "name": "Sales Dashboard",
    "created_at": "2020-01-01T14:15:00.000000+00:00",
    "updated_at": "2020-01-01T14:15:00.000000+00:00",
    "charts": [
      {
        "id": "3f2a9c10-1b4e-4c8a-9f21-7d6e5a4b3c2d",
        "name": "Monthly Revenue",
        "type": "bar",
        "layout": { "x": 0, "y": 0, "w": 6, "h": 4 }
      }
    ]
  }
}
```

**Notas (inteligência LL Mídia).** Este endpoint devolve a **estrutura** do painel (metadados e disposição dos gráficos), não os números. O `layout` (`x`, `y`, `w`, `h`) reproduz a grelha do dashboard na Clint e é útil se quiser recriar o painel numa UI própria. Para obter os valores dos gráficos, encadeie com `GET /v2/dashboards/{id}/data` (usando o mesmo `id`) ou, gráfico a gráfico, com `GET /v2/charts/{id}/data` (usando os `id` de cada item de `charts`). Um `id` de dashboard inexistente devolve `404`.

---

## `GET` `/v2/dashboards/{id}/data` — Dados dos gráficos do dashboard

**O que faz.** Executa as consultas dos gráficos do dashboard e devolve os respetivos dados. Os gráficos são paginados a **10 por página**. Falhas de gráficos individuais são tratadas com tolerância: em vez de fazer falhar o pedido inteiro, o gráfico problemático devolve uma mensagem de erro no campo `error`.

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.**

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string (uuid) | Sim | UUID do dashboard. |

**Parâmetros de query.**

| Nome | Tipo | Obrigatório | Default | Descrição |
|---|---|---|---|---|
| `page` | integer | Não | `1` | Seleciona a página de gráficos do resultado (mín. `1`). Cada página traz até 10 gráficos. |
| `date_start` | string | Não | — | Filtro de data inicial (inclusivo). Formato `AAAA-MM-DD` (ex.: `2026-01-01`). |
| `date_end` | string | Não | — | Filtro de data final (inclusivo). Formato `AAAA-MM-DD` (ex.: `2026-12-31`). |
| `user_id` | string (uuid) | Não | — | Filtra por ID de utilizador. |
| `origin_id` | string (uuid) | Não | — | Filtra por ID de origem. |
| `origin_group_id` | string (uuid) | Não | — | Filtra por ID de grupo de origens. |
| `tag_id` | string (uuid) | Não | — | Filtra por ID de tag. |
| `timezone` | string | Não | — | Fuso horário para os cálculos de datas (máx. 50 caracteres, ex.: `America/Sao_Paulo`). |
| `limit` | integer | Não | — | Limita as linhas devolvidas por gráfico (aplica-se aos tipos tabela/lista). Mín. `1`, máx. `15000`. |
| `chart_ids` | string | Não | — | Lista de IDs de gráficos separados por vírgula; devolve apenas esses gráficos em vez de todos (ex.: `uuid-1,uuid-2`). |

**Corpo da requisição.** Nenhum.

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `200` | Dados dos gráficos do dashboard. | [`PaginatedV2`](../03-modelo-de-dados.md#paginatedv2) + `data.charts`: array de [`ChartData`](../03-modelo-de-dados.md#chartdata). |
| `400` | Erro de validação (ex.: formato de data inválido). | — |
| `401` | Erro de autenticação — `api-token` inválido ou em falta. | — |
| `404` | Dashboard não encontrado. | — |

O resultado combina o envelope [`PaginatedV2`](../03-modelo-de-dados.md#paginatedv2) (`status`, `total_count`, `page`, `total_pages`, `has_next`, `has_previous`) com um objeto `data` que contém o array `charts`, em que cada item é um [`ChartData`](../03-modelo-de-dados.md#chartdata):

| Campo | Tipo | Descrição |
|---|---|---|
| `id` | string (uuid) | Identificador do gráfico. |
| `name` | string | Nome do gráfico. |
| `type` | string (enum) | Tipo do gráfico. Valores: `line`, `area`, `bar`, `stackedBar`, `table`, `number`, `kpiPanel`, `pie`, `donut`, `funnel`. |
| `result` | ver abaixo | Dados do gráfico (Cube.js). O formato depende do `type`. Ausente quando há `error`. |
| `error` | string | Presente em vez de `result` quando a consulta do gráfico falhou (ex.: `"Failed to fetch chart data"`). |

Formatos possíveis de `result` (campo `oneOf`):

- **Número/KPI** (`number`, `kpiPanel`): objeto `{ "value": <número> }`.
- **Tabela** (`table`): objeto `{ "columns": [ { "key", "title", "type" } ], "rows": [ { ... } ] }`.
- **Série** (`line`, `area`, `bar`, `stackedBar`, `pie`, `donut`, `funnel`): array de objetos de série `[ { "name": <string>, "data": [ { ... } ] } ]`.

**Exemplo — requisição**
```bash
curl -X GET "https://api.clint.digital/v2/dashboards/8feade82-d77b-4e8b-9d35-fd43e972b5c8/data?page=1&date_start=2026-01-01&date_end=2026-12-31&timezone=America/Sao_Paulo" \
  -H "api-token: SUA_API_KEY"
```

**Exemplo — resposta (`200`)**
```json
{
  "status": 200,
  "total_count": 12,
  "page": 1,
  "total_pages": 2,
  "has_next": true,
  "has_previous": false,
  "data": {
    "charts": [
      {
        "id": "3f2a9c10-1b4e-4c8a-9f21-7d6e5a4b3c2d",
        "name": "Monthly Revenue",
        "type": "number",
        "result": { "value": 42 }
      },
      {
        "id": "a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d",
        "name": "Leads by Origin",
        "type": "table",
        "result": {
          "columns": [
            { "key": "origin", "title": "Origem", "type": "string" },
            { "key": "count", "title": "Total", "type": "number" }
          ],
          "rows": [
            { "origin": "Website", "count": 128 },
            { "origin": "Instagram", "count": 74 }
          ]
        }
      },
      {
        "id": "b2c3d4e5-f6a7-4b8c-9d0e-1f2a3b4c5d6e",
        "name": "Revenue Trend",
        "type": "line",
        "result": [
          {
            "name": "2026",
            "data": [
              { "x": "2026-01", "y": 12000 },
              { "x": "2026-02", "y": 15300 }
            ]
          }
        ]
      },
      {
        "id": "c3d4e5f6-a7b8-4c9d-0e1f-2a3b4c5d6e7f",
        "name": "Broken Chart",
        "type": "bar",
        "error": "Failed to fetch chart data"
      }
    ]
  }
}
```

**Notas (inteligência LL Mídia).** Este é o endpoint que efetivamente **executa** as consultas (Cube.js) e devolve valores — é o mais pesado da categoria. A paginação aqui é por **gráficos** (10 por página), não por linhas: se o painel tiver mais de 10 gráficos, percorra as páginas com `page` até `has_next` ser `false`. Use `chart_ids` para pedir apenas alguns gráficos específicos e poupar processamento; e `limit` para conter o número de linhas nos gráficos de tabela/lista (até 15000). Trate sempre cada item defensivamente: verifique a presença de `error` antes de ler `result`, porque a tolerância a falhas devolve gráficos parcialmente falhados sem rebentar o `200`. O `400` costuma vir de datas mal formatadas — respeite o padrão `AAAA-MM-DD` em `date_start`/`date_end`.

---

## `GET` `/v2/charts/{id}/data` — Dados de um único gráfico

**O que faz.** Executa a consulta de um único gráfico (por `id`) e devolve os seus dados. Aceita os mesmos filtros analíticos do endpoint de painel, exceto a paginação por página (é um só gráfico).

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.**

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string (uuid) | Sim | UUID do gráfico. |

**Parâmetros de query.**

| Nome | Tipo | Obrigatório | Default | Descrição |
|---|---|---|---|---|
| `date_start` | string | Não | — | Filtro de data inicial (inclusivo). Formato `AAAA-MM-DD` (ex.: `2026-01-01`). |
| `date_end` | string | Não | — | Filtro de data final (inclusivo). Formato `AAAA-MM-DD` (ex.: `2026-12-31`). |
| `user_id` | string (uuid) | Não | — | Filtra por ID de utilizador. |
| `origin_id` | string (uuid) | Não | — | Filtra por ID de origem. |
| `origin_group_id` | string (uuid) | Não | — | Filtra por ID de grupo de origens. |
| `tag_id` | string (uuid) | Não | — | Filtra por ID de tag. |
| `timezone` | string | Não | — | Fuso horário para os cálculos de datas (máx. 50 caracteres, ex.: `America/Sao_Paulo`). |
| `limit` | integer | Não | — | Limita as linhas devolvidas (aplica-se aos tipos tabela/lista). Mín. `1`, máx. `15000`. |

**Corpo da requisição.** Nenhum.

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `200` | Dados do gráfico. | Objeto `{ status, data }`, onde `data` é [`ChartData`](../03-modelo-de-dados.md#chartdata). |
| `400` | Erro de validação (ex.: formato de data inválido). | — |
| `401` | Erro de autenticação — `api-token` inválido ou em falta. | — |
| `404` | Gráfico não encontrado. | — |

O campo `data` é um [`ChartData`](../03-modelo-de-dados.md#chartdata) (mesma estrutura descrita no endpoint anterior: `id`, `name`, `type`, `result` — variável por tipo — ou `error`).

**Exemplo — requisição**
```bash
curl -X GET "https://api.clint.digital/v2/charts/3f2a9c10-1b4e-4c8a-9f21-7d6e5a4b3c2d/data?date_start=2026-01-01&date_end=2026-12-31&limit=500" \
  -H "api-token: SUA_API_KEY"
```

**Exemplo — resposta (`200`)**
```json
{
  "status": 200,
  "data": {
    "id": "3f2a9c10-1b4e-4c8a-9f21-7d6e5a4b3c2d",
    "name": "Monthly Revenue",
    "type": "number",
    "result": { "value": 42 }
  }
}
```

**Notas (inteligência LL Mídia).** Use este endpoint quando quiser refrescar apenas um gráfico (por exemplo, um KPI num widget) sem puxar o painel inteiro — é mais leve que `/v2/dashboards/{id}/data`. O `id` aqui é o **id do gráfico** (obtido de `charts[].id` em `GET /v2/dashboards/{id}`), não o id do dashboard; passar um id de dashboard aqui devolve `404`. Tal como no endpoint de painel, o formato de `result` depende do `type`, e um gráfico falhado pode trazer `error` em vez de `result` — valide antes de ler. Datas fora do padrão `AAAA-MM-DD` produzem `400`.

---

## Objetos relacionados

- [`DashboardSummary`](../03-modelo-de-dados.md#dashboardsummary) — resumo de dashboard usado na listagem (`id`, `name`, datas, `charts_count`).
- [`Dashboard`](../03-modelo-de-dados.md#dashboard) — dashboard completo com o array `charts` (cada gráfico com `type` e `layout`).
- [`ChartData`](../03-modelo-de-dados.md#chartdata) — dados de um gráfico (Cube.js); `result` varia por tipo, ou `error` quando a consulta falha.
- [`PaginatedV2`](../03-modelo-de-dados.md#paginatedv2) — envelope de paginação v2 (snake_case) usado na listagem e nos dados do painel.
- [`ID`](../03-modelo-de-dados.md#id) — string UUID usada em identificadores e filtros.
- [`DateTime`](../03-modelo-de-dados.md#datetime) — string data-hora ISO-8601 (`created_at`, `updated_at`).
