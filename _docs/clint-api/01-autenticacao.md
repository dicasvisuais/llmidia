# Autenticação — Clint API

Toda a Clint API é autenticada por um único header obrigatório — `api-token` — que deve acompanhar **todas** as chamadas, em `/v1` e `/v2`.

> Fonte: OpenAPI oficial da Clint (`@clint-api/v1.0`) · Base `https://api.clint.digital` · Auth: header `api-token` · Documento interno LL Mídia.

## Visão geral

A Clint API usa **autenticação por chave de API**. Não há fluxo OAuth, sessão ou cookie: cada requisição transporta a sua credencial no header HTTP `api-token`. A chave identifica o *owner* (a conta) e determina o que a requisição pode ler ou escrever.

O header é **obrigatório em todas as operações**, sem exceção — tanto nos endpoints `/v1` (CRM base: contactos, organizações, negócios, tags, origens, utilizadores, etc.) como nos endpoints `/v2` (omnichannel e mensageria: canais, templates, chats, mensagens, dashboards, **SMS**, **VOICE**, **Webhooks** e **Activities**). Na especificação OpenAPI a autenticação não é declarada como um esquema global; é expressa **operação a operação** através do parâmetro de header partilhado `headerParamXAPIKey` (nome `api-token`), que está marcado como `required: true` em cada endpoint. Na prática, o comportamento é o mesmo para toda a API: sem `api-token` válido, a chamada não passa.

