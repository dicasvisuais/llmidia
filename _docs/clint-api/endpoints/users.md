# Users — Clint API

Recurso de leitura dos utilizadores (membros) da conta na Clint, expostos como registos com `id`, `email`, `first_name` e `last_name`.

> Fonte: OpenAPI oficial da Clint (`@clint-api/v1.0`) · Base `https://api.clint.digital` · Auth: header `api-token` · Documento interno LL Mídia.

## Visão geral

A categoria **Users** agrupa as operações de gestão de utilizadores da conta. Na prática, e conforme a fatia da spec, expõe apenas leitura: listar todos os utilizadores (com paginação) e obter um utilizador específico pelo seu `id`. É útil para descobrir quem são os membros/operadores associados à conta — por exemplo, para depois relacionar responsáveis a negócios, atribuições ou atividades noutras categorias.

Estes endpoints vivem em **`/v1`** (CRM base) e usam o envelope de paginação padrão `Paginated` na listagem. Cada utilizador é representado pelo schema `User`, composto por `id` (UUID), `email`, `first_name` e `last_name`.

Todas as chamadas exigem o header `api-token`. A fatia da spec define apenas a resposta `200` para ambas as operações; o comportamento de erro transversal da API (autenticação em falta, recurso inexistente, validação) não é especificado nesta fatia — consulte as [Convenções](../02-convencoes.md).

A `tag_description` desta fatia é apenas descritiva (`"Operations related to managing users"`), sem guia passo-a-passo (ao contrário de SMS/VOICE/Webhooks), pelo que não há procedimento adicional a reproduzir aqui.

## Índice de endpoints

| Método | Caminho | O que faz |
|---|---|---|
| GET | `/v1/users` | Lista paginada de utilizadores da conta |
| GET | `/v1/users/{id}` | Obtém um utilizador específico pelo `id` |

---

## `GET` `/v1/users` — Listar utilizadores

**O que faz.** Devolve uma lista paginada dos utilizadores da conta.

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.** Nenhum.

**Parâmetros de query.**

| Nome | Tipo | Obrigatório | Default | Descrição |
|---|---|---|---|---|
| `limit` | integer | Não | `200` | Número máximo de linhas devolvidas (mínimo `1`, máximo `1000`). |
| `offset` | integer | Não | `0` | Número de linhas ignoradas no resultado (mínimo `0`). |
| `page` | integer | Não | `1` | Seleciona a página do resultado (mínimo `1`). |

**Corpo da requisição.** Nenhum.

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `200` | Lista de utilizadores | `Paginated` + `data: array<User>` |

O envelope `Paginated` contém os campos: `status` (integer), `totalCount` (integer), `page` (integer), `totalPages` (integer), `hasNext` (boolean) e `hasPrevious` (boolean). O campo `data` é um array de objetos `User`.

**Exemplo — requisição**
```bash
curl -X GET "https://api.clint.digital/v1/users?limit=200&offset=0&page=1" \
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
      "email": "User e-mail",
      "first_name": "User first name",
      "last_name": "User last name"
    }
  ]
}
```

**Notas (inteligência LL Mídia).** Endpoint `/v1`, paginado com o envelope `Paginated`. Combine `limit` com `offset` **ou** `page` para percorrer resultados; use `hasNext`/`hasPrevious` e `totalPages` para saber quando parar, em vez de assumir um número fixo de páginas. `limit` está limitado a `1000` — para contas grandes, itere. O header `api-token` é obrigatório em todas as chamadas; a fatia da spec documenta apenas a resposta `200`.

---

## `GET` `/v1/users/{id}` — Obter utilizador

**O que faz.** Devolve um único utilizador identificado pelo seu `id`.

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.**

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string (uuid) | Sim | UUID do utilizador a obter. |

**Parâmetros de query.** Nenhum.

**Corpo da requisição.** Nenhum.

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `200` | Objeto de utilizador | `{ status: integer, data: User }` |

O corpo da resposta contém `status` (integer) e `data`, um objeto `User` com os campos `id` (UUID), `email`, `first_name` e `last_name`.

**Exemplo — requisição**
```bash
curl -X GET "https://api.clint.digital/v1/users/8feade82-d77b-4e8b-9d35-fd43e972b5c8" \
  -H "api-token: SUA_API_KEY"
```

**Exemplo — resposta (`200`)**
```json
{
  "status": 200,
  "data": {
    "id": "8feade82-d77b-4e8b-9d35-fd43e972b5c8",
    "email": "User e-mail",
    "first_name": "User first name",
    "last_name": "User last name"
  }
}
```

**Notas (inteligência LL Mídia).** O `id` no caminho é um UUID (string). Ao contrário da listagem, este endpoint não é paginado nem aceita parâmetros de query. O header `api-token` é obrigatório. A fatia da spec documenta apenas a resposta `200`; para o comportamento de erro (id inexistente ou mal formado, token em falta) consulte as [Convenções](../02-convencoes.md).

---

## Objetos relacionados

- [`User`](../03-modelo-de-dados.md#user) — utilizador da conta (`id`, `email`, `first_name`, `last_name`).
- [`Paginated`](../03-modelo-de-dados.md#paginated) — envelope de paginação padrão v1 (`status`, `totalCount`, `page`, `totalPages`, `hasNext`, `hasPrevious`).
- [`ID`](../03-modelo-de-dados.md#id) — identificador UUID (string) usado no campo `id`.
