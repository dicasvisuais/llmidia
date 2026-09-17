# Messages — Clint API

Leitura e envio de mensagens de conversas omnichannel (WhatsApp Oficial, WhatsApp e Instagram) do módulo de atendimento da Clint.

> Fonte: OpenAPI oficial da Clint (`@clint-api/v1.0`) · Base `https://api.clint.digital` · Auth: header `api-token` · Documento interno LL Mídia.

## Visão geral

As **Messages** são as mensagens individuais que compõem as conversas (chats) do módulo de atendimento omnichannel da Clint. Todas as operações desta categoria vivem em **`/v2`** e devolvem/recebem envelopes com chaves em `snake_case`. Cada mensagem é representada pelo schema `Message`, que inclui o conteúdo (`content`), o tipo de emissor (`type`: `USER`, `CUSTOMER`, `EVENT`, `NOTE`), o tipo de conteúdo (`content_type`: `TEXT`, `IMAGE`, `AUDIO`, `VIDEO`, `DOCUMENT`), as flags de entrega (`sent`, `delivered`, `seen`) e um `status` derivado dessas flags (`QUEUED`, `SENT`, `DELIVERED`, `READ`).

A fatia expõe **9 operações**, divididas em três blocos: **leitura** (listar mensagens de um chat, obter uma mensagem por ID), **nota interna** (criar uma anotação privada numa conversa) e **envio** (texto, imagem, vídeo, documento, áudio e template). A `tag_description` oficial desta categoria é explícita quanto ao alcance de cada bloco: *"Operations related to messages. Reads cover WHATSAPP_OFFICIAL, WHATSAPP and INSTAGRAM; sending endpoints remain WHATSAPP_OFFICIAL-only (v2)"*. Ou seja: **a leitura abrange os três canais** (`WHATSAPP_OFFICIAL`, `WHATSAPP` e `INSTAGRAM`), mas **o envio (endpoints `POST /v2/messages/*`) está restrito a `WHATSAPP_OFFICIAL`**. Tentar enviar por um canal de outro tipo devolve `400` com `Channel account type must be WHATSAPP_OFFICIAL`.

Nos endpoints de envio, o corpo aponta sempre para uma conta de canal (`channel_account_id`) e um contacto (`contact_id`); o `chat_id` é opcional — se omitido, o sistema encontra ou cria automaticamente o chat. O envio é assíncrono: a resposta é imediata com `status: "QUEUED"` e um `message_id`, e a entrega real é depois refletida nas flags/`status` da mensagem (consultáveis via os endpoints de leitura). Fora a janela de 24 horas do WhatsApp: mensagens livres (texto, mídia) só passam com a *messaging window* aberta — se estiver fechada, a API devolve `400` (`Messaging window is closed...`) e a via para reabrir a conversa é enviar um **template** (`POST /v2/messages/template`), que pode ser enviado a qualquer momento desde que o template esteja `APPROVED` pela Meta.

Autenticação: todas as chamadas exigem o header `api-token`. Erros comuns transversais: `400` (validação, incluindo UUID inválido), `401` (sem/`api-token` inválido), `403` (a funcionalidade *attendance API* não está ativa na conta — aplicável à criação de nota) e `404` (chat, contacto, conta de canal, template ou mensagem não encontrados). Esta fatia não traz guia passo-a-passo de SMS/VOICE/Webhooks, pelo que não há fluxo adicional a reproduzir para além do que consta acima.

## Índice de endpoints

| Método | Caminho | O que faz |
|---|---|---|
| GET | `/v2/messages/chat/{chatId}` | Lista, de forma paginada, as mensagens de um chat (ordenadas por `created_at` desc). |
| GET | `/v2/messages/{id}` | Obtém uma mensagem pelo seu ID. |
| POST | `/v2/messages/note` | Cria uma nota interna (anotação privada) numa conversa. |
| POST | `/v2/messages/text` | Envia uma mensagem de texto via WhatsApp Oficial. |
| POST | `/v2/messages/image` | Envia uma mensagem de imagem via WhatsApp Oficial. |
| POST | `/v2/messages/video` | Envia uma mensagem de vídeo via WhatsApp Oficial. |
| POST | `/v2/messages/document` | Envia uma mensagem de documento via WhatsApp Oficial. |
| POST | `/v2/messages/audio` | Envia uma mensagem de áudio (ou nota de voz/PTT) via WhatsApp Oficial. |
| POST | `/v2/messages/template` | Envia um template (HSM) aprovado via WhatsApp Oficial. |

---

## `GET` `/v2/messages/chat/{chatId}` — Listar mensagens de um chat

**O que faz.** Devolve uma lista paginada das mensagens de um chat específico (`WHATSAPP_OFFICIAL`, `WHATSAPP` ou `INSTAGRAM`). As mensagens são ordenadas por `created_at` descendente. Suporta paginação e filtros por tipo de conteúdo e por janelas de tempo (`created_at`/`updated_at`).

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.**

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `chatId` | string (uuid) | Sim | UUID do chat (ex.: `550e8400-e29b-41d4-a716-446655440000`). |

