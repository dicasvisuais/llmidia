# SMS — Clint API

Envio de mensagens SMS transacionais e em massa, com estado de entrega devolvido por webhook.

> Fonte: OpenAPI oficial da Clint (`@clint-api/v1.0`) · Base `https://api.clint.digital` · Auth: header `api-token` · Documento interno LL Mídia.

## Visão geral

A categoria **SMS** expõe o envio de mensagens de texto pela plataforma Clint. São endpoints de **v2** (mensageria/omnichannel): um para envio unitário (`POST /v2/sms`) e outro para envio em massa até 1000 mensagens numa só chamada (`POST /v2/sms/bulk`). Ambos exigem que a funcionalidade **SMS & Voice** esteja ativada na conta pela equipa da Clint — caso contrário a chamada devolve `403` com a mensagem `"This feature is not available for your account."`.

O modelo é assíncrono: a API responde de imediato com um estado `QUEUED` e um `message_id`. O **estado final de entrega** não vem na resposta — é enviado depois para o webhook configurado na conta, através dos eventos `sms.delivered`, `sms.undelivered` e `sms.error`. Guarde sempre o `message_id` devolvido para conseguir cruzar essas atualizações de estado mais tarde (ver [Webhooks](./webhooks.md)).

Há três aspetos operacionais a reter. Primeiro, o **saldo da carteira (wallet)**: sem crédito suficiente a chamada devolve `402`. Segundo, o **rate limit**: 10000 pedidos por minuto, por conta e por canal (SMS, VOICE e upload de áudio são contados em separado); no envio em massa a lotação é debitada pelo número de itens do lote. Terceiro, os números de telefone devem ir em **formato internacional** (ex.: `+5511999998888`).

> **Guia oficial (`tag_description`) — reproduzido**
>
> Envie um SMS num só passo. Autentique-se com a sua API key no header `api-token` e use números de telefone em formato internacional (ex.: `+5511999998888`).
>
> **Enviar um SMS** — `POST /v2/sms`
>
> ```json
> { "phone": "+5511999998888", "message": "Hello!" }
> ```
>
> Devolve `{ "success": true, "message_id": "abc123", "status": "QUEUED" }`. Guarde o `message_id` para casar depois com as atualizações de estado (ver Webhooks).
>
> Se a sua conta não tiver esta funcionalidade, a chamada devolve `403 — "This feature is not available for your account."`

Em suma: use `POST /v2/sms` para envios pontuais (códigos OTP, alertas) e `POST /v2/sms/bulk` para campanhas ou lotes, tirando partido da **validação independente por item** — os válidos entram em fila (`accepted`) e os inválidos são devolvidos (`rejected`) sem falhar o lote inteiro (sucesso parcial).

## Índice de endpoints

| Método | Caminho | O que faz |
|---|---|---|
| POST | `/v2/sms` | Envia um SMS; devolve `QUEUED` e `message_id`. |
| POST | `/v2/sms/bulk` | Envia até 1000 SMS num pedido, com sucesso parcial (`accepted`/`rejected`). |

---

## `POST` `/v2/sms` — Enviar SMS

**O que faz.** Envia um SMS. Responde de imediato com estado `QUEUED`; o estado final de entrega é enviado para o webhook configurado (eventos `sms.delivered` / `sms.undelivered` / `sms.error`). Requer a funcionalidade **SMS & Voice** ativada na conta pela equipa da Clint.

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.** Nenhum.

**Parâmetros de query.** Nenhum.

**Header específico.**

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `api-token` | string | Sim | API Token da conta. |

**Corpo da requisição.** `application/json` (obrigatório) — objeto:

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `phone` | string | Sim | Número de destino em formato internacional (ex.: `+5511999998888`). |
| `message` | string | Sim | Texto da mensagem (ex.: `Your code is 1234`). |
| `sms_type` | string (enum) | Não | Tipo de SMS. Valores: `SMS`, `SMS_FLASH`. Default: `SMS`. |

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `201` | Queued (mensagem colocada em fila). | objeto: `success` (boolean), `message_id` (string), `status` (string, ex.: `QUEUED`). |
| `400` | Erro de validação. | — |
| `401` | API token inválido ou em falta. | — |
| `402` | Saldo da carteira insuficiente para enviar. | objeto: `status` (integer, ex.: `402`), `message` (string, ex.: `Insufficient wallet balance to send SMS`), `wallet_balance` (number, ex.: `0.5`), `minimum_required` (number, ex.: `1`). |
| `403` | Funcionalidade SMS & Voice não ativada na conta. | objeto: `status` (integer, ex.: `403`), `message` (string, ex.: `This feature is not available for your account.`). |
| `429` | Rate limit excedido. Limite: 10000 pedidos por minuto, por conta e por canal (SMS, VOICE e upload de áudio são contados em separado). | — |

**Exemplo — requisição**
```bash
curl -X POST "https://api.clint.digital/v2/sms" \
  -H "api-token: SUA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "phone": "+5511999998888",
    "message": "Your code is 1234",
    "sms_type": "SMS"
  }'
```

**Exemplo — resposta (`201`)**
```json
{
  "success": true,
  "message_id": "abc123",
  "status": "QUEUED"
}
```

