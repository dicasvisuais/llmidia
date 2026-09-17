# Organizations — Clint API

Recurso para consultar e atualizar organizações (empresas/contas) associadas ao CRM.

> Fonte: OpenAPI oficial da Clint (`@clint-api/v1.0`) · Base `https://api.clint.digital` · Auth: header `api-token` · Documento interno LL Mídia.

## Visão geral

As **Organizations** representam as organizações (empresas/contas) geridas no CRM da Clint. A descrição oficial da categoria na spec é sucinta: *"Operations related to managing organizations"*. Esta fatia expõe duas operações, ambas em **v1** e ambas centradas num identificador único (`id`, UUID) de uma organização já existente.

O recurso vive no CRM base (`/v1`), tal como contactos, negócios e tags. Cada organização tem um conjunto de campos fixos (`id`, `created_at`, `updated_at`, `name`) e um objeto livre `fields` para dados personalizados (pares chave/valor). A fatia **não** inclui operações de criação (`create`) nem de listagem (`list`) — apenas leitura por ID e atualização por ID.

Não há guia passo-a-passo (`tag_description` sem tutorial SMS/VOICE/Webhooks), nem feature flags/escopos declarados nesta fatia. Aplicam-se apenas os comportamentos gerais da API: autenticação por header `api-token` e os códigos de erro comuns (ver [Convenções](../02-convencoes.md)).

Autenticação obrigatória em todas as chamadas: header `api-token: <SUA_API_KEY>` (ver [Autenticação](../01-autenticacao.md)).

## Índice de endpoints

| Método | Caminho | O que faz |
|---|---|---|
| GET | `/v1/organizations/{id}` | Obtém uma organização pelo seu ID |
| POST | `/v1/organizations/{id}` | Atualiza uma organização existente |

---

## `GET` `/v1/organizations/{id}` — Get organization

**O que faz.** Recupera uma única organização pelo seu ID (UUID).

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.**

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | `string` (uuid) | Sim | UUID da organização a consultar. Ex.: `8feade82-d77b-4e8b-9d35-fd43e972b5c8`. |

**Parâmetros de query.** Nenhum.

**Corpo da requisição.** Nenhum.

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `200` | An organization object | `{ status: integer, data: `[`Organization`](../03-modelo-de-dados.md#organization)` }` |

Estrutura da resposta `200`:

| Campo | Tipo | Descrição |
|---|---|---|
| `status` | `integer` | Estado da resposta (ex.: `200`). |
| `data` | [`Organization`](../03-modelo-de-dados.md#organization) | Objeto da organização. |

Campos de [`Organization`](../03-modelo-de-dados.md#organization):

| Campo | Tipo | Descrição |
|---|---|---|
| `id` | `string` (uuid) | Identificador único da organização. |
| `created_at` | `string` (date-time) | Data/hora de criação (ISO-8601). |
| `updated_at` | `string` (date-time) | Data/hora da última atualização (ISO-8601). |
| `name` | `string` | Nome da organização. |
| `fields` | `object` | Campos personalizados (objeto livre de pares chave/valor). |

**Exemplo — requisição**
```bash
curl -X GET "https://api.clint.digital/v1/organizations/8feade82-d77b-4e8b-9d35-fd43e972b5c8" \
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
    "name": "Organization name",
    "fields": {}
  }
}
```

**Notas (inteligência LL Mídia).** Operação de leitura pura (idempotente, sem efeitos colaterais). O `id` tem de ser um UUID válido de uma organização existente — caso contrário espere `404`. Sem `api-token` válido, `401`. O objeto `fields` é um mapa livre: a spec não fixa as suas chaves, pelo que o consumidor deve tratá-lo defensivamente (pode vir vazio `{}`). Esta fatia não expõe endpoint de listagem, por isso é preciso já ter o `id` da organização (obtido por outro fluxo, ex.: a partir de um contacto/negócio associado).

---

## `POST` `/v1/organizations/{id}` — Update organization

**O que faz.** Atualiza uma única organização existente, identificada pelo `id`.

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.**

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | `string` (uuid) | Sim | UUID da organização a atualizar. Ex.: `8feade82-d77b-4e8b-9d35-fd43e972b5c8`. |

**Parâmetros de query.** Nenhum.

**Corpo da requisição.** Schema [`OrganizationCreateSchema`](../03-modelo-de-dados.md#organizationcreateschema) (obrigatório, `application/json`).

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `name` | `string` | Não | Nome da organização. Ex.: `Organization name`. |
| `fields` | `object` (valores `string`) | Não | Campos personalizados: objeto de pares chave/valor, em que cada valor é uma `string`. |

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `200` | Organization updated | `{ id: `[`ID`](../03-modelo-de-dados.md#id)` }` |

Estrutura da resposta `200`:

| Campo | Tipo | Descrição |
|---|---|---|
| `id` | `string` (uuid) | Identificador único da organização atualizada. |

**Exemplo — requisição**
```bash
curl -X POST "https://api.clint.digital/v1/organizations/8feade82-d77b-4e8b-9d35-fd43e972b5c8" \
  -H "api-token: SUA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Organization name",
    "fields": {
      "segmento": "Educação",
      "cnpj": "12.345.678/0001-90"
    }
  }'
```

**Exemplo — resposta (`200`)**
```json
{
  "id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8"
}
```

**Notas (inteligência LL Mídia).** Apesar de o schema do corpo se chamar `OrganizationCreateSchema`, esta operação é de **atualização** de um registo já existente (o `id` vem no caminho, não no corpo). Nenhum campo do corpo é marcado como obrigatório na spec, mas o `requestBody` em si é obrigatório — envie pelo menos o(s) campo(s) que pretende alterar. Atenção à diferença de tipos do `fields`: na resposta de leitura (`Organization`) é um objeto livre, mas no corpo de escrita (`OrganizationCreateSchema`) os valores são explicitamente `string` — evite passar números/objetos aninhados. A resposta devolve apenas o `id`; para confirmar os valores gravados, faça um `GET` subsequente. `id` inexistente → `404`; sem `api-token` → `401`; corpo inválido → `422`.

---

## Objetos relacionados

- [`Organization`](../03-modelo-de-dados.md#organization) — Organização do CRM (`id`, `created_at`, `updated_at`, `name`, `fields`).
- [`OrganizationCreateSchema`](../03-modelo-de-dados.md#organizationcreateschema) — Corpo de escrita da organização (`name`, `fields` com valores `string`).
- [`ID`](../03-modelo-de-dados.md#id) — Identificador UUID (string).
- [`DateTime`](../03-modelo-de-dados.md#datetime) — Data/hora em formato ISO-8601.
- [`Fields`](../03-modelo-de-dados.md#fields) — Objeto livre de campos personalizados.
