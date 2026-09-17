# Criar link — `POST /v1/links/create`

> Fonte: <https://developers.switchy.io/docs/guides/how-to-create-a-link> · Base `https://api.switchy.io` · Auth: header `Api-Authorization` · Documento interno LL Mídia.

## Visão geral

Único endpoint público para **criar** um link curto no Switchy. Existe porque o GraphQL não expõe mutations: citação da doc, *"Actually, we don't expose our GraphQL endpoint for creating link. We are working hard to allow it. In the meantime, you can use this endpoint to do it."*

> ⚠️ **Os domínios `hi.switchy.io` e `swiy.io` só estão disponíveis através da API para integrações oficiais.** Como `hi.switchy.io` é o valor por omissão de `domain`, uma integração não-parceira deve passar **sempre** um `domain` próprio explicitamente. Ver [`../04-limites-e-regras.md`](../04-limites-e-regras.md).

## Pedido

```
POST https://api.switchy.io/v1/links/create
Content-Type: application/json
Api-Authorization: SEU_TOKEN
```

### Corpo

| Parâmetro | Descrição | Tipo | Por omissão |
|---|---|---|---|
| `link` | Link a criar para o redireccionamento | `Link` | sem valor por omissão |
| `autofill` | O servidor adiciona automaticamente os metadados Open Graph | boolean | `true` |

Campos obrigatórios dentro de `link`: **`url`** e **`domain`**. Tudo o resto é opcional. O tipo `Link` completo, com todos os campos e valores por omissão, está em [`../03-modelo-de-dados.md`](../03-modelo-de-dados.md).

## Exemplo mínimo (oficial)

```bash
curl 'https://api.switchy.io/v1/links/create' \
  -H 'Content-Type: application/json' \
  -H 'Api-Authorization: YOUR_TOKEN_HERE' \
  -d \
  {
    "link": {
      "title": "",
      "description": "",
      "url": "https://example.com/",
      "pixels": [],
      "showGDPR": false,
      "extraOptionsLinkRotator": [],
      "extraOptionsGeolocations": [],
      "tags": []
    }
  }
  --compressed
```

> ⚠️ O exemplo oficial está **mal formado como comando shell**: falta o `\` depois do header de autorização, o JSON do `-d` não vem entre aspas e o exemplo "mínimo" omite o `domain`, que a própria doc lista como obrigatório. Usar a versão corrigida abaixo.

### Versão corrigida (inteligência LL Mídia)

```bash
curl -X POST 'https://api.switchy.io/v1/links/create' \
  -H 'Content-Type: application/json' \
  -H "Api-Authorization: $SWITCHY_TOKEN" \
  -d '{
    "link": {
      "url": "https://exemplo.com/pagina",
      "domain": "links.suamarca.com",
      "tags": ["campanha-x"]
    }
  }'
```

Sem `id`, o slug é gerado aleatoriamente (`randomId()`). Sem `title`/`description`/`image` e com `autofill` no valor por omissão (`true`), o servidor vai buscar os metadados Open Graph à página de destino.

## Exemplo completo (oficial)

```bash
curl 'https://api.switchy.io/v1/links/create' \
  -H 'Content-Type: application/json' \
  -H 'Api-Authorization: YOUR_TOKEN_HERE' \
  -d \
  {
    "link": {
      "title": "My title",
      "description": "My description",
      "url": "https://example.com/",
      "image": "https://example.com/my.jpg",
      "pixels": [
        {
          "createdAt": "2021-01-30T08:03:34.568+00:00",
          "id": "pixel id",
          "platform": "twitter",
          "title": "pixel title",
          "value": "tixel value",
          "workspaceId": 5972
        }
      ],
      "showGDPR": true,
      "extraOptionsGeolocations": [
        {
          "url": "https://github.com",
          "value": "AO"
        }
      ],
      "tags": [
        "mytag"
      ],
      "domain": "hi.swutchy.io",
      "id": "my-slug",
      "folderId": 10313,
      "favicon": "https://example.com/favicon_144x144.png",
      "linkExpiration": {
        "enable": true,
        "timezone": 4,
        "url": "http://google.com",
        "end": "2021-12-29T10:03:51.421Z"
      },
      "deepLinkingEnable": true,
      "note": "My Notes",
      "passwordProtect": {
        "enable": true,
        "password": "123"
      }
    }
  }
  --compressed
```

> Erratas do exemplo oficial, para não serem copiadas às cegas:
> - `"domain": "hi.swutchy.io"` — gralha por `hi.switchy.io` (que, de resto, é restrito).
> - `"value": "tixel value"` — gralha por "pixel value".
> - O `id` do pixel no exemplo é a string literal `"pixel id"`, quando a tabela de tipos diz que é um **UUID**.

## Campos que este endpoint aceita e o de actualização não

Comparando as duas tabelas oficiais, só a criação documenta:

| Campo | Nota |
|---|---|
| `domain` | **Obrigatório** na criação; ausente da tabela de actualização |
| `id` | O slug; ausente da tabela de actualização |

E na criação, `url` está marcado **Mandatory**; na actualização, não.

## Resposta

**A doc oficial não documenta a resposta** — nem o corpo, nem o código HTTP, nem os erros. `(inferência)` Espera-se um `2xx` com o objecto do link criado (incluindo o `id`/slug gerado quando não foi fornecido), mas isto tem de ser verificado empiricamente na primeira chamada real.

## Limites

| Limite | Valor |
|---|---|
| Links por dia | 10.000 |
| Links por hora | 1.000 |

Para limites superiores, contactar o live chat da Switchy. Detalhes em [`../04-limites-e-regras.md`](../04-limites-e-regras.md).