**Parâmetros de query.**

| Nome | Tipo | Obrigatório | Default | Descrição |
|---|---|---|---|---|
| `limit` | integer | Não | `200` | Número máximo de linhas devolvidas (mín. `1`, máx. `1000`). |
| `offset` | integer | Não | `0` | Número de linhas ignoradas no resultado (mín. `0`). |
| `page` | integer | Não | `1` | Seleciona a página do resultado (mín. `1`). |
| `content_type` | string (enum) | Não | — | Filtra as mensagens por tipo de conteúdo. Valores: `TEXT`, `IMAGE`, `AUDIO`, `VIDEO`, `DOCUMENT`. |
| `created_at_start` | string (`DateTime`, ISO-8601) | Não | — | Filtra mensagens cujo `created_at` é maior ou igual a esta data. Um `YYYY-MM-DD` simples é tratado como o início do dia. Útil para extração incremental por janela de tempo. |
| `created_at_end` | string (`DateTime`, ISO-8601) | Não | — | Filtra mensagens cujo `created_at` é menor ou igual a esta data. Um `YYYY-MM-DD` simples normaliza para o início do dia, logo **exclui** esse dia — passe um timestamp completo (ex.: `...T23:59:59Z`) para o incluir. |
| `updated_at_start` | string (`DateTime`, ISO-8601) | Não | — | Filtra mensagens cujo `updated_at` é maior ou igual a esta data. O `updated_at` reflete a última alteração à mensagem — incluindo mudanças de estado de entrega/leitura — pelo que serve para re-obter mensagens cujo estado mudou desde a última sincronização. |
| `updated_at_end` | string (`DateTime`, ISO-8601) | Não | — | Filtra mensagens cujo `updated_at` é menor ou igual a esta data. Um `YYYY-MM-DD` simples normaliza para o início do dia, logo **exclui** esse dia — passe um timestamp completo (ex.: `...T23:59:59Z`) para o incluir. |

**Corpo da requisição.** Nenhum.

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `200` | Lista paginada de mensagens | `PaginatedV2` + `data: array<Message>` |
| `400` | Formato de UUID inválido | `{ status: integer, message: string }` |
| `401` | Erro de autenticação — `api-token` inválido ou em falta | — |

O envelope `PaginatedV2` inclui os campos:

| Nome | Tipo | Descrição |
|---|---|---|
| `status` | integer | Estado da resposta (ex.: `200`). |
| `total_count` | integer | Total de itens de acordo com os filtros atuais. |
| `page` | integer | Página atual. |
| `total_pages` | integer | Total de páginas de acordo com os filtros atuais. |
| `has_next` | boolean | Indica se existe página seguinte. |
| `has_previous` | boolean | Indica se existe página anterior. |
| `data` | array\<`Message`\> | Lista de mensagens (ver schema `Message`). |

Campos de cada `Message`:

| Nome | Tipo | Descrição |
|---|---|---|
| `id` | string (uuid) | UUID da mensagem. |
| `created_at` | string (`DateTime`) | Data de criação (ISO-8601). |
| `updated_at` | string (`DateTime`) | Data da última alteração (ISO-8601) — inclui mudanças de estado de entrega/leitura. |
| `chat_id` | string (uuid), nullable | UUID do chat a que pertence a mensagem. |
| `user_id` | string (uuid), nullable | UUID do utilizador associado à mensagem. |
| `content` | string | Conteúdo textual da mensagem. |
| `type` | string (enum) | Emissor/tipo da mensagem: `USER`, `CUSTOMER`, `EVENT`, `NOTE`. |
| `sent` | boolean | Indica se a mensagem foi enviada. |
| `seen` | boolean | Indica se a mensagem foi vista/lida. |
| `delivered` | boolean | Indica se a mensagem foi entregue. |
| `content_type` | string | Tipo de conteúdo (ex.: `TEXT`, `IMAGE`, `AUDIO`, `VIDEO`, `DOCUMENT`). |
| `content_url` | string, nullable | URL do ficheiro de conteúdo, quando aplicável (mídia). |
| `external_id` | string, nullable | Identificador externo do provedor (ex.: `wamid.HBgNNTU0ODk...` da Meta). |
| `content_object` | object, nullable | Objeto de conteúdo estruturado, quando aplicável. |
| `content_action` | object, nullable | Objeto de ação de conteúdo, quando aplicável. |
| `status` | string (enum) | Estado derivado de `sent`/`delivered`/`seen`: `QUEUED`, `SENT`, `DELIVERED`, `READ`. |
| `source` | string, nullable | Origem da mensagem (ex.: `API`, `CAMPAIGN`, `WEB`). |

