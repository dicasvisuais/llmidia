# VOICE — Clint API

Chamadas de voz automatizadas (voice broadcast): carregue um áudio uma vez e reutilize-o para disparar chamadas que reproduzem essa mensagem gravada.

> Fonte: OpenAPI oficial da Clint (`@clint-api/v1.0`) · Base `https://api.clint.digital` · Auth: header `api-token` · Documento interno LL Mídia.

## Visão geral

A categoria **VOICE** permite fazer chamadas telefónicas automatizadas que reproduzem um áudio pré-gravado (mensagem de voz). Faz parte da mensageria omnichannel da Clint (endpoints `/v2`) e partilha a mesma funcionalidade da categoria SMS: exige que a *feature* **SMS & Voice** esteja activada na sua conta pela equipa da Clint. Sem essa activação, as chamadas devolvem `403` com a mensagem `"This feature is not available for your account."`.

O fluxo é sempre em **dois passos**, conforme o guia oficial da spec:

**1. Carregar o áudio (apenas uma vez por ficheiro)** — `POST /v2/voice/audios`

Envie um link HTTPS público que aponte para um ficheiro **MP3** (`.mp3` — outros formatos não são suportados). A Clint descarrega o ficheiro, armazena-o e devolve um `audio_id`:

```json
{ "audio_url": "https://yoursite.com/message.mp3" }
```

Resposta:

```json
{ "audio_id": "api-audio-xxxx", "reused": false }
```

Guarde o `audio_id`. Carregar exactamente o mesmo ficheiro outra vez devolve o **mesmo** `audio_id` com `"reused": true` — não há upload duplicado nem armazenamento em dobro.

**2. Fazer a chamada** — `POST /v2/voice`

Passe o telefone de destino e o `audio_id` do passo anterior:

```json
{ "phone": "+5511999998888", "audio_id": "api-audio-xxxx" }
```

Resposta:

```json
{ "success": true, "message_id": "def456", "status": "QUEUED" }
```

A chamada devolve imediatamente com estado `QUEUED` (enfileirada). O **resultado final** da chamada não vem nesta resposta — é entregue de forma assíncrona no seu webhook configurado, através dos eventos `voice.answered` (atendida), `voice.no_answer` (não atendida) e `voice.failed` (falhou). Configure o webhook antes de disparar chamadas para capturar estes resultados.

Para volume, use `POST /v2/voice/bulk`, que enfileira até **1000** chamadas num único pedido, todas com o mesmo áudio. Cada telefone é validado de forma independente (sucesso parcial): os válidos entram em `accepted` (cada um com o seu `message_id`) e os inválidos em `rejected`.

**Limites e cobrança.** O *rate limit* é de **10000 pedidos por minuto**, contabilizado por conta e por canal — SMS, VOICE e upload de áudio contam separadamente. No caso do `bulk`, o lote é cobrado contra o *rate limit* pela quantidade de itens (`requests`). As chamadas consomem saldo da carteira (*wallet*): sem saldo suficiente, a API devolve `402`.

## Índice de endpoints

| Método | Caminho | O que faz |
|---|---|---|
| POST | `/v2/voice/audios` | Carrega um áudio MP3 e devolve um `audio_id` para reutilizar nas chamadas |
| POST | `/v2/voice` | Faz uma chamada de voz que reproduz um áudio pré-carregado |
| POST | `/v2/voice/bulk` | Faz até 1000 chamadas de voz num único pedido, todas com o mesmo áudio |

---

## `POST` `/v2/voice/audios` — Upload VOICE audio

**O que faz.** Carrega um ficheiro de áudio para poder ser usado em chamadas de voz. Forneça um link HTTPS público para o ficheiro; a Clint descarrega-o, armazena-o e devolve um `audio_id` a usar ao fazer chamadas (`POST /v2/voice`). **Apenas ficheiros MP3 são suportados** — o URL tem de apontar para um `.mp3`. Carregar o mesmo ficheiro outra vez devolve o mesmo `audio_id` — é armazenado apenas uma vez. Exige a *feature* **SMS & Voice** activada na conta pela equipa da Clint.

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.** Nenhum.

