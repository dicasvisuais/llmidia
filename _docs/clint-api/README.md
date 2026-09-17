# Clint API — Documentação interna (LL Mídia)

API para gerir contactos, negócios e tags na aplicação Clint.

**SMS & Voz:** enviar SMS e chamadas de voz e receber estado de entrega. Ver as secções **SMS**, **VOICE** e **Webhooks** abaixo para guias passo-a-passo.

> Documento de referência interno LL Mídia, em português (pt-PT), fiel à especificação OpenAPI oficial da Clint e enriquecido com a nossa inteligência prática de uso. Nomes de campos, parâmetros, valores de enum, headers e paths ficam **em inglês**, exactamente como na spec.

---

## Proveniência

Esta documentação é um "clone" interno gerado a partir da especificação **OpenAPI 3.0.2 oficial** da Clint.

| | |
|---|---|
| **Fonte** | OpenAPI oficial da Clint (`@clint-api/v1.0`) |
| **Base URL (produção)** | `https://api.clint.digital` |
| **Versão da API** | `1.0.0` |
| **Título oficial** | Clint API |
| **Spec crua** | [`./openapi.json`](./openapi.json) |

Regra de ouro: **não inventamos** campos, parâmetros, endpoints, valores de enum ou respostas. Tudo o que consta destes documentos provém da spec; qualquer acréscimo prático nosso vem marcado como inteligência LL Mídia ou `(inferência)`.

---

## Arranque rápido

### 1. Autenticação por header `api-token`

Todas as chamadas exigem o header `api-token` com a sua chave de API. Não há OAuth nem tokens de sessão — é uma única chave por conta, enviada em **todas** as requisições.

```
api-token: SUA_API_KEY
```

Sem o header (ou com uma chave inválida) a API responde `401`. Detalhes em [`./01-autenticacao.md`](./01-autenticacao.md).

### 2. Primeiro pedido autenticado — `GET /v1/contacts`

```bash
curl -X GET "https://api.clint.digital/v1/contacts?limit=10" \
  -H "api-token: SUA_API_KEY"
```

Uma resposta `200` devolve o envelope paginado `Paginated` com a lista de contactos. Se receber `401`, confirme o header `api-token`; se receber `403` com `"This feature is not available for your account."`, o recurso está bloqueado por feature flag/escopo na sua conta.

---

## Como a documentação está organizada

| Documento | Conteúdo |
|---|---|
| [`./01-autenticacao.md`](./01-autenticacao.md) | Header `api-token`, obtenção da chave, erros de autenticação. |
| [`./02-convencoes.md`](./02-convencoes.md) | Paginação (`Paginated`/`PaginatedV2`), filtros, campos, formatos de data, erros comuns (`401`/`403`/`404`/`422`). |
| [`./03-modelo-de-dados.md`](./03-modelo-de-dados.md) | Schemas e objetos de dados (Contact, Deal, Tag, Chat, Message, etc.), com âncora por schema. |
| [`./openapi.json`](./openapi.json) | Especificação OpenAPI crua (fonte da verdade). |
| `./endpoints/<slug>.md` | Um documento por categoria, com todos os endpoints detalhados. |

Notas transversais:

- **Versões:** endpoints em `/v1` (CRM base — contactos, negócios, tags, etc.) e `/v2` (mensageria/omnichannel, dashboards, atividades, SMS, voz, webhooks).
- **Paginação (v1):** envelope `Paginated`; query `limit` (máx. 1000, default 200), `offset` (≥ 0), `page` (≥ 1). Endpoints `/v2` usam `PaginatedV2` quando paginados.
- **IDs:** UUID (string). **Datas:** ISO-8601 (schema `DateTime`).

---

## Índice por categoria

18 categorias, 63 endpoints no total.

| # | Categoria | Endpoints | Documento |
|---|---|---|---|
| 1 | Contacts | 8 | [`./endpoints/contacts.md`](./endpoints/contacts.md) |
| 2 | Organizations | 2 | [`./endpoints/organizations.md`](./endpoints/organizations.md) |
| 3 | Deals | 6 | [`./endpoints/deals.md`](./endpoints/deals.md) |
| 4 | Groups | 2 | [`./endpoints/groups.md`](./endpoints/groups.md) |
| 5 | Lost Status | 2 | [`./endpoints/lost-status.md`](./endpoints/lost-status.md) |
| 6 | Origins | 2 | [`./endpoints/origins.md`](./endpoints/origins.md) |
| 7 | Tags | 4 | [`./endpoints/tags.md`](./endpoints/tags.md) |
| 8 | Users | 2 | [`./endpoints/users.md`](./endpoints/users.md) |
| 9 | Account | 1 | [`./endpoints/account.md`](./endpoints/account.md) |
| 10 | Channel Accounts | 2 | [`./endpoints/channel-accounts.md`](./endpoints/channel-accounts.md) |
| 11 | Message Templates | 2 | [`./endpoints/message-templates.md`](./endpoints/message-templates.md) |
| 12 | Chats | 4 | [`./endpoints/chats.md`](./endpoints/chats.md) |
| 13 | Messages | 9 | [`./endpoints/messages.md`](./endpoints/messages.md) |
| 14 | Dashboards | 4 | [`./endpoints/dashboards.md`](./endpoints/dashboards.md) |
| 15 | SMS | 2 | [`./endpoints/sms.md`](./endpoints/sms.md) |
| 16 | VOICE | 3 | [`./endpoints/voice.md`](./endpoints/voice.md) |
| 17 | Webhooks | 2 | [`./endpoints/webhooks.md`](./endpoints/webhooks.md) |
| 18 | Activities | 6 | [`./endpoints/activities.md`](./endpoints/activities.md) |

