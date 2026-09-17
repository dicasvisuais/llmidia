# Convenções — Clint API

Regras transversais a toda a Clint API: versões, paginação, filtros, campos personalizados, formatos de dados e erros. Leia isto antes de usar qualquer endpoint.

> Fonte: OpenAPI oficial da Clint (`@clint-api/v1.0`) · Base `https://api.clint.digital` · Auth: header `api-token` · Documento interno LL Mídia.

Ligações úteis: [Índice geral](./README.md) · [Autenticação](./01-autenticacao.md) · [Modelo de dados](./03-modelo-de-dados.md) · Endpoints: [`account`](./endpoints/account.md), [`contacts`](./endpoints/contacts.md).

---

## (a) Base URL e versões (v1 vs v2)

- **Base URL (produção):** `https://api.clint.digital`
- **Autenticação:** header obrigatório `api-token: <SUA_API_KEY>` em **todas** as chamadas (ver [`01-autenticacao.md`](./01-autenticacao.md)).
- Todos os caminhos são versionados por prefixo: `/v1/...` ou `/v2/...`.

| Versão | O que vive aqui | Exemplos de recursos |
|---|---|---|
| **`/v1`** | CRM base (a espinha dorsal do produto) | `contacts`, `deals`, `tags`, `organizations`, `origins`, `groups`, `account/fields` |
| **`/v2`** | Mensageria/omnichannel, analítica e automações | `channel-accounts`, `chats`, `messages`, `dashboards`, `activities`, SMS, voz, webhooks |

Regras práticas:

- **Não misture versões** num mesmo recurso: os contactos vivem em `/v1/contacts`; as conversas desse contacto vivem em `/v2/chats`. São APIs coordenadas, mas com envelopes e convenções ligeiramente diferentes (ver paginação, abaixo).
- Muitos recursos `/v2` estão **atrás de feature flags e escopos** de chave (ver secção (h)). Um `/v1` disponível não garante que o `/v2` correspondente esteja ativo na conta.
- Os campos e enumerações mantêm sempre os **nomes em inglês** exatamente como na spec (ex.: `origin_id`, `tag_ids`, `WHATSAPP_OFFICIAL`). Não traduza chaves.

---

## (b) Paginação

Endpoints de listagem devolvem um **envelope de paginação** que embrulha os resultados numa propriedade `data` (array). O formato do envelope depende da versão.

### Envelope `Paginated` (v1)

Chaves em **camelCase**.

| Campo | Tipo | Descrição |
|---|---|---|
| `status` | integer | Estado da resposta (ex.: `200`) |
| `totalCount` | integer | Total de itens **considerando os filtros atuais** |
| `page` | integer | Página atual |
| `totalPages` | integer | Total de páginas com os filtros atuais |
| `hasNext` | boolean | Existe página seguinte |
| `hasPrevious` | boolean | Existe página anterior |
| `data` | array | Os registos da página |

```json
{
  "status": 200,
  "totalCount": 50,
  "page": 1,
  "totalPages": 10,
  "hasNext": true,
  "hasPrevious": false,
  "data": [ /* ... */ ]
}
```

> Nota: um **GET de item único** (ex.: `GET /v1/contacts/{id}`) **não** é paginado — devolve `{ "status": 200, "data": { ... } }`, apenas com `status` + `data`.

### Envelope `PaginatedV2` (v2)

Mesma semântica, mas chaves em **snake_case**. Use este envelope ao consumir listagens de `/v2` (chats, mensagens, atividades, etc.).

| Campo (v2) | Equivalente v1 | Tipo |
|---|---|---|
| `status` | `status` | integer |
| `total_count` | `totalCount` | integer |
| `page` | `page` | integer |
| `total_pages` | `totalPages` | integer |
| `has_next` | `hasNext` | boolean |
| `has_previous` | `hasPrevious` | boolean |

```json
{
  "status": 200,
  "total_count": 50,
  "page": 1,
  "total_pages": 10,
  "has_next": true,
  "has_previous": false,
  "data": [ /* ... */ ]
}
```

### Parâmetros de query de paginação

| Parâmetro | Tipo | Default | Limites | Descrição |
|---|---|---|---|---|
| `limit` | integer | `200` | `min 1`, `max 1000` | Número máximo de linhas devolvidas |
| `offset` | integer | `0` | `>= 0` | Número de linhas saltadas no resultado |
| `page` | integer | `1` | `>= 1` | Página do resultado a devolver |

