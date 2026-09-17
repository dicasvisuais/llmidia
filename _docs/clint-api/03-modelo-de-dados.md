# Modelo de dados (schemas) — Clint API

Referência de todos os schemas (objetos e tipos) devolvidos e aceites pela Clint API, extraída da especificação OpenAPI oficial.

> Fonte: OpenAPI oficial da Clint (`@clint-api/v1.0`) · Base `https://api.clint.digital` · Auth: header `api-token` · Documento interno LL Mídia.

## Como ler este documento

- **IDs** são sempre UUID (string) — ver [ID](#id). **Datas** seguem ISO-8601 — ver [DateTime](#datetime).
- Onde a spec usa `$ref` (referência entre schemas), a coluna *Exemplo/Notas* remete para o schema relacionado pela âncora (ex.: "ver [Tag](#tag)").
- Os nomes de campos, valores de enum e headers mantêm-se **em inglês**, exactamente como na spec.
- Cada schema tem uma âncora estável derivada do cabeçalho (ex.: `### Contact` → `#contact`), usada pelas ligações cruzadas dos restantes documentos.

**Ligações úteis:** [Índice geral](./README.md) · Endpoints: [contacts](./endpoints/contacts.md) · [organizations](./endpoints/organizations.md) · [groups](./endpoints/groups.md) · [lost-status](./endpoints/lost-status.md).

### Índice

- **Tipos partilhados / base:** [ID](#id) · [DateTime](#datetime) · [FullPhone](#fullphone) · [Fields](#fields) · [AccountFields](#accountfields) · [TagColor](#tagcolor) · [DealStatus](#dealstatus) · [Paginated](#paginated) · [PaginatedV2](#paginatedv2)
- **Objetos de domínio:** [Contact](#contact) · [ContactCreateSchema](#contactcreateschema) · [ContactAttachment](#contactattachment) · [Organization](#organization) · [OrganizationCreateSchema](#organizationcreateschema) · [Deal](#deal) · [DealCreateSchema](#dealcreateschema) · [DealUpdateSchema](#dealupdateschema) · [Group](#group) · [LostStatus](#loststatus) · [Origin](#origin) · [Tag](#tag) · [TagCreateSchema](#tagcreateschema) · [User](#user) · [ChannelAccount](#channelaccount) · [Chat](#chat) · [Message](#message) · [MessageTemplate](#messagetemplate) · [Dashboard](#dashboard) · [DashboardSummary](#dashboardsummary) · [ChartData](#chartdata) · [Activity](#activity)

---

## Parte 1 — Tipos partilhados / base

Tipos escalares e envelopes reutilizados por vários objetos de domínio.

### ID

Identificador único universal (UUID) que identifica cada recurso da API.

| Campo | Tipo | Exemplo/Notas |
|---|---|---|
| *(valor)* | string (uuid) | `8feade82-d77b-4e8b-9d35-fd43e972b5c8` |

### DateTime

Data e hora em formato ISO-8601 (com fuso horário).

| Campo | Tipo | Exemplo/Notas |
|---|---|---|
| *(valor)* | string (date-time) | `2020-01-01T14:15:00.000000+00:00` |

### FullPhone

Telefone completo, com DDI, em formato E.164.

| Campo | Tipo | Exemplo/Notas |
|---|---|---|
| *(valor)* | string | `+5548999999999` |

### Fields

Mapa aberto de campos personalizados (chave → valor) associado a um recurso. A estrutura das chaves disponíveis por conta é descrita em [AccountFields](#accountfields).

| Campo | Tipo | Exemplo/Notas |
|---|---|---|
| *(objeto)* | object | Mapa livre chave→valor. Estrutura definida pela conta — ver [AccountFields](#accountfields). |

### AccountFields

Definição, ao nível da conta, dos grupos e campos personalizados disponíveis para cada entidade (`DEAL`, `CONTACT`, `ORGANIZATION`).

| Campo | Tipo | Exemplo/Notas |
|---|---|---|
| `groups` | object | Grupos de campos por entidade. |
| `groups.DEAL` | object | Mapa chave→nome do grupo (negócios). |
| `groups.CONTACT` | object | Mapa chave→nome do grupo (contactos). |
| `groups.ORGANIZATION` | object | Mapa chave→nome do grupo (organizações). |
| `fields` | object | Definição dos campos por entidade. |
| `fields.DEAL.type` / `fields.CONTACT.type` / `fields.ORGANIZATION.type` | string | Tipo do campo. Ex.: `TEXT`. |
| `fields.DEAL.group` / `fields.CONTACT.group` / `fields.ORGANIZATION.group` | string | Grupo a que o campo pertence. Ex.: `default`. |
| `fields.DEAL.label` / `fields.CONTACT.label` / `fields.ORGANIZATION.label` | string | Rótulo do campo. Ex.: `notes`. |

### TagColor

Cor da etiqueta, restrita a uma paleta de valores hexadecimais.

| Campo | Tipo | Exemplo/Notas |
|---|---|---|
| *(valor)* | string (enum) | Default `#f44336`. Valores: `#f44336`, `#e91e63`, `#9c27b0`, `#673ab7`, `#3f51b5`, `#2196f3`, `#03a9f4`, `#00bcd4`, `#009688`, `#4caf50`, `#8bc34a`, `#faa200`, `#ff9800`, `#ff5722`, `#795548`, `#607d8b`. |

Usado por [Tag](#tag) e [TagCreateSchema](#tagcreateschema).

### DealStatus

Estado de um negócio no funil.

| Campo | Tipo | Exemplo/Notas |
|---|---|---|
| *(valor)* | string (enum) | Default `OPEN`. Valores: `OPEN`, `WON`, `LOST`. |

Usado por [Deal](#deal), [DealCreateSchema](#dealcreateschema) e [DealUpdateSchema](#dealupdateschema).

### Paginated

Envelope de paginação das respostas **v1** (chaves em camelCase).

| Campo | Tipo | Exemplo/Notas |
|---|---|---|
| `status` | integer | Código de estado da resposta. Ex.: `200`. |
| `totalCount` | integer | Total de itens conforme os filtros atuais. Ex.: `50`. |
| `page` | integer | Página atual. Ex.: `1`. |
| `totalPages` | integer | Total de páginas conforme os filtros. Ex.: `10`. |
| `hasNext` | boolean | Indica se existe página seguinte. |
| `hasPrevious` | boolean | Indica se existe página anterior. |

### PaginatedV2

Envelope de paginação das respostas **v2** (chaves em snake_case).

| Campo | Tipo | Exemplo/Notas |
|---|---|---|
| `status` | integer | Código de estado da resposta. Ex.: `200`. |
| `total_count` | integer | Total de itens conforme os filtros atuais. Ex.: `50`. |
| `page` | integer | Página atual. Ex.: `1`. |
| `total_pages` | integer | Total de páginas conforme os filtros. Ex.: `10`. |
| `has_next` | boolean | Indica se existe página seguinte. |
| `has_previous` | boolean | Indica se existe página anterior. |

---

## Parte 2 — Objetos de domínio

Recursos do CRM e da mensageria/omnichannel.

### Contact

Contacto (pessoa) no CRM, com etiquetas e campos personalizados.

| Campo | Tipo | Exemplo/Notas |
|---|---|---|
| `id` | string (uuid) | ver [ID](#id). |
| `created_at` | string (date-time) | ver [DateTime](#datetime). |
| `updated_at` | string (date-time) | ver [DateTime](#datetime). |
| `name` | string | `Contact name`. |
| `email` | string | `contact@email.com`. |
| `organization` | string | `Organization name`. |
| `instagram` | string | `Instagram ID`. |
| `tags` | array | Etiquetas do contacto — itens do tipo [Tag](#tag). |
| `fields` | object | Campos personalizados — ver [Fields](#fields). |
| `fullPhone` | string | Telefone completo — ver [FullPhone](#fullphone). |

### ContactCreateSchema

Corpo para criar um contacto. Ver endpoint [contacts](./endpoints/contacts.md).

| Campo | Tipo | Exemplo/Notas |
|---|---|---|
| `name` | string | `Contact name`. |
| `ddi` | string | Código do país. Ex.: `+55`. |
| `phone` | string | Número sem DDI. Ex.: `48999999999`. |
| `email` | string | `contact@email.com`. |
| `username` | string | ID do Instagram. Ex.: `Instagram ID`. |
| `fields` | object | Campos personalizados (mapa chave→valor). Suporta o subgrupo `organization` (mapa chave→valor). |

### ContactAttachment

Anexo (documento) associado a um contacto, com metadados de quem o carregou e do ficheiro.

| Campo | Tipo | Exemplo/Notas |
|---|---|---|
| `id` | string (uuid) | ver [ID](#id). |
| `contact_id` | string (uuid) | Contacto associado — ver [Contact](#contact) / [ID](#id). |
| `created_at` | string (date-time) | ver [DateTime](#datetime). |
| `updated_at` | string (date-time) | ver [DateTime](#datetime). |
| `uploaded_by` | object | Autor do carregamento. |
| `uploaded_by.id` | string (uuid) | ver [ID](#id). |
| `uploaded_by.first_name` | string | `Maria`. |
| `uploaded_by.last_name` | string | `Silva`. |
| `document` | object | Metadados do ficheiro. |
| `document.id` | string (uuid) | ver [ID](#id). |
| `document.name` | string | `contract-signed`. |
| `document.file_name` | string | `contract-signed.pdf`. |
| `document.url` | string (uri) | URL do ficheiro. Ex.: `https://file.clint.digital/.../contract-signed.pdf`. |
| `document.extension` | string | `pdf`. |
| `document.file_size` | integer | Tamanho em bytes. Ex.: `204800`. |

### Organization

Organização (empresa) no CRM, com campos personalizados.

| Campo | Tipo | Exemplo/Notas |
|---|---|---|
| `id` | string (uuid) | ver [ID](#id). |
| `created_at` | string (date-time) | ver [DateTime](#datetime). |
| `updated_at` | string (date-time) | ver [DateTime](#datetime). |
| `name` | string | `Organization name`. |
| `fields` | object | Campos personalizados — ver [Fields](#fields). |

### OrganizationCreateSchema

Corpo para criar uma organização. Ver endpoint [organizations](./endpoints/organizations.md).

| Campo | Tipo | Exemplo/Notas |
|---|---|---|
| `name` | string | `Organization name`. |
| `fields` | object | Campos personalizados (mapa chave→valor). |

### Deal

Negócio (oportunidade) no funil, com contacto, responsável, etapa e estado.

| Campo | Tipo | Exemplo/Notas |
|---|---|---|
| `id` | string (uuid) | ver [ID](#id). |
| `origin_id` | string (uuid) | Origem/funil — ver [Origin](#origin) / [ID](#id). |
| `user` | object | Responsável pelo negócio — ver [User](#user). |
| `user.id` | string (uuid) | ver [ID](#id). |
| `user.full_name` | string | `User full name`. |
| `contact` | object | Contacto do negócio — ver [Contact](#contact). |
| `contact.id` | string (uuid) | ver [ID](#id). |
| `contact.name` | string | `Contact name`. |
| `contact.email` | string | `contact@email.com`. |
| `contact.phone` | string | `+5548999999999`. |
| `created_at` | string (date-time) | ver [DateTime](#datetime). |
| `stage_id` | string (uuid) | Etapa atual — ver [ID](#id). |
| `updated_stage_at` | string (date-time) | ver [DateTime](#datetime). |
| `status` | string (enum) | ver [DealStatus](#dealstatus). |
| `won_at` | string (date-time) | ver [DateTime](#datetime). |
| `won_by` | string (uuid) | ver [ID](#id). |
| `lost_status_id` | string (uuid) | Motivo de perda — ver [LostStatus](#loststatus) / [ID](#id). |
| `lost_at` | string (date-time) | ver [DateTime](#datetime). |
| `lost_by` | string (uuid) | ver [ID](#id). |
| `fields` | object | Campos personalizados — ver [Fields](#fields). |

### DealCreateSchema

Corpo para criar um negócio. Campo `origin_id` é **obrigatório**.

| Campo | Tipo | Exemplo/Notas |
|---|---|---|
| `origin_id` | string (uuid) | **Obrigatório.** Origem/funil — ver [Origin](#origin) / [ID](#id). |
| `name` | string | `Contact name`. |
| `phone` | string | `48999999999`. |
| `email` | string | `contact@email.com`. |
| `username` | string | ID do Instagram. |
| `value` | number | Valor do negócio. Ex.: `200.5`. |
| `stage_id` | string (uuid) | Etapa — ver [ID](#id). |
| `user_id` | string (uuid) | Responsável — ver [User](#user) / [ID](#id). |
| `contact_id` | string (uuid) | Contacto existente — ver [Contact](#contact) / [ID](#id). |
| `fields` | object | Campos personalizados (mapa chave→valor); suporta os subgrupos `contact` e `organization`. |

### DealUpdateSchema

Corpo para atualizar um negócio. Todos os campos são opcionais.

| Campo | Tipo | Exemplo/Notas |
|---|---|---|
| `name` | string | `Contact name`. |
| `phone` | string | `48999999999`. |
| `email` | string | `contact@email.com`. |
| `value` | number | `200.5`. |
| `stage_id` | string (uuid) | ver [ID](#id). |
| `status` | string (enum) | ver [DealStatus](#dealstatus). |
| `user_id` | string (uuid) | ver [User](#user) / [ID](#id). |
| `origin_id` | string (uuid) | ver [Origin](#origin) / [ID](#id). |
| `fields` | object | Campos personalizados (mapa chave→valor); suporta os subgrupos `contact` e `organization`. |

### Group

Grupo (pasta) que agrupa origens/funis. Ver endpoint [groups](./endpoints/groups.md).

| Campo | Tipo | Exemplo/Notas |
|---|---|---|
| `id` | string (uuid) | ver [ID](#id). |
| `name` | string | `Group name`. |
| `archived_at` | string (date-time) | ver [DateTime](#datetime). |
| `archived_by` | string (uuid) | ver [ID](#id). |

### LostStatus

Motivo de perda de um negócio. Ver endpoint [lost-status](./endpoints/lost-status.md).

| Campo | Tipo | Exemplo/Notas |
|---|---|---|
| `id` | string (uuid) | ver [ID](#id). |
| `name` | string | `Lost status name`. |

### Origin

Origem/funil, com o grupo a que pertence e as respetivas etapas (`stages`).

| Campo | Tipo | Exemplo/Notas |
|---|---|---|
| `id` | string (uuid) | ver [ID](#id). |
| `name` | string | `Origin name`. |
| `group` | object | Grupo a que pertence — ver [Group](#group). |
| `group.id` | string (uuid) | ver [ID](#id). |
| `group.name` | string | `Group name`. |
| `stages` | array | Etapas do funil. |
| `stages[].id` | string (uuid) | ver [ID](#id). |
| `stages[].label` | string | Nome da etapa. |
| `stages[].order` | integer | Ordem da etapa. |
| `stages[].type` | string | Tipo da etapa. |
| `archived_at` | string (date-time) | ver [DateTime](#datetime). |
| `archived_by` | string (uuid) | ver [ID](#id). |

### Tag

Etiqueta aplicável a contactos, com nome e cor.

| Campo | Tipo | Exemplo/Notas |
|---|---|---|
| `id` | string (uuid) | ver [ID](#id). |
| `name` | string | `Tag name`. |
| `color` | string (enum) | ver [TagColor](#tagcolor). |
| `created_at` | string (date-time) | Data de criação. Ex.: `2026-01-15T10:30:00.000Z`. |

### TagCreateSchema

Corpo para criar uma etiqueta. Campos `name` e `color` são **obrigatórios**.

| Campo | Tipo | Exemplo/Notas |
|---|---|---|
| `name` | string | **Obrigatório.** `Tag name`. |
| `color` | string (enum) | **Obrigatório.** ver [TagColor](#tagcolor). |

### User

Utilizador (operador) da conta.

| Campo | Tipo | Exemplo/Notas |
|---|---|---|
| `id` | string (uuid) | ver [ID](#id). |
| `email` | string | `User e-mail`. |
| `first_name` | string | `User first name`. |
| `last_name` | string | `User last name`. |

### ChannelAccount

Conta de canal de mensageria (WhatsApp/Instagram) ligada à conta.

| Campo | Tipo | Exemplo/Notas |
|---|---|---|
| `id` | string (uuid) | ver [ID](#id). |
| `created_at` | string (date-time) | ver [DateTime](#datetime). |
| `name` | string | `WhatsApp Business`. |
| `type` | string (enum) | `WHATSAPP_OFFICIAL` (Cloud API — único tipo que suporta envio por esta API), `WHATSAPP` (WhatsApp Web/ZAPI), `INSTAGRAM` (Instagram Direct). |
| `status` | string (enum) | `CONNECTED`, `DISCONNECTED`, `CANCELLED`. |
| `avatar` | string (nullable) | URL do avatar. Ex.: `https://example.com/avatar.jpg`. |
| `identifier` | string (nullable) | Identificador do canal. Ex.: `5548999999999`. |
| `team_id` | string (uuid, nullable) | Equipa — ver [ID](#id). |

### Chat

Conversa (chat) de omnichannel associada a um contacto e a uma conta de canal.

| Campo | Tipo | Exemplo/Notas |
|---|---|---|
| `id` | string (uuid) | ver [ID](#id). |
| `created_at` | string (date-time) | ver [DateTime](#datetime). |
| `contact_id` | string (uuid) | Contacto — ver [Contact](#contact) / [ID](#id). |
| `user_id` | string (uuid) | Atendente — ver [User](#user) / [ID](#id). |
| `status` | string (enum) | `OPEN`, `CLOSED`, `WAITING`, `SNOOZED`, `REOPENED`. |
| `seen` | boolean | `false`. |
| `unread` | boolean | `true`. |
| `replied` | boolean | `false`. |
| `unseen_count` | integer | Nº de mensagens não vistas. Ex.: `3`. |
| `last_message_at` | string (date-time) | ver [DateTime](#datetime). |
| `last_response_at` | string (date-time, nullable) | Última resposta. Ex.: `2024-01-15T09:00:00.000Z`. |
| `last_status_at` | string (date-time, nullable) | Última mudança de estado. Ex.: `2024-01-15T10:30:00.000Z`. |
| `first_response_at` | string (date-time, nullable) | Primeira resposta. Ex.: `2024-01-14T08:00:00.000Z`. |
| `channel_account_id` | string (uuid) | Conta de canal — ver [ChannelAccount](#channelaccount) / [ID](#id). |
| `team_id` | string (uuid, nullable) | Equipa. Ex.: `8feade82-d77b-4e8b-9d35-fd43e972b5c8`. |
| `closed_at` | string (date-time, nullable) | Quando o chat foi fechado. |
| `first_customer_message_at` | string (date-time, nullable) | Primeira mensagem do cliente. |
| `close_window_at` | string (date-time, nullable) | Fecho da janela de mensageria do WhatsApp (24h após a última mensagem do cliente). |

### Message

Mensagem individual dentro de um [Chat](#chat).

| Campo | Tipo | Exemplo/Notas |
|---|---|---|
| `id` | string (uuid) | ver [ID](#id). |
| `created_at` | string (date-time) | ver [DateTime](#datetime). |
| `updated_at` | string (date-time) | ver [DateTime](#datetime). |
| `chat_id` | string (uuid, nullable) | Chat — ver [Chat](#chat). |
| `user_id` | string (uuid, nullable) | Autor (atendente) — ver [User](#user). |
| `content` | string | `Hello, how can I help you?`. |
| `type` | string (enum) | `USER`, `CUSTOMER`, `EVENT`, `NOTE`. |
| `sent` | boolean | `true`. |
| `seen` | boolean | `false`. |
| `delivered` | boolean | `true`. |
| `content_type` | string | `TEXT`. |
| `content_url` | string (nullable) | URL de anexo. Ex.: `https://example.com/file.jpg`. |
| `external_id` | string (nullable) | ID externo. Ex.: `wamid.HBgNNTU0ODk...`. |
| `content_object` | object (nullable) | Objeto de conteúdo estruturado. |
| `content_action` | object (nullable) | Ação de conteúdo (interativos). |
| `status` | string (enum) | `QUEUED`, `SENT`, `DELIVERED`, `READ` (derivado das flags `sent`/`delivered`/`seen`). |
| `source` | string (nullable) | Origem da mensagem. Ex.: `API`, `CAMPAIGN`, `WEB`. |

### MessageTemplate

Template de mensagem do WhatsApp Official (aprovado na Meta).

| Campo | Tipo | Exemplo/Notas |
|---|---|---|
| `id` | string (uuid) | ver [ID](#id). |
| `external_id` | string | ID do template na Meta. Ex.: `123456789`. |
| `name` | string | Nome do template. Ex.: `welcome_message`. |
| `status` | string (enum) | Estado de aprovação: `APPROVED`, `PENDING`, `REJECTED`. |
| `language` | string | Código de idioma. Ex.: `pt_BR`. |
| `category` | string (enum) | `MARKETING`, `UTILITY`, `AUTHENTICATION`. |
| `components` | array | Componentes do template (`HEADER`, `BODY`, `FOOTER`, `BUTTONS`). |
| `components[].type` | string (enum) | `HEADER`, `BODY`, `FOOTER`, `BUTTONS`. |
| `components[].text` | string | Ex.: `Hello {{1}}, welcome!`. |
| `components[].format` | string | Ex.: `TEXT`. |
| `variables` | object (nullable) | Mapeamento de variáveis por componente. Ex.: `{"body":["{{1}}"]}`. |

### Dashboard

Dashboard com o detalhe completo dos seus gráficos e respetivo posicionamento.

| Campo | Tipo | Exemplo/Notas |
|---|---|---|
| `id` | string (uuid) | ver [ID](#id). |
| `name` | string | `Sales Dashboard`. |
| `created_at` | string (date-time) | ver [DateTime](#datetime). |
| `updated_at` | string (date-time) | ver [DateTime](#datetime). |
| `charts` | array | Gráficos do dashboard (dados detalhados em [ChartData](#chartdata)). |
| `charts[].id` | string (uuid) | ver [ID](#id). |
| `charts[].name` | string | `Monthly Revenue`. |
| `charts[].type` | string (enum) | `line`, `area`, `bar`, `stackedBar`, `table`, `number`, `kpiPanel`, `pie`, `donut`, `funnel`. |
| `charts[].layout` | object | Posição na grelha do dashboard (`x`, `y`, `w`, `h` — inteiros). |

### DashboardSummary

Resumo de dashboard usado no endpoint de listagem (sem detalhe dos gráficos).

| Campo | Tipo | Exemplo/Notas |
|---|---|---|
| `id` | string (uuid) | ver [ID](#id). |
| `name` | string | `Sales Dashboard`. |
| `created_at` | string (date-time) | ver [DateTime](#datetime). |
| `updated_at` | string (date-time) | ver [DateTime](#datetime). |
| `charts_count` | integer | Nº de gráficos no dashboard. Ex.: `12`. |

### ChartData

Dados de um gráfico devolvidos por uma consulta Cube.js. O formato de `result` varia conforme o tipo de gráfico.

| Campo | Tipo | Exemplo/Notas |
|---|---|---|
| `id` | string (uuid) | ver [ID](#id). |
| `name` | string | `Monthly Revenue`. |
| `type` | string (enum) | `line`, `area`, `bar`, `stackedBar`, `table`, `number`, `kpiPanel`, `pie`, `donut`, `funnel`. |
| `result` | object \| array | Dados do gráfico (formato depende do tipo): `number`/KPI → `{value}`; `table` → `{columns, rows}`; séries (`line`, `bar`, `pie`, …) → array de `{name, data}`. |
| `error` | string | Presente em vez de `result` quando a consulta falha. Ex.: `Failed to fetch chart data`. |

### Activity

Atividade/tarefa associada (opcionalmente) a um negócio — chamada, e-mail, reunião, mensagem, etc.

| Campo | Tipo | Exemplo/Notas |
|---|---|---|
| `id` | string (uuid) | ver [ID](#id). |
| `title` | string | `Call the customer back`. |
| `type` | string (enum) | `CALL`, `MAIL`, `SCHEDULE`, `TASK`, `MEETING`, `WHATSAPP`, `INSTAGRAM`. |
| `content` | string (nullable) | Corpo em texto livre. Para `WHATSAPP`/`INSTAGRAM` é o texto da mensagem; para `CALL`/`MAIL`/`TASK` são as notas. Ex.: `Discuss the renewal proposal.`. |
| `template_id` | string (uuid, nullable) | UUID do [MessageTemplate](#messagetemplate) (WhatsApp Official) anexado, ou `null` quando não há. |
| `completed` | boolean | `false`. |
| `completed_at` | string (date-time, nullable) | Data de conclusão. |
| `completed_by` | string (uuid, nullable) | Quem concluiu — ver [ID](#id). |
| `due_at` | string (date-time, nullable) | Prazo (alias público da coluna interna `to_check_at`). Ex.: `2024-01-20T14:00:00.000Z`. |
| `custom` | boolean | `true` se criada manualmente (API/UI); `false` se gerada automaticamente por template de etapa. |
| `no_show` | boolean | `false`. |
| `stage_id` | string (uuid) | Etapa — ver [ID](#id). |
| `created_at` | string (date-time) | ver [DateTime](#datetime). |
| `updated_at` | string (date-time, nullable) | Última atualização. |
| `deal` | object (nullable) | Negócio associado — ver [Deal](#deal). |
| `deal.id` | string (uuid) | ver [ID](#id). |
| `deal.status` | string (enum, nullable) | `OPEN`, `WON`, `LOST` — ver [DealStatus](#dealstatus). |
| `deal.origin_id` | string (uuid) | ver [Origin](#origin) / [ID](#id). |
| `deal.stage_id` | string (uuid) | ver [ID](#id). |
| `deal.contact` | object (nullable) | Contacto do negócio — ver [Contact](#contact). |
| `deal.contact.id` | string (uuid) | ver [ID](#id). |
| `deal.contact.name` | string (nullable) | `Jane Doe`. |

---

*Documento interno LL Mídia — modelo de dados da Clint API. Voltar ao [índice geral](./README.md).*