---

## Tabela mestra de endpoints

Todos os 63 endpoints, agrupados por categoria. A coluna **Doc** liga ao documento detalhado da categoria.

### Contacts (8)

| Método | Caminho | O que faz | Doc |
|---|---|---|---|
| GET | `/v1/contacts` | Lista contactos (paginado). | [contacts.md](./endpoints/contacts.md) |
| POST | `/v1/contacts` | Cria um contacto. | [contacts.md](./endpoints/contacts.md) |
| GET | `/v1/contacts/{id}` | Obtém um contacto por ID. | [contacts.md](./endpoints/contacts.md) |
| POST | `/v1/contacts/{id}` | Atualiza um contacto. | [contacts.md](./endpoints/contacts.md) |
| DELETE | `/v1/contacts/{id}` | Remove um contacto. | [contacts.md](./endpoints/contacts.md) |
| POST | `/v1/contacts/{id}/tags` | Adiciona tags a um contacto. | [contacts.md](./endpoints/contacts.md) |
| DELETE | `/v1/contacts/{id}/tags` | Remove uma tag de um contacto. | [contacts.md](./endpoints/contacts.md) |
| GET | `/v1/contacts/{id}/attachments` | Lista anexos de um contacto. | [contacts.md](./endpoints/contacts.md) |

### Organizations (2)

| Método | Caminho | O que faz | Doc |
|---|---|---|---|
| GET | `/v1/organizations/{id}` | Obtém uma organização por ID. | [organizations.md](./endpoints/organizations.md) |
| POST | `/v1/organizations/{id}` | Atualiza uma organização. | [organizations.md](./endpoints/organizations.md) |

### Deals (6)

| Método | Caminho | O que faz | Doc |
|---|---|---|---|
| GET | `/v1/deals` | Lista negócios (paginado). | [deals.md](./endpoints/deals.md) |
| POST | `/v1/deals` | Cria um negócio. | [deals.md](./endpoints/deals.md) |
| GET | `/v1/deals/{id}` | Obtém um negócio por ID. | [deals.md](./endpoints/deals.md) |
| POST | `/v1/deals/{id}` | Atualiza um negócio. | [deals.md](./endpoints/deals.md) |
| DELETE | `/v1/deals/{id}` | Remove um negócio. | [deals.md](./endpoints/deals.md) |
| GET | `/v2/deals/{id}/history` | Histórico (timeline) do negócio. | [deals.md](./endpoints/deals.md) |

### Groups (2)

| Método | Caminho | O que faz | Doc |
|---|---|---|---|
| GET | `/v1/groups` | Lista grupos (pipelines). | [groups.md](./endpoints/groups.md) |
| GET | `/v1/groups/{id}` | Obtém um grupo por ID. | [groups.md](./endpoints/groups.md) |

### Lost Status (2)

| Método | Caminho | O que faz | Doc |
|---|---|---|---|
| GET | `/v1/lost-status` | Lista motivos de perda. | [lost-status.md](./endpoints/lost-status.md) |
| GET | `/v1/lost-status/{id}` | Obtém um motivo de perda por ID. | [lost-status.md](./endpoints/lost-status.md) |

### Origins (2)

| Método | Caminho | O que faz | Doc |
|---|---|---|---|
| GET | `/v1/origins` | Lista origens. | [origins.md](./endpoints/origins.md) |
| GET | `/v1/origins/{id}` | Obtém uma origem por ID. | [origins.md](./endpoints/origins.md) |

### Tags (4)

| Método | Caminho | O que faz | Doc |
|---|---|---|---|
| GET | `/v1/tags` | Lista tags (paginado). | [tags.md](./endpoints/tags.md) |
| POST | `/v1/tags` | Cria uma tag. | [tags.md](./endpoints/tags.md) |
| GET | `/v1/tags/{id}` | Obtém uma tag por ID. | [tags.md](./endpoints/tags.md) |
| DELETE | `/v1/tags/{id}` | Remove uma tag. | [tags.md](./endpoints/tags.md) |

### Users (2)

| Método | Caminho | O que faz | Doc |
|---|---|---|---|
| GET | `/v1/users` | Lista utilizadores. | [users.md](./endpoints/users.md) |
| GET | `/v1/users/{id}` | Obtém um utilizador por ID. | [users.md](./endpoints/users.md) |

### Account (1)

| Método | Caminho | O que faz | Doc |
|---|---|---|---|
| GET | `/v1/account/fields` | Lista campos (personalizados) da conta. | [account.md](./endpoints/account.md) |

### Channel Accounts (2)