**Exemplo — requisição**
```bash
curl -X GET "https://api.clint.digital/v2/messages/chat/550e8400-e29b-41d4-a716-446655440000?limit=200&page=1&content_type=TEXT&updated_at_start=2026-06-01T00:00:00Z" \
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
      "updated_at": "2020-01-01T14:16:00.000000+00:00",
      "chat_id": "550e8400-e29b-41d4-a716-446655440000",
      "user_id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8",
      "content": "Hello, how can I help you?",
      "type": "USER",
      "sent": true,
      "seen": false,
      "delivered": true,
      "content_type": "TEXT",
      "content_url": null,
      "external_id": "wamid.HBgNNTU0ODk...",
      "content_object": null,
      "content_action": null,
      "status": "DELIVERED",
      "source": "API"
    }
  ]
}
```

**Exemplo — resposta (`400`)**
```json
{
  "status": 400,
  "message": "Param chat_id must be a valid UUID"
}
```

**Notas (inteligência LL Mídia).** Este é o endpoint-chave para extração incremental: combine `updated_at_start` com o timestamp da última sincronização para apanhar apenas mensagens novas **ou** cujo estado de entrega/leitura mudou (as flags `delivered`/`seen` alteram o `updated_at`). Cuidado com o "off-by-one" nas datas de fim: `created_at_end`/`updated_at_end` com um `YYYY-MM-DD` simples normalizam para o início do dia e **excluem** esse dia — para incluir o dia inteiro, passe `...T23:59:59Z`. A ordenação é sempre `created_at` desc; para paginar de forma robusta em bases grandes, itere com `page` guiando-se por `has_next` em vez de confiar só em `total_count`. Note que a leitura abrange os três canais (`WHATSAPP_OFFICIAL`, `WHATSAPP`, `INSTAGRAM`), ao contrário do envio.

---

## `GET` `/v2/messages/{id}` — Obter mensagem

**O que faz.** Devolve uma única mensagem pelo seu ID. Só retorna mensagens que pertençam ao dono autenticado (owner).

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.**

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string (uuid) | Sim | UUID da mensagem (ex.: `8feade82-d77b-4e8b-9d35-fd43e972b5c8`). |

**Parâmetros de query.**

| Nome | Tipo | Obrigatório | Default | Descrição |
|---|---|---|---|---|
| `content_type` | string (enum) | Não | — | Filtra a mensagem por tipo de conteúdo. Valores: `TEXT`, `IMAGE`, `AUDIO`, `VIDEO`, `DOCUMENT`. |

**Corpo da requisição.** Nenhum.

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `200` | Objeto de mensagem | `{ status: integer, data: Message }` |
| `400` | Formato de UUID inválido | `{ status: integer, message: string }` |
| `401` | Erro de autenticação — `api-token` inválido ou em falta | — |
| `404` | Mensagem não encontrada | `{ status: integer, message: string }` |

Campos do corpo (`200`):

| Nome | Tipo | Descrição |
|---|---|---|
| `status` | integer | Estado da resposta (ex.: `200`). |
| `data` | `Message` | O objeto da mensagem (ver schema `Message`). |

**Exemplo — requisição**
```bash
curl -X GET "https://api.clint.digital/v2/messages/8feade82-d77b-4e8b-9d35-fd43e972b5c8" \
  -H "api-token: SUA_API_KEY"
```

**Exemplo — resposta (`200`)**
```json
{
  "status": 200,
  "data": {
    "id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8",
    "created_at": "2020-01-01T14:15:00.000000+00:00",
    "updated_at": "2020-01-01T14:16:00.000000+00:00",
    "chat_id": "550e8400-e29b-41d4-a716-446655440000",
    "user_id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8",
    "content": "Hello, how can I help you?",
    "type": "USER",
    "sent": true,
    "seen": false,
    "delivered": true,
    "content_type": "TEXT",
    "content_url": null,
    "external_id": "wamid.HBgNNTU0ODk...",
    "content_object": null,
    "content_action": null,
    "status": "DELIVERED",
    "source": "API"
  }
}
```

**Exemplo — resposta (`404`)**
```json
{
  "status": 404,
  "message": "Message not found"
}
```

**Notas (inteligência LL Mídia).** O acesso é *owner-scoped*: uma mensagem de outro dono devolve `404` (e não `403`), pelo que um `404` aqui pode significar "não existe" **ou** "não é sua". Use este endpoint para reconfirmar o `status` final de uma mensagem que enviou (após o `QUEUED` inicial dos endpoints de envio) — o `status` é derivado das flags `sent`/`delivered`/`seen`. Um `id` malformado devolve `400` (`Param ID must be a valid UUID`) antes de qualquer procura.

---

## `POST` `/v2/messages/note` — Criar nota de conversa

