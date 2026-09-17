# Channel Accounts — Clint API

Contas de canal (WhatsApp e Instagram) ligadas à sua conta Clint, usadas como origem/destino da mensageria omnichannel (v2).

> Fonte: OpenAPI oficial da Clint (`@clint-api/v1.0`) · Base `https://api.clint.digital` · Auth: header `api-token` · Documento interno LL Mídia.

## Visão geral

As **Channel Accounts** representam as contas de canal ligadas à sua conta Clint na camada omnichannel (**v2**). Cada conta corresponde a um número/perfil de um canal de mensagens e é identificada por um `type`. Segundo a spec, os tipos suportados são três: `WHATSAPP_OFFICIAL` (WhatsApp Cloud API), `WHATSAPP` (WhatsApp Web/ZAPI) e `INSTAGRAM` (Instagram Direct). Esta categoria é apenas de **leitura**: expõe duas operações `GET` — uma lista paginada e a obtenção de uma conta por `id`.

O guia da própria spec (`tag_description`) resume o âmbito da categoria: *"Operations related to channel accounts — WHATSAPP_OFFICIAL, WHATSAPP and INSTAGRAM (v2)"*. Ou seja, estas operações servem para descobrir que contas de canal existem, qual o seu estado de ligação e a que equipa (`team_id`) pertencem, antes de as usar noutras operações de mensageria. Note-se, a partir da descrição do campo `type`, que apenas `WHATSAPP_OFFICIAL` (Cloud API) suporta o **envio** de mensagens através desta API; os restantes tipos existem para leitura/contexto.

Ambos os endpoints vivem em `/v2`. A listagem devolve um envelope paginado no formato **`PaginatedV2`** (chaves em snake_case: `total_count`, `page`, `total_pages`, `has_next`, `has_previous`), com o array de contas em `data`. Contas **soft-deleted** (apagadas de forma lógica) são **sempre excluídas** dos resultados. É possível restringir a listagem a um único tipo através do parâmetro de query opcional `type`.

Para autenticação, todas as chamadas exigem o header obrigatório `api-token`. A ausência ou invalidez deste header devolve `401`. Valores inválidos no `type` (listagem) ou num `id` que não seja UUID (detalhe) devolvem `400`, e um `id` inexistente devolve `404`.

## Índice de endpoints

| Método | Caminho | O que faz |
|---|---|---|
| GET | `/v2/channel-accounts` | Lista paginada de contas de canal (opcionalmente filtrada por `type`). |
| GET | `/v2/channel-accounts/{id}` | Obtém uma conta de canal específica pelo seu `id` (UUID). |

---

## `GET` `/v2/channel-accounts` — List channel accounts

**O que faz.** Devolve uma lista paginada de contas de canal. Por omissão, devolve todos os tipos de canal suportados (`WHATSAPP_OFFICIAL`, `WHATSAPP`, `INSTAGRAM`); contas soft-deleted são sempre excluídas. Use o parâmetro de query opcional `type` para restringir o resultado a um único tipo de canal.

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.** Nenhum.

**Parâmetros de query.**

| Nome | Tipo | Obrigatório | Default | Descrição |
|---|---|---|---|---|
| `limit` | integer | Não | `100` | Número máximo de linhas devolvidas. Mínimo `1`, máximo `100`. |
| `offset` | integer | Não | `0` | Número de linhas ignoradas do resultado. Mínimo `0`. |
| `page` | integer | Não | `1` | Seleciona a página do resultado. Mínimo `1`. |
| `type` | string (enum) | Não | — | Filtra as contas por tipo. Quando omitido, devolve todos os tipos suportados. Valores: `WHATSAPP_OFFICIAL`, `WHATSAPP`, `INSTAGRAM`. |

**Corpo da requisição.** Nenhum.

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `200` | Lista paginada de contas de canal. | `PaginatedV2` + `data: ChannelAccount[]` |
| `400` | Valor de `type` inválido. | `{ status, message }` (ex.: `"Param type must be one of: WHATSAPP_OFFICIAL, WHATSAPP, INSTAGRAM"`) |
| `401` | Erro de autenticação — `api-token` inválido ou em falta. | — |

**Exemplo — requisição**
```bash
curl -X GET "https://api.clint.digital/v2/channel-accounts?type=WHATSAPP_OFFICIAL&limit=100&page=1" \
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
      "created_at": "2020-01-01T14:15:00.000000+00:00",
      "name": "WhatsApp Business",
      "type": "WHATSAPP_OFFICIAL",
      "status": "CONNECTED",
      "avatar": "https://example.com/avatar.jpg",
      "identifier": "5548999999999",
      "team_id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8"
    }
  ]
}
```

**Exemplo — resposta (`400`)**
```json
{
  "status": 400,
  "message": "Param type must be one of: WHATSAPP_OFFICIAL, WHATSAPP, INSTAGRAM"
}
```

