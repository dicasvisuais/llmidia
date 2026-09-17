# Message Templates — Clint API

Consulta dos templates de mensagem do WhatsApp Official (v2) ligados a uma conta de canal, tal como aprovados na Meta.

> Fonte: OpenAPI oficial da Clint (`@clint-api/v1.0`) · Base `https://api.clint.digital` · Auth: header `api-token` · Documento interno LL Mídia.

## Visão geral

Os **Message Templates** são os modelos de mensagem do **WhatsApp Official** (templates HSM/WABA) que ficam registados na conta e são submetidos à Meta para aprovação. Esta categoria da API (v2) é **apenas de leitura**: permite **listar** os templates ligados a uma conta de canal WhatsApp Official e **obter** um template específico pelo seu `id`. A criação, edição e submissão de templates continua a fazer-se do lado da Meta/WABA — aqui só se consultam.

Cada template traz o seu `external_id` (o ID atribuído pela Meta), o `name`, o `status` de aprovação (`APPROVED`, `PENDING` ou `REJECTED`), o `language`, a `category` (`MARKETING`, `UTILITY` ou `AUTHENTICATION`) e a lista de `components` (`HEADER`, `BODY`, `FOOTER`, `BUTTONS`) que definem o corpo da mensagem. O campo `variables` (quando presente) descreve o mapeamento de variáveis por componente — útil para saber quais os placeholders (`{{1}}`, `{{2}}`, ...) a preencher no envio.

Ambos os endpoints exigem que o recurso pertença ao dono autenticado: na **listagem**, o parâmetro de query `channel_account_id` é **obrigatório** e tem de pertencer ao owner autenticado; na **obtenção por id**, a API valida que o template pertence a uma conta de canal WhatsApp Official desse mesmo owner. A listagem devolve um envelope paginado `PaginatedV2` (chaves em snake_case: `total_count`, `page`, `total_pages`, `has_next`, `has_previous`).

O `tag_description` oficial desta categoria é: *"Operations related to WhatsApp Official message templates (v2)"*. Não há, nesta fatia, um guia passo-a-passo adicional (ao contrário de SMS/VOICE/Webhooks).

## Índice de endpoints

| Método | Caminho | O que faz |
|---|---|---|
| GET | `/v2/message-templates` | Lista paginada de templates de uma conta de canal WhatsApp Official. |
| GET | `/v2/message-templates/{id}` | Obtém um template de mensagem específico pelo seu `id`. |

---

## `GET` `/v2/message-templates` — List message templates

**O que faz.** Devolve uma lista paginada de templates de mensagem ligados a uma conta de canal WhatsApp Official. O parâmetro de query `channel_account_id` é obrigatório e tem de pertencer ao owner autenticado.

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.** Nenhum.

**Parâmetros de query.**

| Nome | Tipo | Obrigatório | Default | Descrição |
|---|---|---|---|---|
| `channel_account_id` | `string` (`uuid`) | Sim | — | UUID da conta de canal (obrigatório). Tem de pertencer ao owner autenticado. |
| `limit` | `integer` | Não | `200` | Número máximo de linhas devolvidas (mínimo `1`, máximo `200`). |
| `offset` | `integer` | Não | `0` | Número de linhas ignoradas no resultado (mínimo `0`). |
| `page` | `integer` | Não | `1` | Seleciona a página do resultado (mínimo `1`). |

**Parâmetros de header.**

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `api-token` | `string` | Sim | API Token do owner autenticado. |

**Corpo da requisição.** Nenhum.

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `200` | Lista paginada de templates de mensagem. | `PaginatedV2` + `data: MessageTemplate[]` |
| `400` | `channel_account_id` em falta ou inválido. | `{ status, message }` — ex.: `"Query param channel_account_id is required"` |
| `401` | Erro de autenticação — `api-token` inválido ou em falta. | — |
| `404` | Conta de canal não encontrada. | `{ status, message }` — ex.: `"Channel account not found"` |

Campos do envelope de resposta (`PaginatedV2`):