**O que faz.** Cria uma nota interna (anotação) numa conversa. A nota aparece no histórico da conversa tal como uma nota criada na interface, restrita ao dono autenticado. **As notas internas nunca são enviadas ao contacto.** Menções (`@`) no conteúdo são guardadas literalmente mas não disparam notificações de menção.

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.** Nenhum.

**Parâmetros de query.** Nenhum.

**Corpo da requisição.** `application/json` (obrigatório).

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `chat_id` | string (uuid) | Sim | UUID da conversa (chat) a que adicionar a nota. |
| `content` | string (máx. 4096) | Sim | Conteúdo textual da nota (1 a 4096 caracteres). |
| `author` | string (máx. 120) | Não | Nome de exibição mostrado como autor da nota no histórico. Por defeito, `"API"` quando omitido. |

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `201` | Nota criada | `{ status: integer, data: {...} }` |
| `400` | Erro de validação | `{ status: integer, message: string }` |
| `401` | Erro de autenticação — `api-token` inválido ou em falta | — |
| `403` | A funcionalidade *attendance API* não está ativa na conta | — |
| `404` | Chat não encontrado | `{ status: integer, message: string }` |

Campos do corpo `data` (`201`):

| Nome | Tipo | Descrição |
|---|---|---|
| `success` | boolean | Indica sucesso da operação (ex.: `true`). |
| `id` | string (uuid) | UUID da mensagem de nota criada. |
| `chat_id` | string (uuid) | UUID do chat onde a nota foi criada. |
| `content` | string | Conteúdo da nota criada. |
| `author` | string | Nome do autor apresentado. |
| `type` | string (enum) | Tipo da mensagem; sempre `NOTE`. |
| `created_at` | string (date-time) | Data de criação da nota (ISO-8601). |

**Exemplo — requisição**
```bash
curl -X POST "https://api.clint.digital/v2/messages/note" \
  -H "api-token: SUA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "chat_id": "550e8400-e29b-41d4-a716-446655440000",
    "content": "AI summary: customer asked about pricing and wants a callback tomorrow.",
    "author": "AI Assistant"
  }'
```

**Exemplo — resposta (`201`)**
```json
{
  "status": 201,
  "data": {
    "success": true,
    "id": "550e8400-e29b-41d4-a716-446655440099",
    "chat_id": "550e8400-e29b-41d4-a716-446655440000",
    "content": "AI summary: customer asked about pricing and wants a callback tomorrow.",
    "author": "AI Assistant",
    "type": "NOTE",
    "created_at": "2026-06-15T19:29:13.000Z"
  }
}
```

**Exemplo — resposta (`400`)**
```json
{
  "status": 400,
  "message": "Param content must not be empty"
}
```

**Notas (inteligência LL Mídia).** Este é o único endpoint de escrita da fatia que depende de uma feature flag específica — se a *attendance API* não estiver ativa na conta, devolve `403`. É a via ideal para registar automaticamente resumos de IA, contexto de CRM ou avisos de equipa dentro do fio da conversa, **sem** risco de o contacto os receber. Validações a antecipar (`400`): `chat_id is required`, `chat_id must be a valid UUID`, `content must not be empty`, `content must not exceed 4096 characters` e `author must not exceed 120 characters`. Se `author` for omitido, o histórico mostra `"API"`. Menções `@` são guardadas mas não notificam ninguém — não conte com elas para alertar agentes.

---

## `POST` `/v2/messages/text` — Enviar mensagem de texto

**O que faz.** Envia uma mensagem de texto via WhatsApp Oficial. Reutiliza o fluxo de envio existente — valida a conta de canal, o contacto e a janela de mensagens (*messaging window*) e depois despacha para o provedor WhatsApp. Retorna imediatamente com o estado `QUEUED`.

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.** Nenhum.

**Parâmetros de query.** Nenhum.

**Corpo da requisição.** `application/json` (obrigatório).

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `channel_account_id` | string (uuid) | Sim | UUID da conta de canal WhatsApp Oficial. |
| `contact_id` | string (uuid) | Sim | UUID do contacto destinatário da mensagem. |
| `message` | string | Sim | Conteúdo textual da mensagem. |
| `chat_id` | string (uuid) | Não | UUID de um chat existente. Se omitido, o sistema encontra ou cria um chat automaticamente. |

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `200` | Mensagem em fila para envio | `{ status: integer, data: {...} }` |
| `400` | Erro de validação | `{ status: integer, message: string }` |
| `401` | Erro de autenticação — `api-token` inválido ou em falta | — |
| `404` | Recurso não encontrado (conta de canal ou contacto) | `{ status: integer, message: string }` |

Campos do corpo `data` (`200`):

| Nome | Tipo | Descrição |
|---|---|---|
| `success` | boolean | Indica sucesso da operação (ex.: `true`). |
| `message_id` | string (uuid) | UUID da mensagem criada. |
| `chat_id` | string (uuid) | UUID do chat (existente ou recém-criado). |
| `status` | string (enum) | Estado inicial da mensagem; sempre `QUEUED`. |

