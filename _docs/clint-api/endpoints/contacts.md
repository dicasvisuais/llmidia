# Contacts — Clint API

Gestão de contactos (pessoas) do CRM: listar, criar, consultar, atualizar, remover, gerir etiquetas (tags) e listar anexos.

> Fonte: OpenAPI oficial da Clint (`@clint-api/v1.0`) · Base `https://api.clint.digital` · Auth: header `api-token` · Documento interno LL Mídia.

## Visão geral

Os **contactos** são o registo central de pessoas no CRM da Clint. Cada contacto agrega dados de identificação (`name`, `email`, `instagram`), telefone normalizado (`fullPhone`), organização, etiquetas (`tags`) e campos personalizados (`fields`). Todas as operações desta categoria vivem em **`/v1`** (CRM base) e exigem sempre o header `api-token`.

A `tag_description` oficial desta categoria é simplesmente *"Operations related to managing contacts"* — não traz guia passo-a-passo (ao contrário de SMS/VOICE/Webhooks), pelo que não há fluxo especial a reproduzir aqui.

Esta categoria cobre oito operações: a **listagem paginada** com filtros ricos (por origem, nome, DDI, telefone, e-mail, IDs/nomes de tags e campos personalizados), o **CRUD** de um contacto individual (criar, obter, atualizar, remover), a **gestão de etiquetas** de um contacto (adicionar várias, remover uma) e a **listagem paginada de anexos** (documentos) associados ao contacto. Note-se que a atualização usa o método `POST` (e não `PUT`/`PATCH`) sobre `/v1/contacts/{id}`, e reaproveita o mesmo schema de corpo da criação (`ContactCreateSchema`).

A paginação segue o envelope `Paginated` (v1): a resposta traz `status`, `totalCount`, `page`, `totalPages`, `hasNext` e `hasPrevious`, com os itens em `data`. A listagem principal aceita `limit` (default 200, máx. 1000), `offset` e `page`; já a listagem de anexos usa um `limit` mais restrito (default 100, máx. 100) e `page`. IDs são UUID (schema `ID`) e datas são ISO-8601 (schema `DateTime`).

## Índice de endpoints

| Método | Caminho | O que faz |
|---|---|---|
| GET | `/v1/contacts` | Lista paginada de contactos, com filtros |
| POST | `/v1/contacts` | Cria um novo contacto |
| GET | `/v1/contacts/{id}` | Obtém um contacto por ID |
| POST | `/v1/contacts/{id}` | Atualiza um contacto |
| DELETE | `/v1/contacts/{id}` | Remove um contacto por ID |
| POST | `/v1/contacts/{id}/tags` | Adiciona etiquetas a um contacto |
| DELETE | `/v1/contacts/{id}/tags` | Remove uma etiqueta de um contacto |
| GET | `/v1/contacts/{id}/attachments` | Lista paginada de anexos do contacto |

---

## `GET` `/v1/contacts` — List contacts

**O que faz.** Recupera uma lista paginada de contactos. Suporta filtros por origem, nome, DDI, telefone, e-mail, etiquetas (por ID ou por nome) e campos personalizados.

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.** Nenhum.

**Parâmetros de query.**

| Nome | Tipo | Obrigatório | Default | Descrição |
|---|---|---|---|---|
| `limit` | integer (1–1000) | Não | 200 | Número máximo de linhas devolvidas. |
| `offset` | integer (>=0) | Não | 0 | Número de linhas ignoradas no resultado. |
| `page` | integer (>=1) | Não | 1 | Seleciona a página do resultado. |
| `origin_id` | string (uuid) | Não | — | Filtra por ID de origem. |
| `name` | string | Não | — | Filtra por nome do contacto. |
| `ddi` | string | Não | — | Filtra por DDI do contacto (ex.: `55`). |
| `phone` | string | Não | — | Filtra por telefone do contacto (ex.: `999999999`). |
| `email` | string | Não | — | Filtra por e-mail do contacto. |
| `tag_ids` | string | Não | — | Filtra por IDs de etiquetas usando operador OR. Separados por `,`. |
| `tag_names` | string | Não | — | Filtra por nomes de etiquetas usando operador OR. Separados por `,`. |
| `fields` | object (deepObject) | Não | — | Filtra por campos personalizados do contacto. Pode ser usado várias vezes, um por campo (estilo `deepObject`, ex.: `fields[cpf]=123`). |

**Corpo da requisição.** Nenhum.

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| 200 | Lista de contactos | `Paginated` + `data: Contact[]` |

