# Actualizar link — `PUT /v1/links/:uniqId`

> Fonte: <https://developers.switchy.io/docs/guides/how-to-update-a-link> · Base `https://api.switchy.io` · Auth: header `Api-Authorization` · Documento interno LL Mídia.

## Visão geral

Único endpoint público para **actualizar** um link existente. Tal como na criação, existe porque o GraphQL não expõe mutations: *"Actually, we don't expose our GraphQL endpoint for update link. We are working hard to allow it. In the meantime, you can use this endpoint to do it."*

## Duas formas de identificar o link

A doc dá **duas rotas equivalentes**:

```
PUT https://api.switchy.io/v1/links/:uniqId
```

ou

```
PUT https://api.switchy.io/v1/links/by-domain/:domain/:id
```

| Rota | Quando usar |
|---|---|
| `/v1/links/:uniqId` | Quando se guardou o identificador interno do link (o `uniqId`) |
| `/v1/links/by-domain/:domain/:id` | Quando só se conhece o par domínio + slug — ou seja, o próprio link curto, `domain/id` |

**Inteligência LL Mídia:** a segunda rota é a mais útil na prática, porque permite actualizar um link a partir apenas do URL curto, sem manter uma tabela de correspondência. Notar que `uniqId` (identificador interno) e `id` (slug, `domainName/id`) são **coisas diferentes** — a doc usa `:LINK_ID` nos exemplos cURL, o que baralha os dois.

## Pedido

```
PUT https://api.switchy.io/v1/links/:uniqId
Content-Type: application/json
Api-Authorization: SEU_TOKEN
```

### Corpo

| Parâmetro | Descrição | Tipo | Por omissão |
|---|---|---|---|
| `link` | Link a criar para o redireccionamento | `Link` | sem valor por omissão |
| `autofill` | O servidor adiciona automaticamente os metadados Open Graph | boolean | `true` |

Na actualização, a tabela oficial do tipo `Link` **não marca nenhum campo como obrigatório** — nem sequer `url` — e **não lista** `domain` nem `id`. Faz sentido: o link já existe e é identificado pelo path. O tipo `Link` completo está em [`../03-modelo-de-dados.md`](../03-modelo-de-dados.md).

> ⚠️ **Semântica de PUT — não confirmada.** A doc não diz se um campo omitido é preservado ou reposto no valor por omissão. Sendo `PUT` (e não `PATCH`), o comportamento clássico seria **substituição total**, o que significaria que omitir `tags` as apagaria. `(inferência)` Até haver confirmação empírica, tratar como substituição total: **ler o link primeiro via GraphQL, alterar só o que é preciso, e reenviar o objecto completo.** É a abordagem segura.

## Exemplo mínimo (oficial)

```bash
curl 'https://api.switchy.io/v1/links/:LINK_ID' \
  -X 'PUT' \
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

> Mesmas erratas de shell do guia de criação: falta o `\` depois do header de autorização e o JSON do `-d` não vem entre aspas.

### Versão corrigida (inteligência LL Mídia)

```bash
curl -X PUT "https://api.switchy.io/v1/links/by-domain/links.suamarca.com/campanha-x" \
  -H 'Content-Type: application/json' \
  -H "Api-Authorization: $SWITCHY_TOKEN" \
  -d '{
    "link": {
      "url": "https://exemplo.com/nova-pagina",
      "title": "Novo título",
      "tags": ["campanha-x", "actualizado"]
    }
  }'
```

## Exemplo completo (oficial)

```bash
curl 'https://api.switchy.io/v1/links/:LINK_ID' \
  -X 'PUT' \
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

> Diferença face ao exemplo completo de criação: aqui **não há `domain` nem `id`** no corpo — confirmando que a identidade do link vem do path e não é alterável por esta via. `(inferência)` Não há forma documentada de mudar o slug de um link já criado.

## Resposta

**Não documentada** pela Switchy — nem corpo, nem código HTTP, nem erros (incluindo o caso de link inexistente). Verificar empiricamente.

## Limites

Os mesmos da criação: **10.000 links/dia, 1.000 links/hora**. Ver [`../04-limites-e-regras.md`](../04-limites-e-regras.md).
