# Modelo de dados — Switchy API

> Fontes: <https://developers.switchy.io/docs/guides/how-to-create-a-link> e <https://developers.switchy.io/docs/guides/how-to-update-a-link> · Documento interno LL Mídia.

Os tipos abaixo são os do **corpo REST** de criação e actualização de links. São os únicos tipos formalmente documentados pela Switchy. Os tipos do GraphQL (`workspaces`, `domains`, `links`, …) só se obtêm por introspecção autenticada — ver [`02-graphql.md`](./02-graphql.md).

---

## Envelope do pedido

| Parâmetro | Descrição | Tipo | Valor por omissão |
|---|---|---|---|
| `link` | Link a criar para o redireccionamento | `Link` | sem valor por omissão |
| `autofill` | O servidor adiciona automaticamente os metadados Open Graph | `boolean` | `true` |

```json
{
  "link": { "...": "..." },
  "autofill": true
}
```

> `autofill` é um irmão de `link`, **não** um campo dentro de `link`. Com `autofill` activo (o comportamento por omissão), o servidor vai buscar `title`, `description` e `image` à página de destino.

---

## Tipo `Link`

### Identidade e destino

| Campo | Descrição | Tipo | Por omissão |
|---|---|---|---|
| `url` | **Obrigatório** (na criação). URL de destino do redireccionamento | String | — |
| `domain` | **Obrigatório** (na criação). Nome do domínio do link | String | `hi.switchy.io` |
| `id` | Slug/alias — é o id do link (`domainName/id`) | String | `randomId()` |

> ⚠️ Na **actualização**, `url` deixa de estar marcado como obrigatório e os campos `domain` e `id` não constam da tabela oficial — o link já existe e é identificado pelo path. Ver [`endpoints/atualizar-link.md`](./endpoints/atualizar-link.md).
>
> ⚠️ `domain` tem por omissão `hi.switchy.io`, **mas** esse domínio (e `swiy.io`) está reservado a integrações oficiais. Ver [`04-limites-e-regras.md`](./04-limites-e-regras.md).

### Open Graph / apresentação

| Campo | Descrição | Tipo | Por omissão |
|---|---|---|---|
| `title` | Usado no título Open Graph | String | `null` (preenchível via `autofill`) |
| `description` | Usado na descrição Open Graph | String | `null` (preenchível via `autofill`) |
| `image` | Usado na descrição Open Graph; tem de ser um URL | String | `null` (preenchível via `autofill`) |
| `favicon` | — | String | `null` |

### Organização

| Campo | Descrição | Tipo | Por omissão |
|---|---|---|---|
| `folderId` | — | Int | `null` |
| `tags` | Usado para procurar e ordenar no dashboard | String[] | `[]` |
| `note` | Notas visíveis no painel de administração | String | `null` |

### Tracking e retargeting

| Campo | Descrição | Tipo | Por omissão |
|---|---|---|---|
| `pixels` | Usado para pixelar o link curto, para efeitos de retargeting | `Pixel[]` | `[]` |
| `showGDPR` | Mostra o popup de GDPR se existirem pixels | Boolean | `false` |

### Comportamento do redireccionamento

| Campo | Descrição | Tipo | Por omissão |
|---|---|---|---|
| `extraOptionsGeolocations` | Redireccionamento por geolocalização | jsonb | `[]` |
| `extraOptionsLinkRotator` | Rotação de links | jsonb | `[]` |
| `deepLinkingEnable` | Activa o deeplink, se for possível | Boolean | `false` |
| `masking` | Activa o cloaking do link | Boolean | `false` |
| `linkExpiration` | Define a data em que o link expira e o destino de redireccionamento | `LinkExpiration` | `null` |
| `passwordProtect` | Mostra o popup de protecção por palavra-passe | `PasswordProtect` | `null` |

### Scripts

| Campo | Descrição | Tipo | Por omissão |
|---|---|---|---|
| `linkScripts` | Lista de scripts que serão adicionados a um link se `linkScriptEnable` for `true` | Int[] | `null` |
| `linkScriptEnable` | Activa os `linkScripts` associados ao link | Boolean | `false` |