**Exemplo — requisição**
```bash
curl -X POST "https://api.clint.digital/v2/messages/text" \
  -H "api-token: SUA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "channel_account_id": "550e8400-e29b-41d4-a716-446655440001",
    "contact_id": "550e8400-e29b-41d4-a716-446655440002",
    "message": "Hello, how can I help you?"
  }'
```

**Exemplo — resposta (`200`)**
```json
{
  "status": 200,
  "data": {
    "success": true,
    "message_id": "550e8400-e29b-41d4-a716-446655440099",
    "chat_id": "550e8400-e29b-41d4-a716-446655440003",
    "status": "QUEUED"
  }
}
```

**Exemplo — resposta (`400`)**
```json
{
  "status": 400,
  "message": "Messaging window is closed. Use a template message to reopen the conversation"
}
```

**Notas (inteligência LL Mídia).** O envio é assíncrono: `200` + `QUEUED` significa apenas "aceite para envio", não "entregue" — confirme o estado final via `GET /v2/messages/{id}`. As validações `400` são numerosas e vale a pena tratá-las distintamente: `channel_account_id is required`, `... must be a valid UUID`, `message must not be empty`, `Channel account is not connected` (conta desligada), `Channel account type must be WHATSAPP_OFFICIAL` (canal errado), `Contact does not have a phone number` e `Messaging window is closed...`. Este último é o caso mais frequente em produção: fora da janela de 24 h do WhatsApp, texto/mídia são bloqueados — reabra a conversa com um **template** (`POST /v2/messages/template`). Os `404` distinguem `Channel account not found` de `Contact not found` (ambos também disparam se o recurso for de outro dono).

---

## `POST` `/v2/messages/image` — Enviar mensagem de imagem

**O que faz.** Envia uma mensagem de imagem via WhatsApp Oficial. O URL da imagem é enviado diretamente para a API da Meta (sem upload). Retorna imediatamente com o estado `QUEUED`.

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.** Nenhum.

**Parâmetros de query.** Nenhum.

**Corpo da requisição.** `application/json` (obrigatório).

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `channel_account_id` | string (uuid) | Sim | UUID da conta de canal WhatsApp Oficial. |
| `contact_id` | string (uuid) | Sim | UUID do contacto destinatário da imagem. |
| `url` | string (uri) | Sim | URL direto da imagem a enviar. |
| `caption` | string | Não | Legenda opcional da imagem. |
| `chat_id` | string (uuid) | Não | UUID de um chat existente. Se omitido, o sistema encontra ou cria um chat automaticamente. |

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `200` | Mensagem de imagem em fila para envio | `{ status: integer, data: {...} }` |
| `400` | Erro de validação | `{ status: integer, message: string }` |
| `401` | Erro de autenticação — `api-token` inválido ou em falta | — |
| `404` | Recurso não encontrado | `{ status: integer, message: string }` |

Campos do corpo `data` (`200`):

| Nome | Tipo | Descrição |
|---|---|---|
| `success` | boolean | Indica sucesso da operação (ex.: `true`). |
| `message_id` | string (uuid) | UUID da mensagem criada. |
| `chat_id` | string (uuid) | UUID do chat (existente ou recém-criado). |
| `status` | string (enum) | Estado inicial da mensagem; sempre `QUEUED`. |

**Exemplo — requisição**
```bash
curl -X POST "https://api.clint.digital/v2/messages/image" \
  -H "api-token: SUA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "channel_account_id": "550e8400-e29b-41d4-a716-446655440001",
    "contact_id": "550e8400-e29b-41d4-a716-446655440002",
    "url": "https://example.com/photo.jpg",
    "caption": "Check out this photo!"
  }'
```

**Exemplo — resposta (`200`)**
```json
{
  "status": 200,
  "data": {
    "success": true,
    "message_id": "550e8400-e29b-41d4-a716-446655440099",
    "chat_id": "550e8400-e29b-41d4-a716-446655440003",
    "status": "QUEUED"
  }
}
```

**Exemplo — resposta (`400`)**
```json
{
  "status": 400,
  "message": "Param url is required"
}
```

**Notas (inteligência LL Mídia).** O `url` tem de ser publicamente acessível: a Meta faz *fetch* do ficheiro diretamente, pelo que URLs autenticados ou atrás de login falham do lado do provedor. Tal como no texto, aplica-se a janela de 24 h — imagem só passa com a *messaging window* aberta. O `caption` é opcional. O `404` de recurso não encontrado tende a devolver `Channel account not found` (ou o contacto, conforme o caso).

---

## `POST` `/v2/messages/video` — Enviar mensagem de vídeo