**Parâmetros de query.** Nenhum.

**Corpo da requisição.** `application/json` (obrigatório).

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `audio_url` | string | Sim | URL HTTPS publicamente acessível do ficheiro de áudio a carregar. Tem de apontar para um ficheiro MP3 (`.mp3`) — outros formatos não são suportados. |

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `201` | Áudio registado | Objeto com `audio_id` (string) e `reused` (boolean) |
| `400` | Erro de validação | — |
| `401` | Token de API inválido ou em falta | — |
| `403` | A *feature* SMS & Voice não está activada na conta | — |
| `429` | *Rate limit* excedido. Limite: 10000 pedidos por minuto, por conta e por canal (SMS, VOICE e upload de áudio contados separadamente). | — |

Campos do corpo de resposta (`201`):

| Nome | Tipo | Descrição |
|---|---|---|
| `audio_id` | string | Identificador do áudio carregado. Passe-o em `POST /v2/voice`. |
| `reused` | boolean | `true` se este exacto áudio já tinha sido carregado antes (reutilizado); `false` se foi armazenado agora. |

**Exemplo — requisição**
```bash
curl -X POST "https://api.clint.digital/v2/voice/audios" \
  -H "api-token: SUA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "audio_url": "https://cdn.example.com/clip.mp3"
  }'
```

**Exemplo — resposta (`201`)**
```json
{
  "audio_id": "api-audio-0a1b2c3d4e5f6a7b",
  "reused": false
}
```

**Notas (inteligência LL Mídia).**
- **Só MP3.** O URL tem de terminar num ficheiro `.mp3` acessível publicamente por HTTPS. Ficheiros atrás de autenticação, com redireccionamentos estranhos ou noutro formato falham na validação (`400`).
- **Idempotência por conteúdo.** Reenviar o mesmo ficheiro devolve o mesmo `audio_id` com `"reused": true`. Isto significa que pode chamar este endpoint sem receio de duplicar armazenamento — mas o ideal continua a ser guardar o `audio_id` do lado da sua aplicação e reutilizá-lo directamente.
- **Canal próprio no *rate limit*.** O upload de áudio conta como um canal separado de SMS e VOICE para efeitos do limite de 10000 pedidos/minuto.
- **Pré-requisito das chamadas.** Este endpoint é o passo 1 obrigatório antes de `POST /v2/voice` ou `POST /v2/voice/bulk` — ambos exigem um `audio_id` já existente.

---

## `POST` `/v2/voice` — Send VOICE call

**O que faz.** Faz uma chamada de voz que reproduz um áudio pré-carregado. Carregue primeiro o áudio via `POST /v2/voice/audios` e passe aqui o `audio_id` devolvido. Responde imediatamente com estado `QUEUED`; o resultado da chamada é enviado para o seu webhook configurado (eventos `voice.answered` / `voice.no_answer` / `voice.failed`). Exige a *feature* **SMS & Voice** activada na conta pela equipa da Clint.

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.** Nenhum.

**Parâmetros de query.** Nenhum.

**Corpo da requisição.** `application/json` (obrigatório).

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `phone` | string | Sim | Telefone de destino em formato internacional (ex.: `+5511999998888`). |
| `audio_id` | string | Sim | `audio_id` devolvido por `POST /v2/voice/audios`. |

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `201` | Enfileirada | Objeto com `success` (boolean), `message_id` (string) e `status` (string) |
| `400` | Erro de validação | — |
| `401` | Token de API inválido ou em falta | — |
| `402` | Saldo insuficiente na carteira para fazer a chamada. Adicione crédito e tente novamente. | Objeto com `status`, `message`, `wallet_balance`, `minimum_required` |
| `403` | A *feature* SMS & Voice não está activada na conta | Objeto com `status` e `message` |
| `429` | *Rate limit* excedido. Limite: 10000 pedidos por minuto, por conta e por canal (SMS, VOICE e upload de áudio contados separadamente). | — |

Campos do corpo de resposta (`201`):