> A doc não documenta o tipo `extraOptionsGeolocations` para lá de `jsonb`, mas o exemplo oficial mostra a forma: `[{"url": "https://github.com", "value": "AO"}]` — `value` é um código de país ISO 3166-1 alpha-2 (`AO` = Angola) e `url` o destino para esse país. `(inferência)` Sobre `extraOptionsLinkRotator` não há exemplo algum; nos exemplos oficiais aparece sempre como `[]`.
>
> `linkScripts` é um array de **Int** — são identificadores de scripts já criados na conta, não o código dos scripts. `(inferência)` Os ids obtêm-se presumivelmente via GraphQL.

---

## Tipo `Pixel`

| Campo | Descrição | Tipo | Por omissão |
|---|---|---|---|
| `platform` | **Obrigatório.** A que plataforma pertence o pixel | `Platform` | — |
| `value` | **Obrigatório.** Valor do pixel | String | — |
| `id` | É o id do pixel na base de dados | UUID | `randomId()` |
| `workspaceId` | — | integer | — |

Exemplo oficial de um item de `pixels` (inclui campos não listados na tabela, como `createdAt` e `title`):

```json
{
  "createdAt": "2021-01-30T08:03:34.568+00:00",
  "id": "pixel id",
  "platform": "twitter",
  "title": "pixel title",
  "value": "tixel value",
  "workspaceId": 5972
}
```

> `(inferência)` `title` e `createdAt` aparecem no exemplo mas não na tabela de tipos — provavelmente são opcionais e ecoam a estrutura do pixel tal como está guardado. Na prática, para criar um link com um pixel existente, `platform` + `value` deve chegar.

### Enum `Platform`

```ts
type Platform =
  | 'linkedin'
  | 'facebook'
  | 'gtm'
  | 'quora'
  | 'pinterest'
  | 'twitter'
  | 'ga'
  | 'bing'
  | 'nexus'
  | 'adroll'
  | 'adwords';
```

11 valores. Notar que são **strings em minúsculas**, e que `ga` (Google Analytics), `gtm` (Google Tag Manager) e `adwords` (Google Ads) são três entradas distintas. Não existe valor para TikTok nesta lista.

---

## Tipo `LinkExpiration`

| Campo | Descrição | Tipo | Por omissão |
|---|---|---|---|
| `enable` | **Obrigatório.** A expiração do link está activa | Boolean | — |
| `end` | **Obrigatório.** Data de fim da expiração — exemplo: `2021-11-11T06:59:01.499Z` | String | — |
| `url` | **Obrigatório.** URL para onde o link é redireccionado depois de expirar | UUID | `randomId()` |
| `timezone` | **Obrigatório.** Fuso horário | Int | — |

```json
{
  "enable": true,
  "timezone": 4,
  "url": "http://google.com",
  "end": "2021-12-29T10:03:51.421Z"
}
```

> ⚠️ A tabela oficial diz que `url` é do tipo `UUID` com omissão `randomId()` — isso é claramente um **erro de copy-paste na doc da Switchy**: a descrição e o exemplo mostram um URL (String). Tratar como String.
>
> `timezone` é um **Int** e a doc não diz que unidade é. `(inferência)` Pelo exemplo (`4`), parece ser um offset em horas, não um identificador IANA. Confirmar empiricamente antes de usar em produção.

---

## Tipo `PasswordProtect`

| Campo | Descrição | Tipo | Por omissão |
|---|---|---|---|
| `enable` | **Obrigatório.** A protecção por palavra-passe está activa | Boolean | — |
| `password` | **Obrigatório.** Palavra-passe do link protegido | String | — |

```json
{ "enable": true, "password": "123" }
```

> A palavra-passe viaja em texto simples no corpo do pedido (sobre HTTPS). Não é um mecanismo de segurança forte — serve para travar acesso casual.