**Notas (inteligência LL Mídia).**
- O `limit` desta categoria é mais restrito do que noutras zonas da API: máximo **100** (default 100), não 200/1000. Para percorrer contas em lote, avance por `page` ou `offset`.
- A paginação usa o envelope **`PaginatedV2`** (snake_case). Para saber se há mais páginas, use `has_next`/`has_previous` em vez de calcular manualmente com `total_count`.
- Contas soft-deleted nunca aparecem — não é preciso filtrá-las do lado do cliente.
- Se precisar apenas de um tipo (por exemplo, apurar quais números de WhatsApp Cloud API podem enviar mensagens), filtre já na origem com `type=WHATSAPP_OFFICIAL` para reduzir payload.
- `offset` e `page` coexistem na spec; use uma estratégia de paginação de cada vez para evitar saltos inesperados de resultados.

---

## `GET` `/v2/channel-accounts/{id}` — Get channel account

**O que faz.** Devolve uma única conta de canal pelo seu `id`. Suporta os tipos `WHATSAPP_OFFICIAL`, `WHATSAPP` e `INSTAGRAM`.

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.**

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string (UUID) | Sim | Identificador único da conta de canal (UUID). |

**Parâmetros de query.** Nenhum.

**Corpo da requisição.** Nenhum.

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `200` | Objeto da conta de canal. | `{ status, data: ChannelAccount }` |
| `400` | Formato de UUID inválido. | `{ status, message }` (ex.: `"Param ID must be a valid UUID"`) |
| `401` | Erro de autenticação — `api-token` inválido ou em falta. | — |
| `404` | Conta de canal não encontrada. | `{ status, message }` (ex.: `"Channel account not found"`) |

**Exemplo — requisição**
```bash
curl -X GET "https://api.clint.digital/v2/channel-accounts/8feade82-d77b-4e8b-9d35-fd43e972b5c8" \
  -H "api-token: SUA_API_KEY"
```

**Exemplo — resposta (`200`)**
```json
{
  "status": 200,
  "data": {
    "id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8",
    "created_at": "2020-01-01T14:15:00.000000+00:00",
    "name": "WhatsApp Business",
    "type": "WHATSAPP_OFFICIAL",
    "status": "CONNECTED",
    "avatar": "https://example.com/avatar.jpg",
    "identifier": "5548999999999",
    "team_id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8"
  }
}
```

**Exemplo — resposta (`404`)**
```json
{
  "status": 404,
  "message": "Channel account not found"
}
```

**Notas (inteligência LL Mídia).**
- Valide o `id` do lado do cliente antes de chamar: um valor que não seja UUID válido devolve `400` (`"Param ID must be a valid UUID"`), distinto do `404` (UUID válido mas inexistente).
- A resposta de detalhe **não** usa `PaginatedV2`: é um envelope simples com `status` e o objeto em `data`.
- Combine com a listagem para fluxos de sincronização: liste com `GET /v2/channel-accounts` para obter os `id`, e use este endpoint para reconfirmar o `status` (`CONNECTED` / `DISCONNECTED` / `CANCELLED`) de uma conta antes de disparar operações de mensageria.

### Campos do objeto `ChannelAccount`

| Nome | Tipo | Nullable | Descrição |
|---|---|---|---|
| `id` | string (UUID) | Não | Identificador único da conta de canal. |
| `created_at` | string (date-time, ISO-8601) | Não | Data/hora de criação da conta. |
| `name` | string | Não | Nome da conta (ex.: `"WhatsApp Business"`). |
| `type` | string (enum) | Não | Tipo de canal. `WHATSAPP_OFFICIAL` = Cloud API (único tipo que suporta envio através desta API). `WHATSAPP` = WhatsApp Web/ZAPI. `INSTAGRAM` = Instagram Direct. |
| `status` | string (enum) | Não | Estado da ligação. Valores: `CONNECTED`, `DISCONNECTED`, `CANCELLED`. |
| `avatar` | string | Sim | URL do avatar da conta (pode ser `null`). |
| `identifier` | string | Sim | Identificador do canal (ex.: número `"5548999999999"`); pode ser `null`. |
| `team_id` | string (UUID) | Sim | Equipa a que a conta está associada (pode ser `null`). |

## Objetos relacionados

- [`ChannelAccount`](../03-modelo-de-dados.md#channelaccount) — Conta de canal (WhatsApp Official/WhatsApp/Instagram) com estado de ligação e equipa.
- [`PaginatedV2`](../03-modelo-de-dados.md#paginatedv2) — Envelope de paginação v2 (snake_case) usado na listagem.
- [`ID`](../03-modelo-de-dados.md#id) — Identificador UUID (string).
- [`DateTime`](../03-modelo-de-dados.md#datetime) — Data/hora em formato ISO-8601.