| Nome | Tipo | Descrição |
|---|---|---|
| `success` | boolean | Indica se a chamada foi aceite e enfileirada. |
| `message_id` | string | Identificador da chamada enfileirada; use-o para correlacionar com os eventos de webhook. |
| `status` | string | Estado da chamada no momento da resposta (ex.: `QUEUED`). |

Campos do corpo de resposta (`402`):

| Nome | Tipo | Descrição |
|---|---|---|
| `status` | integer | Código do erro (ex.: `402`). |
| `message` | string | Mensagem descritiva (ex.: `Insufficient wallet balance to send voice calls`). |
| `wallet_balance` | number | Saldo actual da carteira (ex.: `0.5`). |
| `minimum_required` | number | Saldo mínimo necessário para fazer a chamada (ex.: `1`). |

Campos do corpo de resposta (`403`):

| Nome | Tipo | Descrição |
|---|---|---|
| `status` | integer | Código do erro (ex.: `403`). |
| `message` | string | Mensagem descritiva (ex.: `This feature is not available for your account.`). |

**Exemplo — requisição**
```bash
curl -X POST "https://api.clint.digital/v2/voice" \
  -H "api-token: SUA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "phone": "+5511999998888",
    "audio_id": "api-audio-0a1b2c3d4e5f6a7b"
  }'
```

**Exemplo — resposta (`201`)**
```json
{
  "success": true,
  "message_id": "def456",
  "status": "QUEUED"
}
```

**Exemplo — resposta (`402`)**
```json
{
  "status": 402,
  "message": "Insufficient wallet balance to send voice calls",
  "wallet_balance": 0.5,
  "minimum_required": 1
}
```

**Exemplo — resposta (`403`)**
```json
{
  "status": 403,
  "message": "This feature is not available for your account."
}
```

**Notas (inteligência LL Mídia).**
- **`QUEUED` não é resultado final.** A resposta `201` confirma apenas que a chamada entrou na fila. Se foi atendida, não atendida ou falhou só é sabido pelos eventos de webhook (`voice.answered` / `voice.no_answer` / `voice.failed`). Configure o webhook antes de disparar em produção.
- **Carteira (*wallet*).** As chamadas consomem saldo. Antes de campanhas grandes, garanta crédito suficiente para evitar `402` a meio do disparo. O corpo do `402` traz `wallet_balance` e `minimum_required` — útil para alertas automáticos.
- **`audio_id` tem de existir.** Um `audio_id` desconhecido ou em falta cai em `400`. Sempre encadeie este passo depois de `POST /v2/voice/audios`.
- **Gating por *feature flag*.** Se receber `403` com `"This feature is not available for your account."`, a *feature* SMS & Voice não está activada — é preciso pedir activação à equipa da Clint.

---

## `POST` `/v2/voice/bulk` — Send VOICE calls in bulk

**O que faz.** Coloca até **1000** chamadas de voz num único pedido, todas a reproduzir o mesmo áudio pré-carregado. Carregue o áudio uma vez via `POST /v2/voice/audios` e passe o `audio_id`. Cada telefone é validado de forma independente — os válidos são enfileirados e devolvidos em `accepted` (cada um com o seu `message_id`), os inválidos em `rejected` (sucesso parcial). O resultado de cada chamada é enviado por item para o seu webhook configurado (eventos `voice.answered` / `voice.no_answer` / `voice.failed`). O lote é cobrado contra o *rate limit* por minuto pela sua quantidade de itens. Exige a *feature* **SMS & Voice** activada na conta pela equipa da Clint.

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.** Nenhum.

**Parâmetros de query.** Nenhum.

