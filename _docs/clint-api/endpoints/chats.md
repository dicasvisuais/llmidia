# Chats — Clint API

Recurso de **omnichannel** (v2) para consultar e gerir conversas (chats) de WhatsApp e Instagram — listar por contacto ou por conta de canal, obter um chat individual e actualizar o atendente/estado.

> Fonte: OpenAPI oficial da Clint (`@clint-api/v1.0`) · Base `https://api.clint.digital` · Auth: header `api-token` · Documento interno LL Mídia.

## Visão geral

A categoria **Chats** expõe endpoints da **v2** (mensageria/omnichannel) para trabalhar com conversas. Um `Chat` representa o fio de conversa entre a conta e um contacto num determinado canal, identificado por `id` (UUID), com referência ao `contact_id`, ao atendente (`user_id`), à conta de canal (`channel_account_id`) e, quando aplicável, à equipa (`team_id`). Traz ainda estado (`status`), flags de leitura/resposta (`seen`, `unread`, `replied`), contador de mensagens por ver (`unseen_count`) e um conjunto de carimbos temporais úteis para SLA e janelas de mensagem (`last_message_at`, `last_response_at`, `first_response_at`, `first_customer_message_at`, `close_window_at`, `closed_at`, `last_status_at`).

O `tag_description` oficial desta categoria diz literalmente: *"Operations related to chats via OpenSearch — WHATSAPP_OFFICIAL, WHATSAPP and INSTAGRAM (v2)"*. Ou seja: as consultas são servidas por um índice **OpenSearch** (motor de pesquisa/indexação, por oposição a uma leitura directa da base transaccional) e cobrem os canais `WHATSAPP_OFFICIAL`, `WHATSAPP` e `INSTAGRAM`. É um texto descritivo — **não** é um guia passo-a-passo do tipo SMS/VOICE/Webhooks, pelo que não há walkthrough a reproduzir aqui. Na prática, por assentarem em índice de pesquisa, as listagens podem ter latência de indexação (um chat acabado de criar/actualizar pode não aparecer instantaneamente nos filtros por data).

Todos os endpoints seguem as convenções da **v2**: autenticação por header `api-token`, respostas em JSON e, nas listagens, o envelope de paginação **`PaginatedV2`** com chaves em `snake_case` (`total_count`, `total_pages`, `has_next`, `has_previous`) — distinto do envelope `Paginated` da v1. Atenção: nesta fatia, o parâmetro `limit` tem **máximo `200`** (e default `200`), ao contrário do tecto de `1000` de outros endpoints v1. As datas são ISO-8601 (schema `DateTime`) e os IDs são UUID (schema `ID`).

Esta fatia disponibiliza **4 operações**: três de leitura (`GET`) — listar chats por contacto, listar chats por conta de canal e obter um chat por `id` — e uma de escrita (`POST`) para actualizar o atendente e/ou o estado de um chat. Todas as operações só devolvem/actualizam chats pertencentes ao dono (owner) autenticado pelo `api-token`.

## Índice de endpoints

| Método | Caminho | O que faz |
|---|---|---|
| GET | `/v2/chats/contact/{contactId}` | Lista, de forma paginada, os chats de um contacto (WhatsApp/Instagram), ordenados por `last_message_at` desc. |
| GET | `/v2/chats/channel-account/{channelAccountId}` | Lista, de forma paginada, os chats de uma conta de canal, ordenados por `last_message_at` desc. |
| GET | `/v2/chats/{id}` | Obtém um chat único pelo seu `id`. |
| POST | `/v2/chats/{id}` | Actualiza o atendente (`user_id`) e/ou o `status` de um chat. |

---

## `GET` `/v2/chats/contact/{contactId}` — List chats by contact

**O que faz.** Devolve uma lista paginada de chats de um contacto específico, através dos canais `WHATSAPP_OFFICIAL`, `WHATSAPP` e `INSTAGRAM`. Os resultados vêm ordenados por `last_message_at` de forma descendente (mais recente primeiro).

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.**

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `contactId` | string (uuid) | Sim | UUID do contacto cujos chats se pretendem listar. |