Alguns recursos `/v2` são ainda protegidos por **feature flags** ao nível da conta e por **escopos** ao nível da própria chave (ver [Escopos e feature flags](#escopos-de-chave-e-feature-flags)). Quando a conta ou a chave não têm o acesso necessário, a API responde `403` mesmo que o `api-token` seja tecnicamente válido.

## O header `api-token`

| Propriedade | Valor |
|---|---|
| Nome | `api-token` |
| Local | Header HTTP |
| Tipo | `string` |
| Obrigatório | Sim (em todas as chamadas, `/v1` e `/v2`) |
| Descrição (spec) | *API Token* |

Formato do header numa requisição HTTP:

```http
api-token: SUA_API_KEY
```

O valor é enviado tal como está, sem prefixo `Bearer` e sem qualquer codificação adicional. Não vai na query string nem no corpo — apenas no header.

## Exemplo — requisição autenticada

O exemplo canónico de uma chamada autenticada é listar contactos (`GET /v1/contacts`). Basta acrescentar o header `api-token`:

```bash
curl -X GET "https://api.clint.digital/v1/contacts" \
  -H "api-token: SUA_API_KEY"
```

Com uma chave válida e com acesso ao recurso, a API responde `200 OK` com o envelope paginado dos contactos. O mesmo header aplica-se, sem alterações, a qualquer outro endpoint — por exemplo, um `POST /v2/sms`, um `PUT /v2/webhooks` ou um `GET /v2/activities`.

Para requisições com corpo (`POST`, `PUT`, `DELETE` com payload), acrescenta-se também o `Content-Type`, mantendo sempre o `api-token`:

```bash
curl -X POST "https://api.clint.digital/v2/activities" \
  -H "api-token: SUA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "deal_id": "550e8400-e29b-41d4-a716-446655440000",
    "title": "Call the customer back",
    "type": "CALL"
  }'
```

## Respostas de erro de autenticação

### `401` — token ausente ou inválido

Quando o header `api-token` **não é enviado** ou contém uma **chave inválida**, a API responde:

| Código | Significado | Corpo |
|---|---|---|
| `401` | *Authentication error - invalid or missing api-token* | Não especificado na spec |

É o erro a tratar primeiro em qualquer integração: se aparece de forma consistente, o problema está na credencial (ausente, mal copiada, expirada ou revogada), não nos parâmetros do pedido.

### `403` — feature flag ou escopo em falta

Quando o `api-token` é válido, mas a **conta não tem a feature flag** necessária ou a **chave não tem o escopo** exigido pela operação, a API responde `403` com uma mensagem fixa:

| Código | Significado | Corpo |
|---|---|---|
| `403` | Acesso vedado por feature flag/escopo | `status` + `message` |

**Exemplo — resposta (`403`)**

```json
{
  "status": 403,
  "message": "This feature is not available for your account."
}
```

A distinção prática é importante: `401` é um problema de **autenticação** (quem és tu?), enquanto `403` é um problema de **autorização** (podes usar este recurso?). Reenviar a mesma chave não resolve um `403` — é preciso ativar a feature na conta e/ou emitir uma chave com o escopo adequado.

## Escopos de chave e feature flags

Além da autenticação base, alguns recursos `/v2` aplicam duas camadas adicionais de controlo de acesso:

- **Feature flags (por conta).** Habilitam um conjunto de funcionalidades na conta. Exemplo documentado na spec: a flag **`ACTIVITIES_API`**, exigida por todos os endpoints de *Activities* (`/v2/activities`).
- **Escopos (por chave de API).** Delimitam o que uma chave concreta pode fazer dentro de um recurso já habilitado. Os escopos mencionados na spec para *Activities* são:
  - **`activities:read`** — leitura (`GET /v2/activities`, `GET /v2/activities/{id}`);
  - **`activities:write`** — escrita (`POST /v2/activities`, atualização, `DELETE /v2/activities/{id}`, `POST /v2/activities/{id}/complete`).

As duas camadas são cumulativas: para criar uma atividade, a conta precisa da flag `ACTIVITIES_API` **e** a chave precisa do escopo `activities:write`. Se qualquer uma faltar, a resposta é `403` com a mensagem `"This feature is not available for your account."` (ver acima). Ao desenhar uma integração, atribua a cada chave apenas os escopos de que necessita — uma chave usada só para relatórios de atividades deve ter `activities:read`, não `activities:write`.

> A documentação de *Activities* detalha o gating por endpoint: ver [`./endpoints/activities.md`](./endpoints/activities.md).

## Onde obter a chave

**(Nota prática / inferência — não consta da especificação OpenAPI.)** A chave de API é emitida e gerida no **painel da Clint** (área de configurações/integrações da conta). A spec descreve apenas *como* a chave é usada nas chamadas (o header `api-token`), não *onde* é criada nem como os escopos e feature flags são atribuídos. Confirme o local exato e o processo de emissão/rotação diretamente no painel ou junto do suporte da Clint.

## Boas práticas de segurança

- **Nunca versione a chave.** Não a comite no Git nem a inclua em HTML, JavaScript de front-end ou qualquer artefacto público. Guarde-a em variáveis de ambiente ou num gestor de segredos.
- **Uso exclusivamente server-side.** O `api-token` identifica o *owner* e não deve ser exposto no browser nem em apps distribuídas ao cliente. Todas as chamadas à Clint API devem partir do seu back-end.
- **Rotação periódica.** Rode a chave com regularidade e imediatamente após qualquer suspeita de fuga. Como não há sessão a expirar, uma chave comprometida permanece válida até ser revogada.
- **Menor privilégio.** Prefira chaves com o mínimo de escopos necessários (ex.: `activities:read` quando só precisa de ler). Isso limita o impacto caso a chave seja exposta.
- **Não a transmita em URLs.** O header nunca deve migrar para a query string — evita que a chave fique registada em logs de servidor, proxies ou histórico.

## Ligações relacionadas

- [Índice geral](./README.md) — mapa de toda a documentação interna da Clint API.
- [Convenções](./02-convencoes.md) — paginação, filtros, formatos de campo e catálogo de erros comuns.
- [Modelo de dados](./03-modelo-de-dados.md) — schemas partilhados (`Activity`, `PaginatedV2`, `DateTime`, `ID`, etc.).
- [Endpoints — Activities](./endpoints/activities.md) — recurso que ilustra o gating por feature flag (`ACTIVITIES_API`) e por escopo (`activities:read` / `activities:write`).