| Método | Caminho | O que faz | Doc |
|---|---|---|---|
| GET | `/v2/channel-accounts` | Lista contas de canal. | [channel-accounts.md](./endpoints/channel-accounts.md) |
| GET | `/v2/channel-accounts/{id}` | Obtém uma conta de canal por ID. | [channel-accounts.md](./endpoints/channel-accounts.md) |

### Message Templates (2)

| Método | Caminho | O que faz | Doc |
|---|---|---|---|
| GET | `/v2/message-templates` | Lista templates de mensagem. | [message-templates.md](./endpoints/message-templates.md) |
| GET | `/v2/message-templates/{id}` | Obtém um template de mensagem por ID. | [message-templates.md](./endpoints/message-templates.md) |

### Chats (4)

| Método | Caminho | O que faz | Doc |
|---|---|---|---|
| GET | `/v2/chats/contact/{contactId}` | Lista chats de um contacto. | [chats.md](./endpoints/chats.md) |
| GET | `/v2/chats/channel-account/{channelAccountId}` | Lista chats de uma conta de canal. | [chats.md](./endpoints/chats.md) |
| GET | `/v2/chats/{id}` | Obtém um chat por ID. | [chats.md](./endpoints/chats.md) |
| POST | `/v2/chats/{id}` | Atualiza um chat. | [chats.md](./endpoints/chats.md) |

### Messages (9)

| Método | Caminho | O que faz | Doc |
|---|---|---|---|
| GET | `/v2/messages/chat/{chatId}` | Lista mensagens de um chat. | [messages.md](./endpoints/messages.md) |
| GET | `/v2/messages/{id}` | Obtém uma mensagem por ID. | [messages.md](./endpoints/messages.md) |
| POST | `/v2/messages/note` | Cria uma nota no chat. | [messages.md](./endpoints/messages.md) |
| POST | `/v2/messages/text` | Envia mensagem de texto. | [messages.md](./endpoints/messages.md) |
| POST | `/v2/messages/image` | Envia mensagem de imagem. | [messages.md](./endpoints/messages.md) |
| POST | `/v2/messages/video` | Envia mensagem de vídeo. | [messages.md](./endpoints/messages.md) |
| POST | `/v2/messages/document` | Envia mensagem de documento. | [messages.md](./endpoints/messages.md) |
| POST | `/v2/messages/audio` | Envia mensagem de áudio. | [messages.md](./endpoints/messages.md) |
| POST | `/v2/messages/template` | Envia mensagem por template. | [messages.md](./endpoints/messages.md) |

### Dashboards (4)

| Método | Caminho | O que faz | Doc |
|---|---|---|---|
| GET | `/v2/dashboards` | Lista dashboards. | [dashboards.md](./endpoints/dashboards.md) |
| GET | `/v2/dashboards/{id}` | Obtém um dashboard por ID. | [dashboards.md](./endpoints/dashboards.md) |
| GET | `/v2/dashboards/{id}/data` | Obtém dados dos gráficos de um dashboard. | [dashboards.md](./endpoints/dashboards.md) |
| GET | `/v2/charts/{id}/data` | Obtém dados de um único gráfico. | [dashboards.md](./endpoints/dashboards.md) |

### SMS (2)

| Método | Caminho | O que faz | Doc |
|---|---|---|---|
| POST | `/v2/sms/bulk` | Envia SMS em massa. | [sms.md](./endpoints/sms.md) |
| POST | `/v2/sms` | Envia um SMS. | [sms.md](./endpoints/sms.md) |

### VOICE (3)

| Método | Caminho | O que faz | Doc |
|---|---|---|---|
| POST | `/v2/voice/audios` | Carrega um áudio de VOICE. | [voice.md](./endpoints/voice.md) |
| POST | `/v2/voice/bulk` | Envia chamadas de voz em massa. | [voice.md](./endpoints/voice.md) |
| POST | `/v2/voice` | Envia uma chamada de voz. | [voice.md](./endpoints/voice.md) |

### Webhooks (2)

| Método | Caminho | O que faz | Doc |
|---|---|---|---|
| GET | `/v2/webhooks` | Obtém a configuração de webhooks. | [webhooks.md](./endpoints/webhooks.md) |
| PUT | `/v2/webhooks` | Define a configuração de webhooks. | [webhooks.md](./endpoints/webhooks.md) |

### Activities (6)

| Método | Caminho | O que faz | Doc |
|---|---|---|---|
| GET | `/v2/activities` | Lista atividades. | [activities.md](./endpoints/activities.md) |
| POST | `/v2/activities` | Cria uma atividade. | [activities.md](./endpoints/activities.md) |
| GET | `/v2/activities/{id}` | Obtém uma atividade por ID. | [activities.md](./endpoints/activities.md) |
| POST | `/v2/activities/{id}` | Atualiza uma atividade. | [activities.md](./endpoints/activities.md) |
| DELETE | `/v2/activities/{id}` | Elimina uma atividade. | [activities.md](./endpoints/activities.md) |
| POST | `/v2/activities/{id}/complete` | Conclui/reabre uma atividade. | [activities.md](./endpoints/activities.md) |