**O que faz.** Envia uma mensagem de vídeo via WhatsApp Oficial. O URL do vídeo é enviado diretamente para a API da Meta (sem upload). Retorna imediatamente com o estado `QUEUED`. O ficheiro tem de ser um formato de vídeo suportado pela Meta (ex.: `video/mp4` ou `video/3gpp`, vídeo H.264 com áudio AAC, máx. 16 MB); a Meta valida o ficheiro e rejeita formatos ou tamanhos não suportados.

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.** Nenhum.

**Parâmetros de query.** Nenhum.

**Corpo da requisição.** `application/json` (obrigatório).

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `channel_account_id` | string (uuid) | Sim | UUID da conta de canal WhatsApp Oficial. |
| `contact_id` | string (uuid) | Sim | UUID do contacto destinatário do vídeo. |
| `url` | string (uri) | Sim | URL direto do vídeo a enviar. |
| `caption` | string | Não | Legenda opcional do vídeo. |
| `chat_id` | string (uuid) | Não | UUID de um chat existente. Se omitido, o sistema encontra ou cria um chat automaticamente. |

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `200` | Mensagem de vídeo em fila para envio | `{ status: integer, data: {...} }` |
| `400` | Erro de validação | `{ status: integer, message: string }` |
| `401` | Erro de autenticação — `api-token` inválido ou em falta | — |
| `404` | Recurso não encontrado | `{ status: integer, message: string }` |

Campos do corpo `data` (`200`):

| Nome | Tipo | Descrição |
|---|---|---|
| `success` | boolean | Indica sucesso da operação (ex.: `true`). |
| `message_id` | string (uuid) | UUID da mensagem criada. |
| `chat_id` | string (uuid) | UUID do chat (existente ou recém-criado). |
| `status` | string (enum) | Estado inicial da mensagem; sempre `QUEUED`. |

**Exemplo — requisição**
```bash
curl -X POST "https://api.clint.digital/v2/messages/video" \
  -H "api-token: SUA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "channel_account_id": "550e8400-e29b-41d4-a716-446655440001",
    "contact_id": "550e8400-e29b-41d4-a716-446655440002",
    "url": "https://example.com/video.mp4",
    "caption": "Check out this video!"
  }'
```

**Exemplo — resposta (`200`)**
```json
{
  "status": 200,
  "data": {
    "success": true,
    "message_id": "550e8400-e29b-41d4-a716-446655440099",
    "chat_id": "550e8400-e29b-41d4-a716-446655440003",
    "status": "QUEUED"
  }
}
```

**Exemplo — resposta (`400`)**
```json
{
  "status": 400,
  "message": "Param url is required"
}
```

**Notas (inteligência LL Mídia).** As restrições de formato/tamanho são impostas pela **Meta**, não pela Clint: `video/mp4` ou `video/3gpp`, H.264 + AAC, máximo 16 MB. Um ficheiro fora destes limites é aceite como `QUEUED` pela API mas depois falha no provedor — monitorize o `status` final da mensagem via `GET /v2/messages/{id}` para detetar rejeições da Meta. Aplica-se a janela de 24 h; o `url` tem de ser publicamente acessível.

---

## `POST` `/v2/messages/document` — Enviar mensagem de documento

**O que faz.** Envia uma mensagem de documento via WhatsApp Oficial. O URL do documento é enviado diretamente para a API da Meta. O `filename` é obrigatório e será apresentado ao destinatário. Retorna imediatamente com o estado `QUEUED`.

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.** Nenhum.

**Parâmetros de query.** Nenhum.

**Corpo da requisição.** `application/json` (obrigatório).

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `channel_account_id` | string (uuid) | Sim | UUID da conta de canal WhatsApp Oficial. |
| `contact_id` | string (uuid) | Sim | UUID do contacto destinatário do documento. |
| `url` | string (uri) | Sim | URL direto do documento a enviar. |
| `filename` | string | Sim | Nome do ficheiro apresentado ao destinatário. |
| `caption` | string | Não | Legenda opcional do documento. |
| `chat_id` | string (uuid) | Não | UUID de um chat existente. Se omitido, o sistema encontra ou cria um chat automaticamente. |

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `200` | Mensagem de documento em fila para envio | `{ status: integer, data: {...} }` |
| `400` | Erro de validação | `{ status: integer, message: string }` |
| `401` | Erro de autenticação — `api-token` inválido ou em falta | — |
| `404` | Recurso não encontrado | `{ status: integer, message: string }` |

Campos do corpo `data` (`200`):

| Nome | Tipo | Descrição |
|---|---|---|
| `success` | boolean | Indica sucesso da operação (ex.: `true`). |
| `message_id` | string (uuid) | UUID da mensagem criada. |
| `chat_id` | string (uuid) | UUID do chat (existente ou recém-criado). |
| `status` | string (enum) | Estado inicial da mensagem; sempre `QUEUED`. |