**Exemplo — requisição**
```bash
curl -X GET "https://api.clint.digital/v1/contacts?limit=200&page=1&tag_names=tag1,tag2" \
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
      "created_at": "2020-01-01T14:15:00.000000+00:00",
      "updated_at": "2020-01-01T14:15:00.000000+00:00",
      "name": "Contact name",
      "email": "contact@email.com",
      "organization": "Organization name",
      "instagram": "Instagram ID",
      "tags": [
        {
          "id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8",
          "name": "Tag name",
          "color": "#f44336",
          "created_at": "2026-01-15T10:30:00.000Z"
        }
      ],
      "fields": {},
      "fullPhone": "+5548999999999"
    }
  ]
}
```

**Notas (inteligência LL Mídia).** `tag_ids` e `tag_names` usam **OR** (traz contactos que tenham qualquer uma das etiquetas indicadas), com valores separados por vírgula sem espaços. O parâmetro `fields` é `deepObject`: passe cada campo personalizado como `fields[chave]=valor`, repetível por campo. Use `page` para navegar de forma previsível; `offset` está disponível para saltos manuais mas raramente é preciso quando já usa `page`. `limit` máximo é 1000 — para exportações grandes, itere sobre `page` até `hasNext` ser `false`.

---

## `POST` `/v1/contacts` — Create contact

**O que faz.** Cria um novo contacto.

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.** Nenhum.

**Parâmetros de query.** Nenhum.

**Corpo da requisição.** Schema `ContactCreateSchema` (obrigatório).

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `name` | string | Não | Nome do contacto. |
| `ddi` | string | Não | Código de discagem internacional (ex.: `+55`). |
| `phone` | string | Não | Número de telefone (ex.: `48999999999`). |
| `email` | string | Não | E-mail do contacto. |
| `username` | string | Não | Identificador de Instagram (Instagram ID). |
| `fields` | object | Não | Campos personalizados como pares chave→valor (string). Suporta a chave especial `organization` como objeto de pares chave→valor. |

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| 201 | Contacto criado | `object` com `id` (`ID`) |

**Exemplo — requisição**
```bash
curl -X POST "https://api.clint.digital/v1/contacts" \
  -H "api-token: SUA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Contact name",
    "ddi": "+55",
    "phone": "48999999999",
    "email": "contact@email.com",
    "username": "Instagram ID",
    "fields": {
      "cpf": "12345678900",
      "organization": {
        "name": "Organization name"
      }
    }
  }'
```

**Exemplo — resposta (`201`)**
```json
{
  "id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8"
}
```

**Notas (inteligência LL Mídia).** A resposta devolve apenas o `id` do novo contacto — para obter o registo completo (com `tags`, `fullPhone`, etc.), faça de seguida um `GET /v1/contacts/{id}`. O telefone é fornecido em partes (`ddi` + `phone`); o CRM devolve depois a versão normalizada em `fullPhone`. Em `fields`, os valores são strings; a exceção é `organization`, que é um objeto de pares chave→valor.

---

## `GET` `/v1/contacts/{id}` — Get contact

**O que faz.** Recupera um único contacto pelo seu ID.

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.**

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string (uuid) | Sim | UUID do contacto. |

**Parâmetros de query.** Nenhum.

**Corpo da requisição.** Nenhum.

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| 200 | Objeto de contacto | `object` com `status` (integer) + `data` (`Contact`) |

**Exemplo — requisição**
```bash
curl -X GET "https://api.clint.digital/v1/contacts/8feade82-d77b-4e8b-9d35-fd43e972b5c8" \
  -H "api-token: SUA_API_KEY"
```

**Exemplo — resposta (`200`)**
```json
{
  "status": 200,
  "data": {
    "id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8",
    "created_at": "2020-01-01T14:15:00.000000+00:00",
    "updated_at": "2020-01-01T14:15:00.000000+00:00",
    "name": "Contact name",
    "email": "contact@email.com",
    "organization": "Organization name",
    "instagram": "Instagram ID",
    "tags": [
      {
        "id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8",
        "name": "Tag name",
        "color": "#f44336",
        "created_at": "2026-01-15T10:30:00.000Z"
      }
    ],
    "fields": {},
    "fullPhone": "+5548999999999"
  }
}
```

**Notas (inteligência LL Mídia).** Ao contrário da listagem (que envolve o `Paginated`), a resposta individual envolve o contacto num objeto `{ status, data }`. Um `id` inexistente devolve tipicamente `404`; um UUID malformado devolve `422`/`400` de validação.

---

## `POST` `/v1/contacts/{id}` — Update contact

**O que faz.** Atualiza um único contacto.

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.**

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string (uuid) | Sim | UUID do contacto. |

**Parâmetros de query.** Nenhum.