| Nome | Tipo | Descrição |
|---|---|---|
| `status` | `integer` | Estado da resposta (ex.: `200`). |
| `total_count` | `integer` | Total de itens com base nos filtros atuais. |
| `page` | `integer` | Página atual. |
| `total_pages` | `integer` | Total de páginas com base nos filtros atuais. |
| `has_next` | `boolean` | Indica se existe página seguinte. |
| `has_previous` | `boolean` | Indica se existe página anterior. |
| `data` | `MessageTemplate[]` | Array de templates de mensagem (ver [`MessageTemplate`](#objetos-relacionados)). |

**Exemplo — requisição**
```bash
curl -X GET "https://api.clint.digital/v2/message-templates?channel_account_id=550e8400-e29b-41d4-a716-446655440000&limit=200&page=1" \
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
      "external_id": "123456789",
      "name": "welcome_message",
      "status": "APPROVED",
      "language": "pt_BR",
      "category": "MARKETING",
      "components": [
        {
          "type": "BODY",
          "text": "Hello {{1}}, welcome!",
          "format": "TEXT"
        }
      ],
      "variables": {
        "body": ["{{1}}"]
      }
    }
  ]
}
```

**Notas (inteligência LL Mídia).**
- `channel_account_id` é **obrigatório**: sem ele a API responde `400` com `"Query param channel_account_id is required"`. Obtenha o UUID da conta de canal WhatsApp Official através dos endpoints de canais/contas antes de listar templates.
- A conta de canal tem de pertencer ao **owner autenticado** (ao `api-token` usado). Um UUID válido mas de outra conta devolve `404` (`"Channel account not found"`).
- Paginação estilo v2 (`PaginatedV2`, snake_case): pode navegar por `page` ou por `offset`. O `limit` está **limitado a 200** (default 200) — mais restritivo do que os endpoints v1. Use `has_next`/`total_pages` para saber quando parar.
- Só devolve templates de contas **WhatsApp Official** (WABA). Contas de WhatsApp não-oficial não têm templates aqui.
- Filtre localmente por `status` (`APPROVED`) antes de tentar disparar um template — só templates aprovados na Meta são utilizáveis no envio.

---

## `GET` `/v2/message-templates/{id}` — Get message template

**O que faz.** Obtém um único template de mensagem pelo seu `id`. Valida que o template pertence a uma conta de canal WhatsApp Official do utilizador autenticado.

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.**

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | `string` (`uuid`) | Sim | UUID do template de mensagem. |

**Parâmetros de query.** Nenhum.

**Parâmetros de header.**

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `api-token` | `string` | Sim | API Token do owner autenticado. |

**Corpo da requisição.** Nenhum.

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `200` | Objeto do template de mensagem. | `{ status, data: MessageTemplate }` |
| `400` | Formato de UUID inválido. | `{ status, message }` — ex.: `"Param ID must be a valid UUID"` |
| `401` | Erro de autenticação — `api-token` inválido ou em falta. | — |
| `404` | Template de mensagem não encontrado. | `{ status, message }` — ex.: `"Message template not found"` |

Campos do envelope de resposta (`200`):

| Nome | Tipo | Descrição |
|---|---|---|
| `status` | `integer` | Estado da resposta (ex.: `200`). |
| `data` | `MessageTemplate` | O template de mensagem (ver [`MessageTemplate`](#objetos-relacionados)). |

**Exemplo — requisição**
```bash
curl -X GET "https://api.clint.digital/v2/message-templates/8feade82-d77b-4e8b-9d35-fd43e972b5c8" \
  -H "api-token: SUA_API_KEY"
```

**Exemplo — resposta (`200`)**
```json
{
  "status": 200,
  "data": {
    "id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8",
    "external_id": "123456789",
    "name": "welcome_message",
    "status": "APPROVED",
    "language": "pt_BR",
    "category": "MARKETING",
    "components": [
      {
        "type": "HEADER",
        "text": "Boas-vindas",
        "format": "TEXT"
      },
      {
        "type": "BODY",
        "text": "Hello {{1}}, welcome!",
        "format": "TEXT"
      },
      {
        "type": "FOOTER",
        "text": "Equipa LL Mídia",
        "format": "TEXT"
      }
    ],
    "variables": {
      "body": ["{{1}}"]
    }
  }
}
```

**Notas (inteligência LL Mídia).**
- O `id` no caminho é o **UUID interno da Clint**, não o `external_id` da Meta. Um valor que não seja um UUID válido devolve `400` (`"Param ID must be a valid UUID"`).
- A API valida a **posse**: um UUID bem formado mas que não pertença a uma conta WhatsApp Official do owner autenticado devolve `404` (`"Message template not found"`).
- A resposta **não** usa o envelope paginado — é `{ status, data }` com um único objeto em `data` (repare que o campo `status` da envelope, `integer`, é distinto do `status` do template, que é o enum de aprovação `APPROVED`/`PENDING`/`REJECTED`).
- Use este endpoint para inspecionar os `components` e o mapeamento de `variables` antes de compor um envio, garantindo que preenche todos os placeholders (`{{1}}`, `{{2}}`, ...) esperados.

---

## Objetos relacionados

- [`MessageTemplate`](../03-modelo-de-dados.md#messagetemplate) — Template de mensagem WhatsApp Official: `id`, `external_id` (ID na Meta), `name`, `status` (`APPROVED`/`PENDING`/`REJECTED`), `language`, `category` (`MARKETING`/`UTILITY`/`AUTHENTICATION`), `components` (`HEADER`/`BODY`/`FOOTER`/`BUTTONS`) e `variables` (mapeamento de variáveis por componente, anulável).
- [`PaginatedV2`](../03-modelo-de-dados.md#paginatedv2) — Envelope de paginação v2 (snake_case): `status`, `total_count`, `page`, `total_pages`, `has_next`, `has_previous`.
- [`ID`](../03-modelo-de-dados.md#id) — Identificador UUID (string) usado no campo `id` do template e no parâmetro de caminho `{id}`.