Notas de inteligência LL Mídia:

- O `limit` de referência é **default 200, máximo 1000**. Alguns endpoints impõem tetos mais baixos: p.ex. `GET /v1/contacts/{id}/attachments` usa `limit` com **máximo/​default 100**, e há endpoints com **máximo/​default 200**. Verifique sempre a tabela de query do endpoint específico em `./endpoints/<slug>.md`.
- Pode paginar por **`offset`** (deslocamento absoluto) **ou** por **`page`** (índice de página). Escolha uma estratégia e seja consistente; misturar `offset` e `page` na mesma chamada torna o comportamento ambíguo.
- Para saber quando parar, **prefira `hasNext`/`has_next`** em vez de calcular à mão a partir de `totalCount`/`total_count` — o total muda se os filtros (ou os dados) mudarem entre chamadas.
- `totalCount`/`total_count` e `totalPages`/`total_pages` refletem **os filtros aplicados**, não o total absoluto da conta.

---

## (c) Filtros

### Filtro por tags — `tag_ids` e `tag_names`

Disponíveis, por exemplo, em `GET /v1/contacts`. Ambos aceitam **vários valores separados por vírgula** e combinam-nos com **operador `OR`** (o registo é incluído se corresponder a *qualquer* uma das tags).

| Parâmetro | Tipo | Operador | Descrição |
|---|---|---|---|
| `tag_ids` | string | `OR` | IDs de tag (UUID) separados por `,` |
| `tag_names` | string | `OR` | Nomes de tag separados por `,` |

```bash
# Contactos com QUALQUER uma destas tags (por ID)
curl -X GET "https://api.clint.digital/v1/contacts?tag_ids=8feade82-d77b-4e8b-9d35-fd43e972b5c8,99999999-d77b-4e8b-9d35-fd43e972b999" \
  -H "api-token: SUA_API_KEY"

# Equivalente por nome
curl -X GET "https://api.clint.digital/v1/contacts?tag_names=tag1,tag2,tag3" \
  -H "api-token: SUA_API_KEY"
```

Notas:

- É um `OR`, **não** um `AND`. A API não expõe um filtro nativo de interseção de tags; se precisar de "tem a tag A **e** a tag B", filtre por uma e faça a interseção do lado do cliente.
- Não deixe espaços à volta das vírgulas.
- Prefira `tag_ids` sempre que possível (estável); `tag_names` é conveniente mas quebra se a tag for renomeada.

### Filtro por campos personalizados — `fields`

O parâmetro `fields` filtra por **campos personalizados** do registo. É um **objeto** serializado em estilo `deepObject` e é **repetível** — pode usá-lo uma vez por cada campo que queira filtrar. A descrição na spec: *"Filter by contact fields. Can be used multiple times for each field."*

Serialização `deepObject`: cada campo vai como `fields[<label>]=<valor>`.

```bash
# Filtrar contactos por dois campos personalizados (cpf e city)
curl -X GET "https://api.clint.digital/v1/contacts?fields[cpf]=12345678900&fields[city]=Lisboa" \
  -H "api-token: SUA_API_KEY"
```

Notas:

- Os **`<label>`** válidos são os `label` dos campos personalizados da conta — obtenha-os em `GET /v1/account/fields` (ver secção (d)).
- O mesmo mecanismo de filtro por `fields` existe noutros recursos que suportam campos personalizados (ex.: `deals`: *"Filter by deal fields..."*). Consulte o endpoint concreto.

---

## (d) Campos personalizados (`Fields` e `AccountFields`)

A Clint permite definir **campos personalizados** por tipo de registo. Aparecem em dois sítios distintos, com schemas distintos:

### `Fields` — valores num registo

Nos objetos de recurso (ex.: `Contact`, `Deal`, `Organization`), a propriedade `fields` é do tipo `Fields`, um **objeto livre** de pares chave→valor com os valores dos campos personalizados desse registo. As chaves correspondem aos `label` definidos na conta.

Ao **criar/​atualizar** (ex.: `ContactCreateSchema`), `fields` é um objeto de strings, podendo aninhar campos de entidades relacionadas (ex.: `organization`):

```json
{
  "name": "Contact name",
  "fields": {
    "cpf": "12345678900",
    "organization": { "cnpj": "00000000000191" }
  }
}
```

### `AccountFields` — definição/​catálogo dos campos da conta