**Corpo da requisição.** Schema `ContactCreateSchema` (obrigatório) — os mesmos campos da criação.

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `name` | string | Não | Nome do contacto. |
| `ddi` | string | Não | Código de discagem internacional (ex.: `+55`). |
| `phone` | string | Não | Número de telefone (ex.: `48999999999`). |
| `email` | string | Não | E-mail do contacto. |
| `username` | string | Não | Identificador de Instagram (Instagram ID). |
| `fields` | object | Não | Campos personalizados como pares chave→valor (string). Suporta a chave especial `organization` como objeto de pares chave→valor. |

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| 200 | Contacto atualizado | `object` com `id` (`ID`) |

**Exemplo — requisição**
```bash
curl -X POST "https://api.clint.digital/v1/contacts/8feade82-d77b-4e8b-9d35-fd43e972b5c8" \
  -H "api-token: SUA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Novo nome",
    "email": "novo@email.com"
  }'
```

**Exemplo — resposta (`200`)**
```json
{
  "id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8"
}
```

**Notas (inteligência LL Mídia).** A atualização usa `POST` (não `PUT`/`PATCH`) e reaproveita o `ContactCreateSchema`. Envie apenas os campos que pretende alterar. A resposta devolve apenas o `id`; confirme o estado final com `GET /v1/contacts/{id}`. Para gerir etiquetas não use este endpoint — use os endpoints dedicados `/v1/contacts/{id}/tags`.

---

## `DELETE` `/v1/contacts/{id}` — Remove contact

**O que faz.** Remove um único contacto pelo seu ID.

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.**

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string (uuid) | Sim | UUID do contacto. |

**Parâmetros de query.** Nenhum.

**Corpo da requisição.** Nenhum.

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| 204 | No Content (removido com sucesso) | Sem corpo |

**Exemplo — requisição**
```bash
curl -X DELETE "https://api.clint.digital/v1/contacts/8feade82-d77b-4e8b-9d35-fd43e972b5c8" \
  -H "api-token: SUA_API_KEY"
```

**Exemplo — resposta (`204`)**
```text
HTTP/1.1 204 No Content
```

**Notas (inteligência LL Mídia).** Sucesso responde `204` sem corpo. A operação é destrutiva; garanta o `id` correto antes de chamar. Repetir a chamada sobre um contacto já removido tende a devolver `404`.

---

## `POST` `/v1/contacts/{id}/tags` — Add tags

**O que faz.** Adiciona etiquetas (tags) a um único contacto.

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.**

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string (uuid) | Sim | UUID do contacto. |

**Parâmetros de query.** Nenhum.

**Corpo da requisição.** Array de strings (obrigatório) — cada elemento é um nome de etiqueta.

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| (array) | array de string | Sim | Lista de etiquetas a adicionar (ex.: `"Tag"`). |

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| 204 | No Content (etiquetas adicionadas) | Sem corpo |

**Exemplo — requisição**
```bash
curl -X POST "https://api.clint.digital/v1/contacts/8feade82-d77b-4e8b-9d35-fd43e972b5c8/tags" \
  -H "api-token: SUA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '["Tag", "Lead quente"]'
```

**Exemplo — resposta (`204`)**
```text
HTTP/1.1 204 No Content
```

**Notas (inteligência LL Mídia).** O corpo é um **array de nomes de etiquetas** (strings), não de IDs. Pode adicionar várias de uma só vez. Sucesso responde `204` sem corpo — para ver as etiquetas resultantes, faça `GET /v1/contacts/{id}` e inspecione o campo `tags`. Para **remover** uma etiqueta, use `DELETE /v1/contacts/{id}/tags` (que aceita `tag_id` ou `tag_name`).

---

## `DELETE` `/v1/contacts/{id}/tags` — Remove tag

**O que faz.** Remove uma etiqueta de um contacto. Deve fornecer o `tag_id` **ou** o `tag_name` (exatamente um).

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.**

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string (uuid) | Sim | UUID do contacto. |

**Parâmetros de query.** Nenhum.

**Corpo da requisição.** Objeto (obrigatório). Forneça **um** dos dois campos (`oneOf`: `tag_id` ou `tag_name`).

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `tag_id` | string | Condicional | O ID da etiqueta a remover. |
| `tag_name` | string | Condicional | O nome da etiqueta a remover. |

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| 204 | No Content (etiqueta removida) | Sem corpo |
| 400 | Bad Request | `object` com `error` (ex.: `"You must provide either a tag id or a tag name"`) |

**Exemplo — requisição**
```bash
curl -X DELETE "https://api.clint.digital/v1/contacts/8feade82-d77b-4e8b-9d35-fd43e972b5c8/tags" \
  -H "api-token: SUA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "tag_id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8" }'
```

Alternativa por nome:
```bash
curl -X DELETE "https://api.clint.digital/v1/contacts/8feade82-d77b-4e8b-9d35-fd43e972b5c8/tags" \
  -H "api-token: SUA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "tag_name": "my-tag" }'
```

**Exemplo — resposta (`204`)**
```text
HTTP/1.1 204 No Content
```

