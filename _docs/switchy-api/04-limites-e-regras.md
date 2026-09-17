# Limites, restrições e regras de uso — Switchy API

> Fontes: <https://developers.switchy.io/docs/overview/index>, guias de criação e actualização de links, e <https://developers.switchy.io/docs/changelog/index> · Documento interno LL Mídia.

## Rate limits

Os limites aparecem no rodapé dos dois guias de link e são idênticos em ambos:

| Limite | Valor |
|---|---|
| Links por dia | **10.000** |
| Links por hora | **1.000** |

> A doc escreve "10.000" e "1.000" à europeia — são dez mil e mil.

Para limites superiores, a Switchy pede que se contacte o live chat.

**Inteligência LL Mídia:** a doc não indica se estes limites são por token, por workspace ou por conta, nem que código HTTP é devolvido quando se excede, nem se existem headers de `X-RateLimit-*`. `(inferência)` O mais provável é o limite ser por workspace, já que o token é por workspace. Numa integração de volume, convém implementar backoff próprio e não confiar em sinalização do servidor.

Também não há limite documentado para as **queries GraphQL** — só para criação de links.

## Domínios restritos

> ⚠️ **Os domínios `hi.switchy.io` e `swiy.io` só estão disponíveis através da API para integrações oficiais.**

Este aviso está em destaque (caixa vermelha) no guia de criação de links.

O problema prático: no tipo `Link`, o campo `domain` tem **`hi.switchy.io` como valor por omissão**. Ou seja, quem criar um link sem especificar `domain` cai exactamente no domínio restrito. Numa integração não-parceira, **passar sempre um `domain` próprio explicitamente**.

Quem quiser fazer uma integração oficial tem de falar com a Switchy pelo live chat.

## Uso pessoal vs. parceria — a regra de fundo

A doc distingue dois cenários, e é firme sobre isso:

**1. Integrar a Switchy na sua plataforma para os seus clientes** (ferramenta de email, agendador de redes sociais, plataforma de SMS, etc. — permitir que os utilizadores liguem a conta Switchy deles à ferramenta): a Switchy diz que gosta da ideia e pede contacto pelo chat para ajudar na integração.

**2. Integrar a Switchy na sua SaaS/plataforma para necessidades do próprio negócio:** a API é acessível **apenas para uso pessoal na própria conta**. Para usar em white-label dentro de uma ferramenta SaaS própria, é obrigatório falar com a Switchy para conhecer os **API Partner Plans**.

E a advertência, textual:

> **"Any account trying to overpass this limit without a granted access can get its API access restricted to preserve the security of the whole platform."**

Tradução prática: usar a API além do uso pessoal sem acordo de parceria pode levar à **restrição do acesso da conta**.

**Leitura LL Mídia:** para o nosso caso — encurtar e gerir os nossos próprios links, nos nossos próprios workspaces, com os nossos domínios — estamos claramente no "uso pessoal na própria conta" e não precisamos de parceria. A linha só seria cruzada se passássemos a oferecer o encurtador a terceiros dentro de um produto nosso.

## Versionamento e estabilidade

Do changelog oficial:

- "Our APIs are subject to change, we will try to provide retrocompatibility between our versions as much as possible." — as APIs estão sujeitas a alterações, com retrocompatibilidade "tanto quanto possível". Não é uma garantia.
- Sobre subscriptions: "We don't know if in the future, we will handle the versionning for Subscriptions" — não há compromisso de versionamento para subscriptions (que, de resto, ainda não existem).

Não há changelog datado, nem cabeçalho de versão, nem política de depreciação. A única marca de versão é o `v1` no path dos dois endpoints.

**Recomendação LL Mídia:** dado que não há sinalização de alterações, vale a pena correr periodicamente a introspecção autenticada e fazer diff contra o schema guardado — é a única forma de detectar mudanças no modelo de dados. Ver [`02-graphql.md`](./02-graphql.md).

## O que a API não permite fazer

Lacunas confirmadas na documentação pública:

| Operação | Estado |
|---|---|
| Criar link | ✅ REST — [`endpoints/criar-link.md`](./endpoints/criar-link.md) |
| Actualizar link | ✅ REST — [`endpoints/atualizar-link.md`](./endpoints/atualizar-link.md) |
| Ler dados | ✅ GraphQL |
| **Apagar link** | ❌ Não documentado |
| **Mutations GraphQL** | ❌ `mutationType: null` — "we are working hard to allow it" |
| **Subscriptions** | ❌ `subscriptionType: null` |
| **Webhooks** | ❌ Não mencionados em lado nenhum da doc |
| **Gerir pastas, domínios, pixels** | ❌ Sem endpoints de escrita; presumivelmente só leitura via GraphQL |
| **Estatísticas de cliques** | ⚠️ Não documentadas; `(inferência)` acessíveis via GraphQL, a confirmar com introspecção autenticada |

## Suporte

A Switchy não tem email de suporte a developers publicado — todo o contacto passa pelo **live chat**, disponível no site e dentro da app. É o canal indicado para:

- resposta garantida de staff da Switchy
- pedidos de suporte que envolvam dados sensíveis ou assuntos privados
- pedidos de funcionalidade
- feedback sobre os produtos Switchy
- obter um `Partner-Authorization`
- pedir limites de rate mais altos

Existe também um [grupo de Facebook da comunidade](https://www.facebook.com/groups/2357989371103633/?source_id=265902427493952) e o [repositório da documentação no GitHub](https://github.com/Switchy-io/api-docs), que aceita contribuições.