**Exemplo — requisição**
```bash
curl -X POST "https://api.clint.digital/v2/messages/document" \
  -H "api-token: SUA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "channel_account_id": "550e8400-e29b-41d4-a716-446655440001",
    "contact_id": "550e8400-e29b-41d4-a716-446655440002",
    "url": "https://example.com/report.pdf",
    "filename": "report.pdf",
    "caption": "Monthly report"
  }'
```

**Exemplo — resposta (`200`)**
```json
{
  "status": 200,
  "data": {
    "success": true,
    "message_id": "550e8400-e29b-41d4-a716-446655440099",
    "chat_id": "550e8400-e29b-41d4-a716-446655440003",
    "status": "QUEUED"
  }
}
```

**Exemplo — resposta (`400`)**
```json
{
  "status": 400,
  "message": "Param filename is required"
}
```

**Notas (inteligência LL Mídia).** Ao contrário dos outros tipos de mídia, aqui o `filename` é **obrigatório** — é o nome que o destinatário vê no WhatsApp; inclua a extensão correta (ex.: `.pdf`) para que o cliente reconheça o tipo. O `url` deve ser publicamente acessível (fetch da Meta). Aplica-se a janela de 24 h. O `caption` é opcional.

---

## `POST` `/v2/messages/audio` — Enviar mensagem de áudio

**O que faz.** Envia uma mensagem de áudio via WhatsApp Oficial. O URL do áudio é enviado diretamente para a API da Meta. **Sem suporte a legenda (`caption`)** em mensagens de áudio. Definir `voice` como `true` entrega o ficheiro como **nota de voz (PTT)** do WhatsApp em vez de um anexo de áudio normal; para PTT o ficheiro tem de ser `.ogg` codificado com o codec OPUS (a Meta valida e rejeita outros formatos). Retorna imediatamente com o estado `QUEUED`.

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.** Nenhum.

**Parâmetros de query.** Nenhum.

**Corpo da requisição.** `application/json` (obrigatório).

| Nome | Tipo | Obrigatório | Default | Descrição |
|---|---|---|---|---|
| `channel_account_id` | string (uuid) | Sim | — | UUID da conta de canal WhatsApp Oficial. |
| `contact_id` | string (uuid) | Sim | — | UUID do contacto destinatário do áudio. |
| `url` | string (uri) | Sim | — | URL direto do ficheiro de áudio a enviar. |
| `voice` | boolean | Não | `false` | Quando `true`, o áudio é entregue como nota de voz (PTT) em vez de anexo normal. Requer um `.ogg` codificado com o codec OPUS. |
| `chat_id` | string (uuid) | Não | — | UUID de um chat existente. Se omitido, o sistema encontra ou cria um chat automaticamente. |

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `200` | Mensagem de áudio em fila para envio | `{ status: integer, data: {...} }` |
| `400` | Erro de validação | `{ status: integer, message: string }` |
| `401` | Erro de autenticação — `api-token` inválido ou em falta | — |
| `404` | Recurso não encontrado | `{ status: integer, message: string }` |

Campos do corpo `data` (`200`):

| Nome | Tipo | Descrição |
|---|---|---|
| `success` | boolean | Indica sucesso da operação (ex.: `true`). |
| `message_id` | string (uuid) | UUID da mensagem criada. |
| `chat_id` | string (uuid) | UUID do chat (existente ou recém-criado). |
| `status` | string (enum) | Estado inicial da mensagem; sempre `QUEUED`. |

**Exemplo — requisição**
```bash
curl -X POST "https://api.clint.digital/v2/messages/audio" \
  -H "api-token: SUA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "channel_account_id": "550e8400-e29b-41d4-a716-446655440001",
    "contact_id": "550e8400-e29b-41d4-a716-446655440002",
    "url": "https://example.com/audio.ogg",
    "voice": true
  }'
```

**Exemplo — resposta (`200`)**
```json
{
  "status": 200,
  "data": {
    "success": true,
    "message_id": "550e8400-e29b-41d4-a716-446655440099",
    "chat_id": "550e8400-e29b-41d4-a716-446655440003",
    "status": "QUEUED"
  }
}
```

**Exemplo — resposta (`400`)**
```json
{
  "status": 400,
  "message": "Param url is required"
}
```

**Notas (inteligência LL Mídia).** Duas particularidades face aos outros tipos de mídia: **não há `caption`** e existe a flag `voice`. Se quiser o efeito de "nota de voz" (o ícone de microfone/PTT no WhatsApp), passe `voice: true` **e** garanta que o ficheiro é `.ogg`/OPUS — qualquer outro formato é rejeitado pela Meta (o `QUEUED` inicial não garante entrega; confirme via `GET /v2/messages/{id}`). Com `voice: false` (ou omitido), o áudio segue como anexo normal. O `url` tem de ser publicamente acessível; aplica-se a janela de 24 h.

---

## `POST` `/v2/messages/template` — Enviar mensagem de template