**Exemplo — resposta (`400`)**
```json
{
  "error": "You must provide either a tag id or a tag name"
}
```

**Notas (inteligência LL Mídia).** A regra `oneOf` significa que deve enviar **exatamente um** identificador: `tag_id` ou `tag_name`. Se não enviar nenhum (ou ambos de forma inválida), a API responde `400` com a mensagem `"You must provide either a tag id or a tag name"`. Remove **uma** etiqueta por chamada — para várias, repita a operação.

---

## `GET` `/v1/contacts/{id}/attachments` — List contact attachments

**O que faz.** Recupera uma lista paginada de anexos (documentos) de um contacto específico. Cada anexo inclui um URL público de documento que pode ser usado diretamente para download ou apresentação.

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.**

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string (uuid) | Sim | UUID do contacto. |

**Parâmetros de query.**

| Nome | Tipo | Obrigatório | Default | Descrição |
|---|---|---|---|---|
| `limit` | integer (1–100) | Não | 100 | Número máximo de linhas devolvidas. |
| `page` | integer (>=1) | Não | 1 | Seleciona a página do resultado. |

**Corpo da requisição.** Nenhum.

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| 200 | Lista paginada de anexos do contacto | `Paginated` + `data: ContactAttachment[]` |
| 400 | Bad Request | `object` com `error` (ex.: `"Param contactId must be a valid UUID"`) |
| 404 | Contact not found | `object` com `error` (ex.: `"Contact not found"`) |

**Exemplo — requisição**
```bash
curl -X GET "https://api.clint.digital/v1/contacts/8feade82-d77b-4e8b-9d35-fd43e972b5c8/attachments?limit=100&page=1" \
  -H "api-token: SUA_API_KEY"
```

**Exemplo — resposta (`200`)**
```json
{
  "status": 200,
  "totalCount": 50,
  "page": 1,
  "totalPages": 1,
  "hasNext": false,
  "hasPrevious": false,
  "data": [
    {
      "id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8",
      "contact_id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8",
      "created_at": "2020-01-01T14:15:00.000000+00:00",
      "updated_at": "2020-01-01T14:15:00.000000+00:00",
      "uploaded_by": {
        "id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8",
        "first_name": "Maria",
        "last_name": "Silva"
      },
      "document": {
        "id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8",
        "name": "contract-signed",
        "file_name": "contract-signed.pdf",
        "url": "https://file.clint.digital/.../contract-signed.pdf",
        "extension": "pdf",
        "file_size": 204800
      }
    }
  ]
}
```

**Exemplo — resposta (`400`)**
```json
{
  "error": "Param contactId must be a valid UUID"
}
```

**Exemplo — resposta (`404`)**
```json
{
  "error": "Contact not found"
}
```

**Notas (inteligência LL Mídia).** Este endpoint usa uma paginação mais restrita do que a listagem principal: `limit` tem default **100** e máximo **100** (não 1000), e não aceita `offset` — apenas `page`. Cada item traz quem carregou o ficheiro (`uploaded_by`) e um objeto `document` com `url` público (formato URI) e `file_size` em **bytes**. Um UUID inválido no caminho devolve `400`; um contacto inexistente devolve `404`.

---

## Objetos relacionados

- [`Contact`](../03-modelo-de-dados.md#contact) — Registo de contacto: `id`, `created_at`, `updated_at`, `name`, `email`, `organization`, `instagram`, `tags[]`, `fields`, `fullPhone`.
- [`ContactCreateSchema`](../03-modelo-de-dados.md#contactcreateschema) — Corpo de criação/atualização: `name`, `ddi`, `phone`, `email`, `username`, `fields`.
- [`ContactAttachment`](../03-modelo-de-dados.md#contactattachment) — Anexo (documento) de um contacto: `id`, `contact_id`, `created_at`, `updated_at`, `uploaded_by`, `document`.
- [`Tag`](../03-modelo-de-dados.md#tag) — Etiqueta: `id`, `name`, `color` (`TagColor`), `created_at`.
- [`TagColor`](../03-modelo-de-dados.md#tagcolor) — Enum de cores hexadecimais da etiqueta (default `#f44336`).
- [`Fields`](../03-modelo-de-dados.md#fields) — Objeto de campos personalizados do contacto.
- [`FullPhone`](../03-modelo-de-dados.md#fullphone) — Telefone normalizado completo (ex.: `+5548999999999`).
- [`ID`](../03-modelo-de-dados.md#id) — Identificador UUID (string).
- [`DateTime`](../03-modelo-de-dados.md#datetime) — Data/hora ISO-8601.
- [`Paginated`](../03-modelo-de-dados.md#paginated) — Envelope de paginação v1: `status`, `totalCount`, `page`, `totalPages`, `hasNext`, `hasPrevious`.
