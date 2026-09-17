# Activities — Clint API

Atividades de negócio (deal activities) — chamadas, tarefas, reuniões, agendamentos e outros follow-ups ligados a um negócio (deal).

> Fonte: OpenAPI oficial da Clint (`@clint-api/v1.0`) · Base `https://api.clint.digital` · Auth: header `api-token` · Documento interno LL Mídia.

## Visão geral

O recurso **Activities** representa as atividades de acompanhamento associadas a um negócio (deal): chamadas (`CALL`), e-mails (`MAIL`), agendamentos (`SCHEDULE`), tarefas (`TASK`), reuniões (`MEETING`), mensagens de WhatsApp (`WHATSAPP`) e mensagens de Instagram (`INSTAGRAM`). Cada atividade tem um título, um tipo, um corpo de texto livre opcional (`content`), uma data de vencimento (`due_at`), um estado de conclusão (`completed`) e fica ancorada a uma fase (`stage_id`) do funil da origem do negócio.

Estes endpoints vivem em **`/v2`** (omnichannel/atividades) e estão **fechados atrás da feature flag `ACTIVITIES_API`**. Conforme o `tag_description` oficial: *"Operations related to deal activities — calls, tasks, meetings, schedules and other follow-ups linked to a deal. Gated by the `ACTIVITIES_API` feature flag and the `activities:read`/`activities:write` API key scopes (v2)."* Na prática, isto significa dois níveis de gating:

1. **Feature flag `ACTIVITIES_API`** — tem de estar activa na conta. Se não estiver, qualquer chamada devolve `403` com a mensagem `"This feature is not available for your account."`.
2. **Escopos da API key** — a chave usada no header `api-token` precisa do escopo `activities:read` para operações de leitura (`GET /v2/activities`, `GET /v2/activities/{id}`) e do escopo `activities:write` para operações de escrita (`POST`, `DELETE`, `complete`).

**Nota de consistência (importante).** A listagem (`GET /v2/activities`) é servida a partir do **índice de pesquisa** e é *eventualmente consistente* — tipicamente alguns segundos atrás das escritas. Já o `GET /v2/activities/{id}` lê da **base de dados primária** e está sempre actualizado. Se precisar de confirmar uma alteração acabada de escrever (criação, update, conclusão), consulte o endpoint por ID em vez de fazer polling à lista.

Há ainda dois conceitos que atravessam vários endpoints e convém interiorizar desde já:

- **Atividades `custom` vs. atividades de template.** O campo `custom` vale `true` quando a atividade foi criada manualmente (via API ou UI) e `false` quando foi gerada automaticamente a partir de um template de fase (stage template). Esta distinção tem consequências reais no endpoint de conclusão/reabertura (ver abaixo).
- **`stage_id` e a origem do negócio.** O `stage_id` de uma atividade tem de pertencer a uma fase da **origem do negócio** (deal's origin). Ao criar, se omitido, assume por defeito a fase actual do negócio.

## Índice de endpoints

| Método | Caminho | O que faz |
|---|---|---|
| GET | `/v2/activities` | Lista paginada de atividades, com múltiplos filtros e intervalo de datas. |
| POST | `/v2/activities` | Cria uma atividade num negócio. |
| GET | `/v2/activities/{id}` | Obtém uma atividade por ID (leitura fresca da base primária). |
| POST | `/v2/activities/{id}` | Actualiza um ou mais campos de uma atividade. |
| DELETE | `/v2/activities/{id}` | Elimina (soft-delete) uma atividade. |
| POST | `/v2/activities/{id}/complete` | Conclui ou reabre uma atividade. |

---

## `GET` `/v2/activities` — Listar atividades

**O que faz.** Devolve uma lista paginada de atividades de negócio, filtrável por tipo, estado da atividade, estado do negócio, origem, fase, utilizador atribuído, negócio, texto livre, tags e um intervalo de datas. Requer a feature flag `ACTIVITIES_API` e uma API key com o escopo `activities:read`.

> **Consistência.** Esta listagem é servida do índice de pesquisa e é eventualmente consistente (tipicamente segundos atrás das escritas). Para confirmar uma alteração acabada de gravar, use `GET /v2/activities/{id}`, que lê da base primária e está sempre fresco.

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.** Nenhum.

**Parâmetros de query.**

| Nome | Tipo | Obrigatório | Default | Descrição |
|---|---|---|---|---|
| `type` | string | Não | — | Lista separada por vírgulas de tipos de atividade a filtrar (`CALL`, `MAIL`, `SCHEDULE`, `TASK`, `MEETING`, `WHATSAPP`, `INSTAGRAM`). Ex.: `CALL,TASK`. |
| `activity_status` | string | Não | — | Lista separada por vírgulas de estados de atividade a filtrar (`DELAYED`, `ON_TIME`, `COMPLETE`, `IGNORED`). Ex.: `DELAYED,ON_TIME`. |
| `deal_status` | string | Não | — | Lista separada por vírgulas de estados de negócio a filtrar (`OPEN`, `WON`, `LOST`). Ex.: `OPEN`. |
| `origin_ids` | string | Não | — | Lista separada por vírgulas de UUIDs de origem a filtrar. |
| `stage_ids` | string | Não | — | Lista separada por vírgulas de UUIDs de fase a filtrar. |
| `user_id` | string (uuid) | Não | — | Filtra pelo utilizador atribuído ao negócio (UUID único). |
| `deal_id` | string (uuid) | Não | — | Filtra por um único UUID de negócio. |
| `text` | string | Não | — | Pesquisa de texto livre no título e no conteúdo da atividade. |
| `tags` | string | Não | — | Lista separada por vírgulas de UUIDs de tag. Uma atividade corresponde quando o seu negócio tem **qualquer** uma destas tags. |
| `tags_exclude` | string | Não | — | Lista separada por vírgulas de UUIDs de tag a excluir. Uma atividade é excluída quando o seu negócio tem qualquer uma destas tags. |
| `date_type` | string (enum) | Não | `due` | Contra que campo de data os filtros `date_start`/`date_end` são aplicados. Valores: `due` (data de vencimento da atividade — default), `created` (data de criação da atividade), `won`/`lost` (o `won_at`/`lost_at` do negócio). |
| `date_start` | string (date-time) | Não | — | Início inclusivo do intervalo de datas (ISO 8601), aplicado ao campo escolhido por `date_type`. |
| `date_end` | string (date-time) | Não | — | Fim inclusivo do intervalo de datas (ISO 8601), aplicado ao campo escolhido por `date_type`. |
| `limit` | integer | Não | `200` | Número máximo de linhas devolvidas (mín. `1`, máx. `200`). |
| `page` | integer | Não | `1` | Página do resultado (mín. `1`). |

**Corpo da requisição.** Nenhum.

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `200` | Lista paginada de atividades. | `PaginatedV2` + `data: Activity[]` |
| `400` | Parâmetro `type` inválido (tem de ser um de `CALL, MAIL, SCHEDULE, TASK, MEETING, WHATSAPP, INSTAGRAM`). | `{ status, message }` |
| `401` | Erro de autenticação — `api-token` inválido ou em falta. | — |
| `403` | Funcionalidade indisponível na conta (feature flag `ACTIVITIES_API` em falta). | `{ status, message }` |

O envelope `PaginatedV2` inclui: `status` (integer), `total_count` (integer — total de itens para os filtros actuais), `page` (integer — página actual), `total_pages` (integer — total de páginas), `has_next` (boolean) e `has_previous` (boolean). O array `data` contém objetos `Activity`.

**Exemplo — requisição**
```bash
curl -X GET "https://api.clint.digital/v2/activities?type=CALL,TASK&activity_status=DELAYED,ON_TIME&deal_status=OPEN&date_type=due&date_start=2024-01-01T00:00:00.000Z&date_end=2024-01-31T23:59:59.999Z&limit=200&page=1" \
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
      "title": "Call the customer back",
      "type": "CALL",
      "content": "Discuss the renewal proposal.",
      "template_id": null,
      "completed": false,
      "completed_at": null,
      "completed_by": null,
      "due_at": "2024-01-20T14:00:00.000Z",
      "custom": true,
      "no_show": false,
      "stage_id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8",
      "created_at": "2020-01-01T14:15:00.000000+00:00",
      "updated_at": null,
      "deal": {
        "id": "550e8400-e29b-41d4-a716-446655440000",
        "status": "OPEN",
        "origin_id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8",
        "stage_id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8",
        "contact": {
          "id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
          "name": "Jane Doe"
        }
      }
    }
  ]
}
```

**Notas (inteligência LL Mídia).**
- Todos os filtros de lista (`type`, `activity_status`, `deal_status`, `origin_ids`, `stage_ids`, `tags`, `tags_exclude`) aceitam **múltiplos valores separados por vírgula** — não repita o parâmetro na query string. Já `user_id` e `deal_id` são UUID **único**.
- `date_type` controla contra que campo o intervalo `date_start`/`date_end` incide. O default é `due`; se quiser um relatório de atividades *criadas* num período use `date_type=created`, e para análises ligadas ao fecho do negócio use `won`/`lost`.
- `tags` e `tags_exclude` filtram pelas tags do **negócio** da atividade, não da atividade em si. Semântica "OR" dentro de cada lista (qualquer tag corresponde/exclui).
- Máximo rígido de `limit=200` por página (mais restritivo que o `limit` de v1). Pagine com `page` e vigie `has_next`/`total_pages`.
- Lembre-se da consistência eventual: logo após um `POST`/update, a atividade pode ainda não aparecer na lista. Para confirmação imediata, leia por ID.

---

## `POST` `/v2/activities` — Criar atividade

**O que faz.** Cria uma atividade num negócio. Requer a feature flag `ACTIVITIES_API` e uma API key com o escopo `activities:write`. O `stage_id` tem de ser uma fase pertencente à origem do negócio — quando omitido, assume por defeito a fase actual do negócio.

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.** Nenhum.

**Parâmetros de query.** Nenhum.

**Corpo da requisição.** `application/json` (obrigatório).

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `deal_id` | string (uuid) | **Sim** | UUID do negócio a que a atividade pertence. Tem de pertencer ao owner autenticado. |
| `title` | string | **Sim** | Título da atividade. |
| `type` | string (enum) | **Sim** | Tipo da atividade. Um de: `CALL`, `MAIL`, `SCHEDULE`, `TASK`, `MEETING`, `WHATSAPP`, `INSTAGRAM`. |
| `due_at` | string (date-time), nullable | Não | Data/hora de vencimento (ISO 8601). |
| `content` | string, nullable | Não | Corpo de texto livre da atividade. Para `WHATSAPP` e `INSTAGRAM` é o texto da mensagem (guardado para renderizar na vista de mensagens da UI Clint); para `CALL`/`MAIL`/`TASK` é o campo de notas. |
| `template_id` | string (uuid) | Não | Template de mensagem WhatsApp Official opcional a anexar. Só suportado quando `type` é `WHATSAPP`; tem de referenciar um template pertencente ao owner autenticado. |
| `stage_id` | string (uuid) | Não | Fase da origem do negócio à qual associar a atividade. Por defeito, a fase actual do negócio quando omitido. |

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `201` | Objeto da atividade criada. | `{ status, data: Activity }` |
| `400` | Validação — ex.: `Param deal_id is required and must be a valid UUID`. | `{ status, message }` |
| `401` | Erro de autenticação — `api-token` inválido ou em falta. | — |
| `403` | Funcionalidade indisponível na conta (feature flag `ACTIVITIES_API` em falta). | `{ status, message }` |
| `404` | `Deal not found`. | `{ status, message }` |

**Exemplo — requisição**
```bash
curl -X POST "https://api.clint.digital/v2/activities" \
  -H "api-token: SUA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "deal_id": "550e8400-e29b-41d4-a716-446655440000",
    "title": "Call the customer back",
    "type": "CALL",
    "due_at": "2024-01-20T14:00:00.000Z",
    "content": "Discuss the renewal proposal.",
    "stage_id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8"
  }'
```

**Exemplo — resposta (`201`)**
```json
{
  "status": 201,
  "data": {
    "id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8",
    "title": "Call the customer back",
    "type": "CALL",
    "content": "Discuss the renewal proposal.",
    "template_id": null,
    "completed": false,
    "completed_at": null,
    "completed_by": null,
    "due_at": "2024-01-20T14:00:00.000Z",
    "custom": true,
    "no_show": false,
    "stage_id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8",
    "created_at": "2020-01-01T14:15:00.000000+00:00",
    "updated_at": null,
    "deal": {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "status": "OPEN",
      "origin_id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8",
      "stage_id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8",
      "contact": {
        "id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
        "name": "Jane Doe"
      }
    }
  }
}
```

**Notas (inteligência LL Mídia).**
- Campos obrigatórios: `deal_id`, `title` e `type`. O negócio referenciado por `deal_id` tem de pertencer ao owner autenticado — caso contrário, `404 Deal not found`.
- Atividades criadas por aqui saem com `custom: true` (criação manual via API). Isto distingue-as das atividades geradas automaticamente por templates de fase (`custom: false`).
- `template_id` é **exclusivo do tipo `WHATSAPP`**. Anexar um template a uma atividade de outro tipo não é suportado.
- Se omitir `stage_id`, a atividade fica na fase actual do negócio. Se o passar, tem de ser uma fase da **mesma origem** do negócio.
- Para `WHATSAPP`/`INSTAGRAM`, o `content` é o texto da mensagem que a UI Clint renderiza na vista de mensagens; para os restantes tipos é o campo de notas.

---

## `GET` `/v2/activities/{id}` — Obter atividade

**O que faz.** Devolve uma única atividade por ID. Só devolve atividades pertencentes ao owner autenticado. Requer a feature flag `ACTIVITIES_API` e uma API key com o escopo `activities:read`. Ao contrário do endpoint de listagem, este lê da base de dados primária e está sempre fresco.

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.**

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string (uuid) | **Sim** | UUID da atividade. |

**Parâmetros de query.** Nenhum.

**Corpo da requisição.** Nenhum.

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `200` | Objeto da atividade. | `{ status, data: Activity }` |
| `400` | `Param ID must be a valid UUID`. | `{ status, message }` |
| `401` | Erro de autenticação — `api-token` inválido ou em falta. | — |
| `403` | Funcionalidade indisponível na conta (feature flag `ACTIVITIES_API` em falta). | `{ status, message }` |
| `404` | `Activity not found`. | `{ status, message }` |

**Exemplo — requisição**
```bash
curl -X GET "https://api.clint.digital/v2/activities/8feade82-d77b-4e8b-9d35-fd43e972b5c8" \
  -H "api-token: SUA_API_KEY"
```

**Exemplo — resposta (`200`)**
```json
{
  "status": 200,
  "data": {
    "id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8",
    "title": "Call the customer back",
    "type": "CALL",
    "content": "Discuss the renewal proposal.",
    "template_id": null,
    "completed": false,
    "completed_at": null,
    "completed_by": null,
    "due_at": "2024-01-20T14:00:00.000Z",
    "custom": true,
    "no_show": false,
    "stage_id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8",
    "created_at": "2020-01-01T14:15:00.000000+00:00",
    "updated_at": null,
    "deal": {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "status": "OPEN",
      "origin_id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8",
      "stage_id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8",
      "contact": {
        "id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
        "name": "Jane Doe"
      }
    }
  }
}
```

**Campos do objeto `Activity` (resposta).**

| Nome | Tipo | Descrição |
|---|---|---|
| `id` | string (uuid) | Identificador da atividade. |
| `title` | string | Título da atividade. |
| `type` | string (enum) | `CALL`, `MAIL`, `SCHEDULE`, `TASK`, `MEETING`, `WHATSAPP`, `INSTAGRAM`. |
| `content` | string, nullable | Corpo de texto livre. Para `WHATSAPP`/`INSTAGRAM` é o texto da mensagem (lido do corpo da mensagem que a UI Clint renderiza); para `CALL`/`MAIL`/`TASK` é o campo de notas. |
| `template_id` | string (uuid), nullable | UUID do template de mensagem WhatsApp Official anexado, ou `null` quando nenhum está anexado. |
| `completed` | boolean | Se a atividade está concluída. |
| `completed_at` | string (date-time), nullable | Data/hora de conclusão, ou `null`. |
| `completed_by` | string (uuid), nullable | UUID do utilizador que concluiu, ou `null`. |
| `due_at` | string (date-time), nullable | Data/hora de vencimento. Alias público da coluna interna `to_check_at`. |
| `custom` | boolean | `true` quando a atividade foi criada manualmente (via API ou UI); `false` quando gerada automaticamente a partir de um template de fase. |
| `no_show` | boolean | Marca de "não compareceu". |
| `stage_id` | string (uuid) | Fase à qual a atividade está associada. |
| `created_at` | string (date-time) | Data/hora de criação (schema `DateTime`). |
| `updated_at` | string (date-time), nullable | Data/hora da última actualização, ou `null`. |
| `deal` | object, nullable | Negócio da atividade (ver abaixo), ou `null`. |
| `deal.id` | string (uuid) | UUID do negócio. |
| `deal.status` | string (enum), nullable | `OPEN`, `WON`, `LOST`, ou `null`. |
| `deal.origin_id` | string (uuid) | UUID da origem do negócio. |
| `deal.stage_id` | string (uuid) | UUID da fase actual do negócio. |
| `deal.contact` | object, nullable | Contacto do negócio, ou `null`. |
| `deal.contact.id` | string (uuid) | UUID do contacto. |
| `deal.contact.name` | string, nullable | Nome do contacto, ou `null`. |

**Notas (inteligência LL Mídia).**
- Este é o endpoint a usar sempre que precisar de **confiança na frescura** do dado (leitura da base primária). Use-o para confirmar criações, updates e conclusões acabadas de executar, em vez de fazer polling à lista.
- Atividades de outro owner devolvem `404 Activity not found` (não `403`), o que evita fuga de existência de recursos entre contas.

---

## `POST` `/v2/activities/{id}` — Actualizar atividade

**O que faz.** Actualiza um ou mais campos de uma atividade. É obrigatório enviar **pelo menos um** de `title`, `type`, `content`, `template_id`, `due_at` ou `stage_id`. O `stage_id` tem de ser uma fase pertencente à origem do negócio. O `template_id` só é suportado para atividades `WHATSAPP` (passe `null` para desanexar). Requer a feature flag `ACTIVITIES_API` e uma API key com o escopo `activities:write`. Só atividades pertencentes ao owner autenticado podem ser actualizadas.

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.**

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string (uuid) | **Sim** | UUID da atividade. |

**Parâmetros de query.** Nenhum.

**Corpo da requisição.** `application/json` (obrigatório). `minProperties: 1` — pelo menos um campo tem de ser fornecido.

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `title` | string | Não* | Novo título da atividade. |
| `type` | string (enum) | Não* | Novo tipo. Um de: `CALL`, `MAIL`, `SCHEDULE`, `TASK`, `MEETING`, `WHATSAPP`, `INSTAGRAM`. |
| `content` | string, nullable | Não* | Corpo de texto livre. Para `WHATSAPP`/`INSTAGRAM` é o texto da mensagem; para `CALL`/`MAIL`/`TASK` é o campo de notas. |
| `template_id` | string (uuid), nullable | Não* | Anexar (UUID) ou desanexar (`null`) um template de mensagem WhatsApp Official. Só suportado quando o tipo da atividade é `WHATSAPP`; tem de referenciar um template pertencente ao owner autenticado. |
| `due_at` | string (date-time), nullable | Não* | Nova data/hora de vencimento (ISO 8601). |
| `stage_id` | string (uuid) | Não* | Nova fase (da origem do negócio) a associar. |

\* Nenhum campo é individualmente obrigatório, mas **pelo menos um** dos seis tem de estar presente (`minProperties: 1`).

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `200` | Objeto da atividade actualizada. | `{ status, data: Activity }` |
| `400` | `At least one of title, type, content, template_id, due_at, stage_id is required`. | `{ status, message }` |
| `401` | Erro de autenticação — `api-token` inválido ou em falta. | — |
| `403` | Funcionalidade indisponível na conta (feature flag `ACTIVITIES_API` em falta). | `{ status, message }` |
| `404` | `Activity not found`. | `{ status, message }` |

**Exemplo — requisição**
```bash
curl -X POST "https://api.clint.digital/v2/activities/8feade82-d77b-4e8b-9d35-fd43e972b5c8" \
  -H "api-token: SUA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Call the customer back tomorrow",
    "content": "Updated notes.",
    "due_at": "2024-01-21T14:00:00.000Z",
    "stage_id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8"
  }'
```

**Exemplo — resposta (`200`)**
```json
{
  "status": 200,
  "data": {
    "id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8",
    "title": "Call the customer back tomorrow",
    "type": "CALL",
    "content": "Updated notes.",
    "template_id": null,
    "completed": false,
    "completed_at": null,
    "completed_by": null,
    "due_at": "2024-01-21T14:00:00.000Z",
    "custom": true,
    "no_show": false,
    "stage_id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8",
    "created_at": "2020-01-01T14:15:00.000000+00:00",
    "updated_at": "2024-01-15T09:30:00.000Z",
    "deal": {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "status": "OPEN",
      "origin_id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8",
      "stage_id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8",
      "contact": {
        "id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
        "name": "Jane Doe"
      }
    }
  }
}
```

**Notas (inteligência LL Mídia).**
- Repare que este endpoint usa `POST` (não `PATCH`/`PUT`) no mesmo caminho do `GET` por ID. É uma actualização parcial: envie apenas os campos a mudar, mas nunca um corpo vazio (`400`).
- `template_id: null` **desanexa** o template. Anexar/atualizar `template_id` só faz sentido em atividades `WHATSAPP`.
- Ao mudar `stage_id`, garanta que a fase pertence à origem do negócio da atividade.
- A resposta traz `updated_at` preenchido após a alteração (era `null` numa atividade nunca actualizada).

---

## `DELETE` `/v2/activities/{id}` — Eliminar atividade

**O que faz.** Elimina (soft-delete) uma atividade. Requer a feature flag `ACTIVITIES_API` e uma API key com o escopo `activities:write`. Só atividades pertencentes ao owner autenticado podem ser eliminadas.

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.**

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string (uuid) | **Sim** | UUID da atividade. |

**Parâmetros de query.** Nenhum.

**Corpo da requisição.** Nenhum.

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `204` | No Content — atividade eliminada com sucesso. | — |
| `400` | `Param ID must be a valid UUID`. | `{ status, message }` |
| `401` | Erro de autenticação — `api-token` inválido ou em falta. | — |
| `403` | Funcionalidade indisponível na conta (feature flag `ACTIVITIES_API` em falta). | `{ status, message }` |
| `404` | `Activity not found`. | `{ status, message }` |

**Exemplo — requisição**
```bash
curl -X DELETE "https://api.clint.digital/v2/activities/8feade82-d77b-4e8b-9d35-fd43e972b5c8" \
  -H "api-token: SUA_API_KEY"
```

**Exemplo — resposta (`204`)**
```
HTTP/1.1 204 No Content
```

**Notas (inteligência LL Mídia).**
- É um **soft-delete**: a atividade deixa de aparecer nas leituras, mas o registo é apenas marcado como eliminado internamente.
- `204` não devolve corpo. Não tente fazer parse de JSON na resposta bem-sucedida.
- Devido à consistência eventual da lista, a atividade eliminada pode ainda surgir por breves segundos em `GET /v2/activities`; confirme via `GET /v2/activities/{id}` (que passará a devolver `404`).

---

## `POST` `/v2/activities/{id}/complete` — Concluir/reabrir atividade

**O que faz.** Marca uma atividade como concluída, ou reabre uma anteriormente concluída. O corpo é opcional; `completed` assume `true` por defeito. **Reabrir** (`completed: false`) uma atividade **não-custom** (criada automaticamente a partir de um template de fase) que já não está na fase actual do negócio é rejeitado com `400` — o serviço interno, nesse caso, eliminaria (soft-delete) a atividade em vez de a reabrir. Conclusões feitas por este endpoint **não carregam atribuição de utilizador** (`completed_by` permanece `null`). Requer a feature flag `ACTIVITIES_API` e uma API key com o escopo `activities:write`.

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.**

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string (uuid) | **Sim** | UUID da atividade. |

**Parâmetros de query.** Nenhum.

**Corpo da requisição.** `application/json` (opcional).

| Nome | Tipo | Obrigatório | Default | Descrição |
|---|---|---|---|---|
| `completed` | boolean | Não | `true` | `true` para concluir a atividade, `false` para a reabrir. |

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `200` | Objeto da atividade actualizada. | `{ status, data: Activity }` |
| `400` | `Cannot reopen a stage-template activity from a previous stage`. | `{ status, message }` |
| `401` | Erro de autenticação — `api-token` inválido ou em falta. | — |
| `403` | Funcionalidade indisponível na conta (feature flag `ACTIVITIES_API` em falta). | `{ status, message }` |
| `404` | `Activity not found`. | `{ status, message }` |

**Exemplo — requisição**
```bash
curl -X POST "https://api.clint.digital/v2/activities/8feade82-d77b-4e8b-9d35-fd43e972b5c8/complete" \
  -H "api-token: SUA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "completed": true }'
```

**Exemplo — resposta (`200`)**
```json
{
  "status": 200,
  "data": {
    "id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8",
    "title": "Call the customer back",
    "type": "CALL",
    "content": "Discuss the renewal proposal.",
    "template_id": null,
    "completed": true,
    "completed_at": "2024-01-20T15:00:00.000Z",
    "completed_by": null,
    "due_at": "2024-01-20T14:00:00.000Z",
    "custom": true,
    "no_show": false,
    "stage_id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8",
    "created_at": "2020-01-01T14:15:00.000000+00:00",
    "updated_at": "2024-01-20T15:00:00.000Z",
    "deal": {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "status": "OPEN",
      "origin_id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8",
      "stage_id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8",
      "contact": {
        "id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
        "name": "Jane Doe"
      }
    }
  }
}
```

**Notas (inteligência LL Mídia).**
- Chamar sem corpo equivale a `completed: true` (concluir). Para reabrir, envie explicitamente `{ "completed": false }`.
- **Cuidado ao reabrir atividades de template.** Se a atividade tem `custom: false` (gerada por template de fase) e o negócio já avançou para outra fase, a reabertura devolve `400 Cannot reopen a stage-template activity from a previous stage`. Isto é uma salvaguarda: internamente, reabrir nessas condições apagaria a atividade em vez de a reabrir. Atividades `custom: true` não sofrem desta restrição.
- As conclusões feitas por API **não têm atribuição** — `completed_by` fica `null`. Se precisar de saber quem concluiu, esse dado não vem desta via.
- Na conclusão, `completed` passa a `true` e `completed_at` fica preenchido; ao reabrir, `completed` volta a `false`.

---

## Objetos relacionados

- [`Activity`](../03-modelo-de-dados.md#activity) — atividade de negócio (chamada, tarefa, reunião, agendamento, mensagem WhatsApp/Instagram), com título, tipo, conteúdo, vencimento, estado de conclusão e o negócio/contacto associados.
- [`PaginatedV2`](../03-modelo-de-dados.md#paginatedv2) — envelope de paginação v2 (chaves snake_case): `status`, `total_count`, `page`, `total_pages`, `has_next`, `has_previous`.
- [`DateTime`](../03-modelo-de-dados.md#datetime) — string ISO 8601 (date-time) usada em `created_at`.
- [`ID`](../03-modelo-de-dados.md#id) — string UUID usada em `id`, `stage_id` e nos identificadores aninhados do `deal`.