**O que faz.** Envia um template de mensagem WhatsApp (HSM). Mensagens de template podem ser enviadas a **qualquer momento**, mesmo com a janela de 24 h fechada. O template tem de estar `APPROVED` pela Meta e pertencer à conta de canal indicada. Os placeholders de variáveis (`{{1}}`, `{{2}}`, etc.) do template são substituídos pelos valores de parâmetros fornecidos.

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.** Nenhum.

**Parâmetros de query.** Nenhum.

**Corpo da requisição.** `application/json` (obrigatório).

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `channel_account_id` | string (uuid) | Sim | UUID da conta de canal WhatsApp Oficial. |
| `contact_id` | string (uuid) | Sim | UUID do contacto destinatário do template. |
| `template_id` | string (uuid) | Sim | UUID do template de mensagem a enviar. Tem de ter estado `APPROVED`. |
| `chat_id` | string (uuid) | Não | UUID de um chat existente. Se omitido, o sistema encontra ou cria um chat automaticamente. |
| `parameters` | object | Não | Valores das variáveis para substituir `{{1}}`, `{{2}}`, etc. nos componentes do template. Cada posição do array mapeia para o número de placeholder correspondente. |
| `parameters.header` | array\<string\> | Não | Valores para as variáveis do componente de cabeçalho (header). |
| `parameters.body` | array\<string\> | Não | Valores para as variáveis do componente de corpo (body). |

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `200` | Mensagem de template em fila para envio | `{ status: integer, data: {...} }` |
| `400` | Erro de validação | `{ status: integer, message: string }` |
| `401` | Erro de autenticação — `api-token` inválido ou em falta | — |
| `404` | Recurso não encontrado | `{ status: integer, message: string }` |

Campos do corpo `data` (`200`):

| Nome | Tipo | Descrição |
|---|---|---|
| `success` | boolean | Indica sucesso da operação (ex.: `true`). |
| `message_id` | string (uuid) | UUID da mensagem criada. |
| `chat_id` | string (uuid) | UUID do chat (existente ou recém-criado). |
| `status` | string (enum) | Estado inicial da mensagem; sempre `QUEUED`. |

**Exemplo — requisição**
```bash
curl -X POST "https://api.clint.digital/v2/messages/template" \
  -H "api-token: SUA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "channel_account_id": "550e8400-e29b-41d4-a716-446655440001",
    "contact_id": "550e8400-e29b-41d4-a716-446655440002",
    "template_id": "550e8400-e29b-41d4-a716-446655440010",
    "parameters": {
      "header": ["John"],
      "body": ["John", "12345", "2024-01-15"]
    }
  }'
```

**Exemplo — resposta (`200`)**
```json
{
  "status": 200,
  "data": {
    "success": true,
    "message_id": "550e8400-e29b-41d4-a716-446655440099",
    "chat_id": "550e8400-e29b-41d4-a716-446655440003",
    "status": "QUEUED"
  }
}
```

**Exemplo — resposta (`400`)**
```json
{
  "status": 400,
  "message": "Template must have APPROVED status"
}
```

**Exemplo — resposta (`404`)**
```json
{
  "status": 404,
  "message": "Message template not found"
}
```

**Notas (inteligência LL Mídia).** Este é o **único endpoint de envio que ignora a janela de 24 h** — é a via correta para iniciar (ou reabrir) uma conversa após o silêncio do WhatsApp. O template tem de estar `APPROVED` pela Meta (senão `400`, `Template must have APPROVED status`) e pertencer à `channel_account_id` indicada. Os `parameters` mapeiam por posição para os placeholders numerados: cada string em `body[0]`, `body[1]`, ... substitui `{{1}}`, `{{2}}`, ... no componente respetivo — garanta que o número de valores bate certo com o desenho do template para evitar rejeições. Um `template_id` inexistente/errado devolve `404` (`Message template not found`). Padrão prático: quando `POST /v2/messages/text` responde `Messaging window is closed...`, faça *fallback* automático para um template.

---

## Objetos relacionados

- [`Message`](../03-modelo-de-dados.md#message) — a mensagem em si: `id`, `created_at`, `updated_at`, `chat_id`, `user_id`, `content`, `type` (`USER`/`CUSTOMER`/`EVENT`/`NOTE`), `sent`, `seen`, `delivered`, `content_type`, `content_url`, `external_id`, `content_object`, `content_action`, `status` (`QUEUED`/`SENT`/`DELIVERED`/`READ`) e `source`.
- [`PaginatedV2`](../03-modelo-de-dados.md#paginatedv2) — envelope de paginação `/v2` (chaves `snake_case`): `status`, `total_count`, `page`, `total_pages`, `has_next`, `has_previous`.
- [`DateTime`](../03-modelo-de-dados.md#datetime) — string de data/hora ISO-8601 usada em `created_at`/`updated_at` e nos filtros de janela de tempo.
- [`ID`](../03-modelo-de-dados.md#id) — string UUID usada como identificador de mensagem.
