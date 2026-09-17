# Schema GraphQL real — introspecção autenticada

> Obtido por introspecção com token de workspace em **2026-09-17**. Spec crua: [`./schema-graphql.json`](./schema-graphql.json). Documento interno LL Mídia.

Este é o schema **real**, não o documentado — a Switchy nunca publicou a lista de tabelas. Substitui as inferências de [`02-graphql.md`](./02-graphql.md).

## Resumo

| | |
|---|---|
| Tabelas base | **8** |
| Campos em `query_root` | 17 (8 tabelas + 8 `_by_pk` + 1 `_aggregate`) |
| Mutations | **nenhuma** |
| Subscriptions | **nenhuma** |

```
UTMTemplates  domains  folders  linkScripts  links  pixels  tokens  workspaces
```

Só `workspaces` tem `_aggregate`. Todas têm `_by_pk` **excepto** `links`.

> ⚠️ `links_by_pk` existe mas **não aceita `uniq`** como argumento (`no such argument "uniq" is expected`). Para ir buscar um link, usar `links(where: {...})`.

---

## ❌ Não existem dados de cliques por data

Varrimento de todo o schema por `click`, `stat`, `analytic`, `event`, `visit`, `hit`, `report`, `daily`, `history`, `log` — resultado:

- **Tipos** com esses nomes: **nenhum**
- **Campos** com esses nomes: apenas `links.clicks` e `links.clicksLimit`

`links.clicks` é um **`Int` cumulativo** — o total desde sempre. Não há tabela de eventos, não há timestamps de clique, não há dimensão temporal em lado nenhum.

A API REST também não ajuda: só tem os dois endpoints de escrita documentados. Qualquer `GET` a `/v1/links/...` devolve `404 Cannot GET`, incluindo `/stats`, `/analytics` e `/clicks`.

**Conclusão: a analítica por data que o dashboard do Switchy mostra na UI não está exposta em nenhuma API pública.**

---

## Tabelas

### `links` (30 campos)

| Campo | Tipo | Nota |
|---|---|---|
| `uniq` | Int | identificador interno (o `:uniqId` do REST) |
| `id` | String | o slug (`domain/id`) |
| `domain` | String | |
| `url` | String | destino |
| **`clicks`** | **Int** | **total cumulativo — sem dimensão temporal** |
| `clicksLimit` | jsonb | `{enable, limit}` — limite de cliques |
| `createdDate` | timestamptz | |
| `title` / `description` / `image` / `favicon` | String | Open Graph |
| `name` | String | não documentado no REST |
| `note` | String | |
| `tags` | jsonb | |
| `folderId` | Int | |
| `pixels` | jsonb | |
| `showGDPR` | Boolean | |
| `linkScripts` | jsonb | |
| `linkExpiration` | jsonb | |
| `passwordProtect` | jsonb | |
| `masking` | Boolean | cloaking |
| `deepLinkingEnable` | Boolean | |
| `caseSensitive` | Boolean | **não documentado no REST** |
| `smartpageUniq` | String | **não documentado** — ligação a smartpage |
| `userUid` | String | |
| `extraOptionsGeolocations` | jsonb | |
| `extraOptionsLinkRotator` | jsonb | |
| `extraOptionsDeviceRotations` | jsonb | **não documentado no REST** |
| `extraOptionsOSRotations` | jsonb | **não documentado no REST** |
| `workspace` | workspaces | relação |

> Quatro campos existem no schema mas **não constam da doc REST**: `caseSensitive`, `smartpageUniq`, `extraOptionsDeviceRotations`, `extraOptionsOSRotations`. `(inferência)` Provavelmente aceites no corpo do create/update, mas não confirmado.

### `workspaces` (14 campos)

`id:Int`, `name`, `companyName`, `ownerId`, `createdDate`, `trialDate`, `removedDate`, `activateRgpdEverywhere:Boolean`, `rgpdUrl`, `noIndex:Boolean`, `smartpagesCount:Int` + relações `links`, `folders`, `pixels`.

### `domains`

`name`, `redirect`, `ownerId`, `workspaceId:Int`, `createdDate`, `removedDate` — confirma o **soft delete** por `removedDate`.

### `folders`

`id:Int`, `name`, `order:Int`, `type`, `ownerUid`, `workspaceId:Int`, `createdAt`.

### `pixels`

`id:String`, `platform:String`, `title`, `value`, `workspaceId:Int`, `createdAt`.

> `platform` é `String` no schema, não um enum — a lista de 11 valores da doc REST é convenção, não constrangimento da base de dados.

### `linkScripts`

`id:Int`, `name`, `script:String`, `html:String`, `htmlOptions:jsonb`, `enable:Boolean`, `workspaceId:Int`, `createdDate`.

> Confirma que `links.linkScripts` guarda **ids** destes registos.

### `UTMTemplates`

`id:Int`, `name`, `source`, `medium`, `campaign`, `content`, `term`, `workspaceId:Int`, `createdAt`.

> Recurso **inexistente na documentação oficial**: templates de UTM reutilizáveis.

### `tokens`

`workspaceId:Int`, `createdAt` + relação `workspace`. Não expõe o valor do token.

---

## Receitas

**Um link por domínio + slug:**

```graphql
query {
  links(where: {domain: {_eq: "links.exemplo.com"}, id: {_eq: "meu-slug"}}) {
    uniq id url title clicks createdDate folderId tags
  }
}
```

**Top links por cliques:**

```graphql
query {
  links(order_by: {clicks: desc}, limit: 20) { id domain url clicks createdDate }
}
```

**Links de uma pasta, criados depois de uma data:**

```graphql
query {
  links(
    where: {folderId: {_eq: 88329}, createdDate: {_gte: "2025-09-01"}}
    order_by: {createdDate: desc}
  ) { id url clicks createdDate }
}
```

**Domínios activos** (excluindo apagados):

```graphql
query { domains(where: {removedDate: {_is_null: true}}) { name redirect createdDate } }
```