**Parâmetros de query.**

| Nome | Tipo | Obrigatório | Default | Descrição |
|---|---|---|---|---|
| `limit` | integer | Não | `200` | Número máximo de linhas devolvidas. Mínimo `1`, máximo `200`. |
| `offset` | integer | Não | `0` | Número de linhas ignoradas no início do resultado. Mínimo `0`. |
| `page` | integer | Não | `1` | Selecciona a página do resultado. Mínimo `1`. |
| `last_message_at_start` | string (date-time) | Não | — | Filtra chats cujo `last_message_at` seja **maior ou igual** a esta data (ISO-8601; um `YYYY-MM-DD` isolado é tratado como o início do dia). |
| `last_message_at_end` | string (date-time) | Não | — | Filtra chats cujo `last_message_at` seja **menor ou igual** a esta data (ISO-8601). Um `YYYY-MM-DD` isolado normaliza para o início do dia, logo exclui esse dia de calendário — passe um timestamp completo (ex.: `...T23:59:59Z`) para o incluir. |
| `last_response_at_start` | string (date-time) | Não | — | Filtra chats cujo `last_response_at` seja **maior ou igual** a esta data (ISO-8601; um `YYYY-MM-DD` isolado é tratado como o início do dia). |
| `last_response_at_end` | string (date-time) | Não | — | Filtra chats cujo `last_response_at` seja **menor ou igual** a esta data (ISO-8601). Um `YYYY-MM-DD` isolado normaliza para o início do dia, logo exclui esse dia de calendário — passe um timestamp completo (ex.: `...T23:59:59Z`) para o incluir. |

**Cabeçalhos específicos.**

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `api-token` | string | Sim | API Token da conta. |

**Corpo da requisição.** Nenhum.

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `200` | A paginated list of chats | `PaginatedV2` + `data: Chat[]` |
| `400` | Invalid UUID format | `{ status: integer, message: string }` (ex.: `"Param contact_id must be a valid UUID"`) |
| `401` | Authentication error - invalid or missing api-token | — |

