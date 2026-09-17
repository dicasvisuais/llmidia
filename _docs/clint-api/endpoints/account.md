# Account — Clint API

Recurso de leitura que devolve a definição dos campos (fields) e grupos de campos configurados na conta, para os objetos `DEAL`, `CONTACT` e `ORGANIZATION`.

> Fonte: OpenAPI oficial da Clint (`@clint-api/v1.0`) · Base `https://api.clint.digital` · Auth: header `api-token` · Documento interno LL Mídia.

## Visão geral

A categoria **Account** expõe metadados de configuração da própria conta. Na prática, permite descobrir dinamicamente quais os campos (padrão e personalizados) que existem para negócios (`DEAL`), contactos (`CONTACT`) e organizações (`ORGANIZATION`), bem como os grupos aos quais esses campos pertencem.

Este é um recurso da API **v1** (CRM base). Não requer feature flags nem escopos especiais para além da autenticação por `api-token`. É útil para construir integrações que precisam de mapear rótulos e tipos de campos antes de criar ou atualizar negócios, contactos e organizações — em vez de assumir nomes de campos fixos, a integração pode consultar `/v1/account/fields` e adaptar-se à configuração real da conta.

A fatia desta categoria não traz `tag_description` (guia passo-a-passo), pelo que não há fluxo adicional a reproduzir aqui. Existe uma única operação, descrita em baixo.

## Índice de endpoints

| Método | Caminho | O que faz |
|---|---|---|
| GET | `/v1/account/fields` | Lista os campos e grupos de campos da conta (DEAL, CONTACT, ORGANIZATION) |

---

## `GET` `/v1/account/fields` — List fields

**O que faz.** Recupera uma lista dos campos configurados na conta. Para cada entrada, devolve os `groups` (grupos de campos) e os `fields` (definição de cada campo) para os três domínios: `DEAL`, `CONTACT` e `ORGANIZATION`.

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.** Nenhum.

**Parâmetros de query.** Nenhum.

**Parâmetros de header.**

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `api-token` | string | Sim | API Token da conta. Enviado no header de todas as chamadas. |

**Corpo da requisição.** Nenhum.

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `200` | A list of fields | Objeto com `data`: array de [`AccountFields`](../03-modelo-de-dados.md#accountfields) |

O corpo da resposta `200` é um objeto com a propriedade `data`, um array de objetos `AccountFields`. Cada `AccountFields` tem a seguinte estrutura:

**`AccountFields`**

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `groups` | object | Não | Grupos de campos, organizados por domínio (`DEAL`, `CONTACT`, `ORGANIZATION`). |
| `groups.DEAL` | object (map string→string) | Não | Mapa de grupos de campos de negócios. Chave e valor são strings. |
| `groups.CONTACT` | object (map string→string) | Não | Mapa de grupos de campos de contactos. Chave e valor são strings. |
| `groups.ORGANIZATION` | object (map string→string) | Não | Mapa de grupos de campos de organizações. Chave e valor são strings. |
| `fields` | object | Não | Definição dos campos, organizada por domínio (`DEAL`, `CONTACT`, `ORGANIZATION`). |
| `fields.DEAL` | object | Não | Definição de um campo de negócio. |
| `fields.DEAL.type` | string | Não | Tipo do campo. Exemplo: `TEXT`. |
| `fields.DEAL.group` | string | Não | Grupo a que o campo pertence. Exemplo: `default`. |
| `fields.DEAL.label` | string | Não | Rótulo do campo. Exemplo: `notes`. |
| `fields.CONTACT` | object | Não | Definição de um campo de contacto. |
| `fields.CONTACT.type` | string | Não | Tipo do campo. Exemplo: `TEXT`. |
| `fields.CONTACT.group` | string | Não | Grupo a que o campo pertence. Exemplo: `default`. |
| `fields.CONTACT.label` | string | Não | Rótulo do campo. Exemplo: `notes`. |
| `fields.ORGANIZATION` | object | Não | Definição de um campo de organização. |
| `fields.ORGANIZATION.type` | string | Não | Tipo do campo. Exemplo: `TEXT`. |
| `fields.ORGANIZATION.group` | string | Não | Grupo a que o campo pertence. Exemplo: `default`. |
| `fields.ORGANIZATION.label` | string | Não | Rótulo do campo. Exemplo: `notes`. |

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
        "DEAL": {
          "default": "Padrão"
        },
        "CONTACT": {
          "default": "Padrão"
        },
        "ORGANIZATION": {
          "default": "Padrão"
        }
      },
      "fields": {
        "DEAL": {
          "type": "TEXT",
          "group": "default",
          "label": "notes"
        },
        "CONTACT": {
          "type": "TEXT",
          "group": "default",
          "label": "notes"
        },
        "ORGANIZATION": {
          "type": "TEXT",
          "group": "default",
          "label": "notes"
        }
      }
    }
  ]
}
```

**Notas (inteligência LL Mídia).**
- É uma operação de leitura (`GET`) e idempotente: pode ser chamada repetidamente sem efeitos colaterais. Ideal para pré-carregar/mapear a configuração da conta no arranque de uma integração.
- Os objetos `groups.DEAL`, `groups.CONTACT` e `groups.ORGANIZATION` são mapas livres (`additionalProperties: string`): as chaves e os valores dependem da configuração da conta, pelo que a integração deve iterar dinamicamente em vez de assumir chaves fixas.
- Os valores `TEXT`, `default` e `notes` nos campos são apenas exemplos da spec; a resposta real refletirá os campos efetivamente configurados na conta (incluindo campos personalizados).
- Use este endpoint em conjunto com os endpoints de negócios, contactos e organizações: primeiro descubra aqui os `label`/`type`/`group` disponíveis, depois construa os payloads de criação/atualização com base nessa configuração.
- Erros comuns transversais à API: `401` (sem/`api-token` inválido). Ver [Autenticação](../01-autenticacao.md) e [Convenções](../02-convencoes.md).

---

## Objetos relacionados

- [`AccountFields`](../03-modelo-de-dados.md#accountfields) — Definição dos campos (`fields`) e grupos de campos (`groups`) da conta, para os domínios `DEAL`, `CONTACT` e `ORGANIZATION`.