**Exemplo — resposta (`402`)**
```json
{
  "status": 402,
  "message": "Insufficient wallet balance to send SMS",
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
- O `201`/`QUEUED` confirma **aceitação**, não entrega. A confirmação de entrega chega apenas por webhook (`sms.delivered` / `sms.undelivered` / `sms.error`) — guarde o `message_id` para casar o callback com o envio.
- `SMS_FLASH` envia um flash SMS (aparece diretamente no ecrã do dispositivo, sem passar pela caixa de mensagens); use `SMS` (default) para o comportamento normal.
- `402` é recuperável: carregue crédito na carteira e repita. O corpo indica `wallet_balance` atual e `minimum_required` para orientar o topup.
- `403` é gating por feature flag da conta (SMS & Voice) — não se resolve via API; é preciso a equipa da Clint ativar.
- O canal SMS partilha o teto de rate limit por canal (10000/min); SMS, VOICE e upload de áudio são contados separadamente.

---

## `POST` `/v2/sms/bulk` — Enviar SMS em massa

**O que faz.** Envia até 1000 SMS num único pedido. Cada item é validado de forma independente — os itens válidos entram em fila e são devolvidos em `accepted` (cada um com o seu `message_id`), os inválidos são devolvidos em `rejected` (**sucesso parcial**). O estado de entrega é enviado por item para o webhook configurado (eventos `sms.delivered` / `sms.undelivered` / `sms.error`). O lote é debitado ao rate limit por minuto pelo número de itens. Requer a funcionalidade **SMS & Voice** ativada na conta pela equipa da Clint.

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.** Nenhum.

**Parâmetros de query.** Nenhum.

**Header específico.**

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `api-token` | string | Sim | API Token da conta. |

**Corpo da requisição.** `application/json` (obrigatório) — objeto:

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `requests` | array (máx. 1000 itens) | Sim | Lista de mensagens a enviar. Cada item é um objeto (ver abaixo). |
| `sms_type` | string (enum) | Não | Tipo de SMS aplicado ao lote. Valores: `SMS`, `SMS_FLASH`. Default: `SMS`. |

Cada item de `requests` é um objeto:

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `phone` | string | Sim | Número de destino em formato internacional (ex.: `+5511999998888`). |
| `message` | string | Sim | Texto da mensagem (ex.: `Your code is 1234`). |

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `202` | Accepted. Os itens válidos entram em fila; ver `accepted`/`rejected` para o resultado por item. | objeto (ver campos abaixo). |
| `401` | API token inválido ou em falta. | — |
| `402` | Saldo da carteira insuficiente para enviar. | — |
| `403` | Funcionalidade SMS & Voice não ativada na conta. | — |
| `429` | Rate limit excedido. Debitado pelo número de itens; limite de 10000 pedidos por minuto, por conta e por canal. | — |

Corpo da resposta `202`:

| Nome | Tipo | Descrição |
|---|---|---|
| `success` | boolean | Indica que o pedido foi aceite para processamento. |
| `summary` | object | Contagens do lote. |
| `summary.total` | integer | Total de itens submetidos. |
| `summary.accepted` | integer | Nº de itens aceites (em fila). |
| `summary.rejected` | integer | Nº de itens rejeitados. |
| `accepted` | array | Itens aceites; cada um com `phone`, `message_id` e `status`. |
| `accepted[].phone` | string | Número do item aceite. |
| `accepted[].message_id` | string | Identificador da mensagem em fila (para casar com webhooks). |
| `accepted[].status` | string | Estado do item (ex.: `QUEUED`). |
| `rejected` | array | Itens rejeitados; cada um com `phone` e `reason`. |
| `rejected[].phone` | string | Número do item rejeitado. |
| `rejected[].reason` | string | Motivo da rejeição do item. |

**Exemplo — requisição**
```bash
curl -X POST "https://api.clint.digital/v2/sms/bulk" \
  -H "api-token: SUA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "sms_type": "SMS",
    "requests": [
      { "phone": "+5511999998888", "message": "Your code is 1234" },
      { "phone": "+5511988887777", "message": "Your code is 5678" }
    ]
  }'
```

**Exemplo — resposta (`202`)**
```json
{
  "success": true,
  "summary": {
    "total": 2,
    "accepted": 1,
    "rejected": 1
  },
  "accepted": [
    {
      "phone": "+5511999998888",
      "message_id": "abc123",
      "status": "QUEUED"
    }
  ],
  "rejected": [
    {
      "phone": "+5511988887777",
      "reason": "Invalid phone number"
    }
  ]
}
```

**Notas (inteligência LL Mídia).**
- **Sucesso parcial:** um `202` não garante que todos os itens foram aceites. Percorra sempre `rejected` e reprocesse esses números; `summary` dá a contagem rápida (`total`/`accepted`/`rejected`).
- O array `requests` tem **limite rígido de 1000 itens** (`maxItems`). Para lotes maiores, divida em vários pedidos e respeite o rate limit.
- O **rate limit é debitado pelo número de itens** do lote, não por chamada — um pedido com 1000 itens consome 1000 do teto de 10000/min por canal.
- `sms_type` aplica-se a todo o lote (não é definível por item, ao contrário de `phone`/`message`).
- Tal como no envio unitário, a entrega final vem por webhook por item (`sms.delivered` / `sms.undelivered` / `sms.error`); guarde cada `message_id` de `accepted`.
- `402` e `403` seguem a mesma lógica do endpoint unitário (carteira sem saldo / feature SMS & Voice não ativada), mas neste endpoint são devolvidos sem corpo detalhado.

---

## Objetos relacionados

Esta categoria não usa schemas nomeados/reutilizáveis (`components/schemas`): os corpos de requisição e de resposta são objetos *inline*, documentados nas tabelas acima. Referências úteis:

- [Webhooks](./webhooks.md) — recebem o estado de entrega por mensagem (`sms.delivered` / `sms.undelivered` / `sms.error`); cruze pelo `message_id`.
- [Convenções](../02-convencoes.md) — erros comuns (`401`, `402`, `403`, `429`) e comportamento de rate limit.
- [Autenticação](../01-autenticacao.md) — header `api-token`.