O schema `AccountFields` descreve **quais** campos personalizados existem e como estão organizados, por tipo de entidade (`DEAL`, `CONTACT`, `ORGANIZATION`):

- **`groups`** — para cada entidade, um mapa de `id → nome` dos grupos de campos.
- **`fields`** — para cada entidade, a definição do campo:
  - `type` (ex.: `TEXT`),
  - `group` (ex.: `default`),
  - `label` (ex.: `notes`) — este é o nome a usar como chave em `Fields` e no filtro `fields[...]`.

### `GET /v1/account/fields` — listar os campos da conta

**O que faz.** Devolve a definição de todos os campos personalizados da conta.

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho / query.** Nenhum.

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `200` | Lista de campos | `{ "data": [ AccountFields ] }` |

**Exemplo — requisição**

```bash
curl -X GET "https://api.clint.digital/v1/account/fields" \
  -H "api-token: SUA_API_KEY"
```

**Exemplo — resposta (`200`)**

```json
{
  "data": [
    {
      "groups": {
        "DEAL": { "8feade82-...": "default" },
        "CONTACT": { "8feade82-...": "default" },
        "ORGANIZATION": { "8feade82-...": "default" }
      },
      "fields": {
        "DEAL": { "type": "TEXT", "group": "default", "label": "notes" },
        "CONTACT": { "type": "TEXT", "group": "default", "label": "notes" },
        "ORGANIZATION": { "type": "TEXT", "group": "default", "label": "notes" }
      }
    }
  ]
}
```

**Nota (inteligência LL Mídia).** Chame este endpoint uma vez e mantenha um mapa `label → tipo` em cache. Use-o para (1) validar as chaves que envia em `fields` ao criar/​atualizar registos e (2) montar os filtros `fields[<label>]=...`. Detalhes completos em [`./endpoints/account.md`](./endpoints/account.md).

---

## (e) Telefone e DDI (`FullPhone`)

- Ao **escrever** um contacto/​negócio, o DDI e o número vão em campos **separados**: `ddi` (ex.: `+55`) e `phone` (ex.: `48999999999`). Nos filtros de listagem também são separados (`ddi`, `phone`).
- Ao **ler**, o objeto devolve `fullPhone` (schema `FullPhone`) — uma **string única em formato E.164** com o DDI e o número concatenados, ex.: `+5548999999999`.

```json
{ "ddi": "+55", "phone": "48999999999" }   // entrada (create/update)
```
```json
{ "fullPhone": "+5548999999999" }          // saída (leitura)
```

Nota: guarde do seu lado o par `ddi` + `phone` (ou reconstrua a partir de `fullPhone`); não assuma um DDI implícito.

---

## (f) Datas (`DateTime`)

- Todas as datas/​horas usam o schema `DateTime`: **string em ISO-8601** com fuso horário.
- Exemplo canónico da spec: `2020-01-01T14:15:00.000000+00:00`. Outros campos usam a variante com `Z`, ex.: `2024-01-15T09:00:00.000Z`. Ambos são ISO-8601 válidos.
- Envie sempre datas em ISO-8601 com offset/​`Z` explícito; **prefira UTC** para evitar ambiguidade. Não envie timestamps epoch nem formatos locais.

---

## (g) IDs (UUID)

- Todos os identificadores de recurso são **UUID** (string, `format: uuid`), ex.: `8feade82-d77b-4e8b-9d35-fd43e972b5c8` (schema `ID`).
- Os parâmetros de caminho `{id}` e os filtros por ID (ex.: `origin_id`, `tag_ids`) esperam UUIDs válidos. Um UUID mal formado costuma devolver `400` (ex.: `"Param contactId must be a valid UUID"`).

---

## Enumerações e valores por omissão

Alguns campos são **enumerações fechadas**. Dois exemplos transversais:

- **`TagColor`** — cor de uma tag. Valor por omissão `#f44336`. Apenas são aceites os 16 hex da paleta fixa: `#f44336`, `#e91e63`, `#9c27b0`, `#673ab7`, `#3f51b5`, `#2196f3`, `#03a9f4`, `#00bcd4`, `#009688`, `#4caf50`, `#8bc34a`, `#faa200`, `#ff9800`, `#ff5722`, `#795548`, `#607d8b`. Qualquer outro valor é rejeitado.
- **`DealStatus`** — estado de um negócio. Valor por omissão `OPEN`; valores possíveis: `OPEN`, `WON`, `LOST`.