**Corpo da requisição.** `application/json` (obrigatório).

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `audio_id` | string | Sim | `audio_id` devolvido por `POST /v2/voice/audios`. |
| `requests` | array | Sim | Lista de destinos (máx. 1000 itens). Cada item é um objeto com o campo `phone`. |
| `requests[].phone` | string | Sim | Telefone de destino de cada chamada, em formato internacional (ex.: `+5511999998888`). |

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `202` | Aceite. Chamadas válidas enfileiradas; ver `accepted`/`rejected` para o resultado por item. | Objeto com `success`, `summary`, `accepted`, `rejected` |
| `400` | Erro de validação (ex.: `audio_id` em falta ou desconhecido) | — |
| `401` | Token de API inválido ou em falta | — |
| `402` | Saldo insuficiente na carteira para enviar | — |
| `403` | A *feature* SMS & Voice não está activada na conta | — |
| `429` | *Rate limit* excedido. Cobrado pela quantidade de itens; limite de 10000 pedidos por minuto, por conta e por canal. | — |

Campos do corpo de resposta (`202`):

| Nome | Tipo | Descrição |
|---|---|---|
| `success` | boolean | Indica se o pedido em lote foi aceite. |
| `summary` | object | Resumo agregado do lote. |
| `summary.total` | integer | Total de itens submetidos. |
| `summary.accepted` | integer | Quantidade de itens aceites e enfileirados. |
| `summary.rejected` | integer | Quantidade de itens rejeitados. |
| `accepted` | array | Lista dos itens aceites. |
| `accepted[].phone` | string | Telefone do item aceite. |
| `accepted[].message_id` | string | Identificador da chamada enfileirada para esse telefone. |
| `accepted[].status` | string | Estado da chamada no momento da resposta (ex.: `QUEUED`). |
| `rejected` | array | Lista dos itens rejeitados. |
| `rejected[].phone` | string | Telefone do item rejeitado. |
| `rejected[].reason` | string | Motivo da rejeição do item. |

**Exemplo — requisição**
```bash
curl -X POST "https://api.clint.digital/v2/voice/bulk" \
  -H "api-token: SUA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "audio_id": "api-audio-0a1b2c3d4e5f6a7b",
    "requests": [
      { "phone": "+5511999998888" },
      { "phone": "+5511988887777" },
      { "phone": "telefone-invalido" }
    ]
  }'
```

**Exemplo — resposta (`202`)**
```json
{
  "success": true,
  "summary": {
    "total": 3,
    "accepted": 2,
    "rejected": 1
  },
  "accepted": [
    {
      "phone": "+5511999998888",
      "message_id": "def456",
      "status": "QUEUED"
    },
    {
      "phone": "+5511988887777",
      "message_id": "ghi789",
      "status": "QUEUED"
    }
  ],
  "rejected": [
    {
      "phone": "telefone-invalido",
      "reason": "Invalid phone number"
    }
  ]
}
```

**Notas (inteligência LL Mídia).**
- **Sucesso parcial (`202`).** Um `202` não garante que todos os telefones foram aceites. Verifique sempre `summary.rejected` e o array `rejected` para saber o que ficou de fora, e faça reenvio apenas dos telefones corrigidos.
- **Correlacione por `message_id`.** Cada item aceite traz o seu `message_id`; guarde o par `phone` → `message_id` para casar com os eventos de webhook por chamada.
- **Máximo de 1000 itens por pedido.** Listas maiores têm de ser partidas em vários pedidos. Lembre-se de que o lote é cobrado contra o *rate limit* pela quantidade de itens.
- **Um só áudio por lote.** Todas as chamadas do lote reproduzem o mesmo `audio_id`. Para mensagens diferentes, faça lotes separados (um por `audio_id`).
- **Carteira.** Tal como na chamada individual, um saldo insuficiente devolve `402` — dimensione o crédito para o tamanho da campanha.

---

## Objetos relacionados

Esta categoria não referencia schemas nomeados na spec — os corpos de requisição e resposta são objetos *inline*, documentados acima em cada endpoint. Para os canais e conceitos relacionados, ver:

- [SMS](./sms.md) — canal irmão que partilha a mesma *feature* SMS & Voice e o mesmo modelo de *rate limit* por canal.
- [Webhooks](./webhooks.md) — recebe os eventos assíncronos de resultado das chamadas (`voice.answered` / `voice.no_answer` / `voice.failed`).
- [Convenções](../02-convencoes.md) — erros comuns (`401`, `402`, `403`, `429`) e autenticação por `api-token`.