O corpo de `200` combina (`allOf`) o envelope [`PaginatedV2`](../03-modelo-de-dados.md#paginatedv2) com uma propriedade `data`, que é um array de [`Chat`](../03-modelo-de-dados.md#chat). Campos do envelope:

| Campo | Tipo | Descrição |
|---|---|---|
| `status` | integer | Código de estado da resposta. |
| `total_count` | integer | Total de itens de acordo com os filtros actuais. |
| `page` | integer | Página actual. |
| `total_pages` | integer | Total de páginas de acordo com os filtros actuais. |
| `has_next` | boolean | Indica se existe página seguinte. |
| `has_previous` | boolean | Indica se existe página anterior. |
| `data` | array de `Chat` | Lista de chats desta página. |

**Exemplo — requisição**
```bash
curl -X GET "https://api.clint.digital/v2/chats/contact/550e8400-e29b-41d4-a716-446655440000?limit=200&page=1&last_message_at_start=2024-01-01&last_message_at_end=2024-01-31T23:59:59Z" \
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
      "contact_id": "550e8400-e29b-41d4-a716-446655440000",
      "user_id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
      "status": "OPEN",
      "seen": false,
      "unread": true,
      "replied": false,
      "unseen_count": 3,
      "last_message_at": "2024-01-15T10:00:00.000000+00:00",
      "last_response_at": "2024-01-15T09:00:00.000Z",
      "last_status_at": "2024-01-15T10:30:00.000Z",
      "first_response_at": "2024-01-14T08:00:00.000Z",
      "channel_account_id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8",
      "team_id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8",
      "closed_at": null,
      "first_customer_message_at": null,
      "close_window_at": "2024-01-16T10:30:00.000Z"
    }
  ]
}
```

**Notas (inteligência LL Mídia).**
- O `contactId` no caminho tem de ser um **UUID** válido; um valor mal formado devolve `400` com a mensagem `"Param contact_id must be a valid UUID"`.
- Como a listagem já vem ordenada por `last_message_at` desc, o primeiro elemento é o chat mais recentemente activo do contacto — útil para saltar directamente para a última conversa.
- Os filtros de data trabalham em pares abertos: use `*_start` e/ou `*_end` conforme a janela pretendida. Lembre-se do detalhe de normalização do `*_end`: um `YYYY-MM-DD` puro fecha o intervalo no **início** desse dia (exclui-o) — para incluir o dia inteiro, passe `...T23:59:59Z`.
- Para percorrer todos os chats, itere por `page` até `has_next` ser `false`; dimensione o loop com `total_count`/`total_pages`. O tecto de `limit` é **`200`** nesta v2.
- Combine `offset` e `page` com cuidado; em geral, escolha **uma** estratégia de paginação e mantenha-a.

---

## `GET` `/v2/chats/channel-account/{channelAccountId}` — List chats by channel account

**O que faz.** Devolve uma lista paginada de chats de uma conta de canal específica (`channel account` — a instância ligada de WhatsApp/Instagram). Os resultados vêm ordenados por `last_message_at` de forma descendente.

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.**

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `channelAccountId` | string (uuid) | Sim | UUID da conta de canal cujos chats se pretendem listar. |

**Parâmetros de query.**

| Nome | Tipo | Obrigatório | Default | Descrição |
|---|---|---|---|---|
| `limit` | integer | Não | `200` | Número máximo de linhas devolvidas. Mínimo `1`, máximo `200`. |
| `offset` | integer | Não | `0` | Número de linhas ignoradas no início do resultado. Mínimo `0`. |
| `page` | integer | Não | `1` | Selecciona a página do resultado. Mínimo `1`. |
| `last_message_at_start` | string (date-time) | Não | — | Filtra chats cujo `last_message_at` seja **maior ou igual** a esta data (ISO-8601; um `YYYY-MM-DD` isolado é tratado como o início do dia). |
| `last_message_at_end` | string (date-time) | Não | — | Filtra chats cujo `last_message_at` seja **menor ou igual** a esta data (ISO-8601). Um `YYYY-MM-DD` isolado normaliza para o início do dia, logo exclui esse dia de calendário — passe um timestamp completo (ex.: `...T23:59:59Z`) para o incluir. |
| `last_response_at_start` | string (date-time) | Não | — | Filtra chats cujo `last_response_at` seja **maior ou igual** a esta data (ISO-8601; um `YYYY-MM-DD` isolado é tratado como o início do dia). |
| `last_response_at_end` | string (date-time) | Não | — | Filtra chats cujo `last_response_at` seja **menor ou igual** a esta data (ISO-8601). Um `YYYY-MM-DD` isolado normaliza para o início do dia, logo exclui esse dia de calendário — passe um timestamp completo (ex.: `...T23:59:59Z`) para o incluir. |

**Cabeçalhos específicos.**

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `api-token` | string | Sim | API Token da conta. |

**Corpo da requisição.** Nenhum.

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `200` | A paginated list of chats | `PaginatedV2` + `data: Chat[]` |
| `400` | Invalid UUID format | `{ status: integer, message: string }` (ex.: `"Param channel_account_id must be a valid UUID"`) |
| `401` | Authentication error - invalid or missing api-token | — |

O corpo de `200` tem a mesma estrutura da listagem por contacto: envelope [`PaginatedV2`](../03-modelo-de-dados.md#paginatedv2) (`status`, `total_count`, `page`, `total_pages`, `has_next`, `has_previous`) combinado (`allOf`) com `data`, um array de [`Chat`](../03-modelo-de-dados.md#chat).

**Exemplo — requisição**
```bash
curl -X GET "https://api.clint.digital/v2/chats/channel-account/550e8400-e29b-41d4-a716-446655440000?limit=200&page=1" \
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
      "contact_id": "550e8400-e29b-41d4-a716-446655440000",
      "user_id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
      "status": "OPEN",
      "seen": false,
      "unread": true,
      "replied": false,
      "unseen_count": 3,
      "last_message_at": "2024-01-15T10:00:00.000000+00:00",
      "last_response_at": "2024-01-15T09:00:00.000Z",
      "last_status_at": "2024-01-15T10:30:00.000Z",
      "first_response_at": "2024-01-14T08:00:00.000Z",
      "channel_account_id": "550e8400-e29b-41d4-a716-446655440000",
      "team_id": null,
      "closed_at": null,
      "first_customer_message_at": null,
      "close_window_at": "2024-01-16T10:30:00.000Z"
    }
  ]
}
```

**Notas (inteligência LL Mídia).**
- O `channelAccountId` no caminho tem de ser um **UUID** válido; um valor mal formado devolve `400` com a mensagem `"Param channel_account_id must be a valid UUID"`.
- Este endpoint é a via natural para operações de caixa de entrada por número/conta: liste todos os chats de uma linha de WhatsApp (ou perfil de Instagram) específica e filtre por data para relatórios de volume/actividade.
- Os filtros de data e a paginação comportam-se exactamente como na listagem por contacto (mesmo tecto de `limit = 200`, mesmo detalhe de normalização do `*_end`).
- Combine com [List chats by contact](#get-v2chatscontactcontactid--list-chats-by-contact) quando precisar de cruzar a perspectiva "por conta de canal" com a perspectiva "por contacto".

---

## `GET` `/v2/chats/{id}` — Get chat

**O que faz.** Devolve um único chat pelo seu identificador. Só retorna chats que pertençam ao dono (owner) autenticado.

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.**

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string (uuid) | Sim | UUID do chat a obter. |

**Parâmetros de query.** Nenhum.

**Cabeçalhos específicos.**

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `api-token` | string | Sim | API Token da conta. |

**Corpo da requisição.** Nenhum.

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `200` | A chat object | `{ status: integer, data: Chat }` |
| `400` | Invalid UUID format | `{ status: integer, message: string }` (ex.: `"Param ID must be a valid UUID"`) |
| `401` | Authentication error - invalid or missing api-token | — |
| `404` | Chat not found | `{ status: integer, message: string }` (ex.: `"Chat not found"`) |

O corpo de `200` combina (`allOf`) um objecto com `status` e um objecto com `data`, sendo `data` um único [`Chat`](../03-modelo-de-dados.md#chat):

| Campo | Tipo | Descrição |
|---|---|---|
| `status` | integer | Código de estado da resposta. |
| `data` | `Chat` | O objecto do chat pedido. |

Campos do objecto [`Chat`](../03-modelo-de-dados.md#chat):

| Campo | Tipo | Descrição |
|---|---|---|
| `id` | string (uuid) | Identificador único do chat. |
| `created_at` | string (date-time) | Data/hora de criação do chat (ISO-8601). |
| `contact_id` | string (uuid) | UUID do contacto associado ao chat. |
| `user_id` | string (uuid) | UUID do utilizador (atendente) atribuído ao chat. |
| `status` | string (enum) | Estado do chat. Valores: `OPEN`, `CLOSED`, `WAITING`, `SNOOZED`, `REOPENED`. |
| `seen` | boolean | Indica se o chat foi visto. |
| `unread` | boolean | Indica se o chat tem mensagens por ler. |
| `replied` | boolean | Indica se o chat já foi respondido. |
| `unseen_count` | integer | Número de mensagens por ver no chat. |
| `last_message_at` | string (date-time) | Data/hora da última mensagem (ISO-8601). |
| `last_response_at` | string (date-time), nullable | Data/hora da última resposta. |
| `last_status_at` | string (date-time), nullable | Data/hora da última alteração de estado. |
| `first_response_at` | string (date-time), nullable | Data/hora da primeira resposta. |
| `channel_account_id` | string (uuid) | UUID da conta de canal do chat. |
| `team_id` | string (uuid), nullable | UUID da equipa associada ao chat. |
| `closed_at` | string (date-time), nullable | Data/hora em que o chat foi fechado. |
| `first_customer_message_at` | string (date-time), nullable | Data/hora da primeira mensagem do cliente no chat. |
| `close_window_at` | string (date-time), nullable | Data/hora em que a janela de mensagem do WhatsApp fecha (24h após a última mensagem do cliente). |

**Exemplo — requisição**
```bash
curl -X GET "https://api.clint.digital/v2/chats/8feade82-d77b-4e8b-9d35-fd43e972b5c8" \
  -H "api-token: SUA_API_KEY"
```

**Exemplo — resposta (`200`)**
```json
{
  "status": 200,
  "data": {
    "id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8",
    "created_at": "2020-01-01T14:15:00.000000+00:00",
    "contact_id": "550e8400-e29b-41d4-a716-446655440000",
    "user_id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
    "status": "OPEN",
    "seen": false,
    "unread": true,
    "replied": false,
    "unseen_count": 3,
    "last_message_at": "2024-01-15T10:00:00.000000+00:00",
    "last_response_at": "2024-01-15T09:00:00.000Z",
    "last_status_at": "2024-01-15T10:30:00.000Z",
    "first_response_at": "2024-01-14T08:00:00.000Z",
    "channel_account_id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8",
    "team_id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8",
    "closed_at": null,
    "first_customer_message_at": null,
    "close_window_at": "2024-01-16T10:30:00.000Z"
  }
}
```

**Notas (inteligência LL Mídia).**
- O `id` no caminho tem de ser um **UUID** válido; um valor mal formado devolve `400` (`"Param ID must be a valid UUID"`), e um UUID válido mas inexistente (ou de outro dono) devolve `404` (`"Chat not found"`).
- Ao contrário das listagens, esta resposta **não** usa `PaginatedV2`: o corpo traz apenas `status` e `data` (um único `Chat`).
- `close_window_at` é o carimbo-chave para o WhatsApp: marca o fecho da janela de 24h de mensagens de sessão. Depois desse instante, para reabrir a conversa normalmente é preciso um template aprovado (fora do âmbito destes endpoints).
- Note a diferença de enums: o campo `status` do `Chat` pode assumir cinco valores (`OPEN`, `CLOSED`, `WAITING`, `SNOOZED`, `REOPENED`), mas a operação de actualização só aceita três (`OPEN`, `SNOOZED`, `CLOSED`) — ver a nota do endpoint seguinte.

---

## `POST` `/v2/chats/{id}` — Update chat

**O que faz.** Actualiza o atendente atribuído (`user_id`) e/ou o `status` de um chat. Regras da spec:
- Atribuir um atendente **força** o `status` para `OPEN`, **excepto** se um `status` explícito for enviado no mesmo pedido (nesse caso, o `status` explícito prevalece).
- Enviar `user_id: null` **desatribui** o atendente sem alterar o estado.
- Definir `status` para `CLOSED` regista `closed_at`; reabrir **não** limpa esse carimbo.
- É obrigatório enviar **pelo menos um** de `user_id` ou `status`.
- Só podem ser actualizados chats que pertençam ao dono (owner) autenticado.

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.**

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string (uuid) | Sim | UUID do chat a actualizar. |

**Parâmetros de query.** Nenhum.

**Cabeçalhos específicos.**

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `api-token` | string | Sim | API Token da conta. |

**Corpo da requisição.** `application/json` (obrigatório; `minProperties: 1` — pelo menos um campo).

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `user_id` | string (uuid), nullable | Não* | UUID do utilizador (atendente) a atribuir. Tem de ser um utilizador da conta. Envie `null` para desatribuir. |
| `status` | string (enum) | Não* | Novo estado do chat. Valores: `OPEN`, `SNOOZED`, `CLOSED`. `CLOSED` regista `closed_at`; reabrir não o limpa. |

\* Individualmente opcionais, mas é obrigatório enviar **pelo menos um** dos dois (senão devolve `400`).

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `200` | The updated chat object | `{ status: integer, data: Chat }` |
| `400` | At least one of user_id or status is required | `{ status: integer, message: string }` (ex.: `"At least one of user_id or status is required"`) |
| `401` | Authentication error - invalid or missing api-token | — |
| `404` | Chat not found | `{ status: integer, message: string }` (ex.: `"Chat not found"`) |

O corpo de `200` combina (`allOf`) um objecto com `status` e um objecto com `data`, sendo `data` o [`Chat`](../03-modelo-de-dados.md#chat) já actualizado (mesma estrutura de campos descrita em [Get chat](#get-v2chatsid--get-chat)).

**Exemplo — requisição**
```bash
curl -X POST "https://api.clint.digital/v2/chats/8feade82-d77b-4e8b-9d35-fd43e972b5c8" \
  -H "api-token: SUA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "user_id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
    "status": "CLOSED"
  }'
```

**Exemplo — resposta (`200`)**
```json
{
  "status": 200,
  "data": {
    "id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8",
    "created_at": "2020-01-01T14:15:00.000000+00:00",
    "contact_id": "550e8400-e29b-41d4-a716-446655440000",
    "user_id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
    "status": "CLOSED",
    "seen": true,
    "unread": false,
    "replied": true,
    "unseen_count": 0,
    "last_message_at": "2024-01-15T10:00:00.000000+00:00",
    "last_response_at": "2024-01-15T09:00:00.000Z",
    "last_status_at": "2024-01-15T10:30:00.000Z",
    "first_response_at": "2024-01-14T08:00:00.000Z",
    "channel_account_id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8",
    "team_id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8",
    "closed_at": "2024-01-15T10:30:00.000Z",
    "first_customer_message_at": null,
    "close_window_at": "2024-01-16T10:30:00.000Z"
  }
}
```

**Notas (inteligência LL Mídia).**
- Apesar de ser uma actualização, o método é **`POST`** (não `PATCH`/`PUT`) — envie sempre `Content-Type: application/json`.
- Cuidado com a interacção `user_id` × `status`: atribuir um atendente sozinho leva o chat a `OPEN`. Se quer atribuir **e** manter/definir outro estado (ex.: `SNOOZED`), envie ambos no mesmo pedido — o `status` explícito ganha.
- Para desatribuir sem mexer no estado, envie exactamente `{"user_id": null}`.
- O corpo vazio `{}` é inválido (`minProperties: 1`) e devolve `400` com `"At least one of user_id or status is required"`.
- O enum de escrita (`OPEN`, `SNOOZED`, `CLOSED`) é **mais restrito** que o de leitura do `Chat` (`OPEN`, `CLOSED`, `WAITING`, `SNOOZED`, `REOPENED`): estados como `WAITING` ou `REOPENED` são geridos pelo sistema, não definíveis directamente por esta chamada.
- Fechar (`CLOSED`) carimba `closed_at`; se reabrir depois (ex.: `OPEN`), o `closed_at` **mantém-se** — não é limpo. Use `last_status_at`/`closed_at` para reconstruir a linha temporal do atendimento.

---

## Objetos relacionados

- [`Chat`](../03-modelo-de-dados.md#chat) — conversa omnichannel (WhatsApp/Instagram): `id`, `created_at`, `contact_id`, `user_id`, `status`, `seen`, `unread`, `replied`, `unseen_count`, `last_message_at`, `last_response_at`, `last_status_at`, `first_response_at`, `channel_account_id`, `team_id`, `closed_at`, `first_customer_message_at`, `close_window_at`.
- [`PaginatedV2`](../03-modelo-de-dados.md#paginatedv2) — envelope de paginação v2 em `snake_case` (`status`, `total_count`, `page`, `total_pages`, `has_next`, `has_previous`).
- [`ID`](../03-modelo-de-dados.md#id) — identificador UUID (string) usado em `id`, `contact_id`, `user_id`, `channel_account_id` e nos parâmetros de caminho.
- [`DateTime`](../03-modelo-de-dados.md#datetime) — data/hora em ISO-8601, usado em `created_at`, `last_message_at` e nos filtros de data das listagens.