Envie sempre o valor **exatamente** como no enum (maiúsculas/​minúsculas incluídas). Ver mais enumerações em [`03-modelo-de-dados.md`](./03-modelo-de-dados.md).

---

## (h) Erros

Códigos de estado transversais:

| Código | Significado | Causa típica |
|---|---|---|
| `400` | Bad Request | Parâmetro inválido (ex.: UUID mal formado, corpo incompleto). Corpo: `{ "error": "..." }` |
| `401` | Authentication error | `api-token` em falta ou inválido |
| `403` | Forbidden | **Feature flag** desativada ou **escopo** da chave em falta |
| `404` | Not Found | Recurso inexistente ou fora da conta. Corpo: `{ "error": "..." }` |
| `422` | Unprocessable Entity | Erro de validação do corpo/​parâmetros |

### O `403` de feature flag

Vários recursos (sobretudo em `/v2`) estão **atrás de feature flags**. Se a funcionalidade não estiver ativa na conta, a chamada devolve `403`. Mensagens observadas na spec:

- `"This feature is not available for your account."` (mensagem genérica de feature flag)
- `"The attendance API feature is not enabled for your account"` (chats/​atendimento)

Estas negações **não** se resolvem trocando de chave nem repetindo o pedido — a flag tem de ser ativada na conta (contacte o suporte/​gestor Clint).

### Escopos (scopes) da chave

Além da flag, alguns endpoints exigem que a **API key tenha o escopo** correspondente. Exemplos presentes na spec:

- **Atividades** (`/v2/activities`): feature flag `ACTIVITIES_API` **e** escopo `activities:read` (leitura) ou `activities:write` (escrita).
- **Histórico de negócios**: escopo `deals:read`. Sem ele, `403 — "API key missing the deals:read scope"`.

Ou seja, um `403` pode significar **feature flag em falta** *ou* **escopo em falta na chave** — mensagens distintas para causas distintas. Confirme ambos: a flag ativa na conta e a chave emitida com os escopos certos (ver [`01-autenticacao.md`](./01-autenticacao.md)).

---

## (i) Boas práticas de integração

- **Header em todas as chamadas.** Nunca omita `api-token`; um `401` cega o resto do fluxo.
- **Trate 403 como configuração, não como transitório.** Distinga "feature flag" de "escopo em falta" pela mensagem e resolva na origem — não faça retry cego.
- **Retry só para transitórios.** Reserve retentativas (com backoff) para erros de rede/​5xx; `400`/`401`/`403`/`404`/`422` são determinísticos e não beneficiam de retry.
- **Pagine defensivamente.** Itere por `hasNext`/`has_next`; não deduza o fim a partir do total. Fixe uma estratégia (`offset` **ou** `page`).
- **Respeite os tetos de `limit`** por endpoint (200/​1000, ou 100 nalguns). Pedir acima do máximo não "desbloqueia" mais linhas.
- **IDs e datas canónicos.** UUID para IDs; ISO-8601 (de preferência UTC) para datas.
- **Campos personalizados a partir do catálogo.** Descubra os `label` válidos em `GET /v1/account/fields` antes de escrever `fields` ou de filtrar por `fields[...]`; não invente chaves.
- **Não misture versões** dentro do mesmo recurso e assuma envelopes diferentes: camelCase (`Paginated`, v1) vs snake_case (`PaginatedV2`, v2).
- **Idempotência.** A API não expõe chave de idempotência; para criações, guarde do seu lado uma referência (ex.: em `fields`) para detetar/​evitar duplicados.

---

## Objetos relacionados

- [`Paginated`](./03-modelo-de-dados.md#paginated) — envelope de paginação v1 (camelCase).
- [`PaginatedV2`](./03-modelo-de-dados.md#paginatedv2) — envelope de paginação v2 (snake_case).
- [`Fields`](./03-modelo-de-dados.md#fields) — valores de campos personalizados num registo.
- [`AccountFields`](./03-modelo-de-dados.md#accountfields) — catálogo/​definição dos campos da conta.
- [`FullPhone`](./03-modelo-de-dados.md#fullphone) — telefone completo em E.164.
- [`DateTime`](./03-modelo-de-dados.md#datetime) — data/​hora ISO-8601.
- [`ID`](./03-modelo-de-dados.md#id) — identificador UUID.
- [`TagColor`](./03-modelo-de-dados.md#tagcolor) — paleta fixa de cores de tag.
- [`DealStatus`](./03-modelo-de-dados.md#dealstatus) — estado de um negócio.
