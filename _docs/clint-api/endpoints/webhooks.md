# Webhooks — Clint API

Configuração de webhooks para receber atualizações de estado de entrega de SMS e de estado de chamadas de voz (VOICE), enviadas por POST para a sua URL.

> Fonte: OpenAPI oficial da Clint (`@clint-api/v1.0`) · Base `https://api.clint.digital` · Auth: header `api-token` · Documento interno LL Mídia.

## Visão geral

Os **Webhooks** permitem-lhe receber atualizações de estado de entrega (SMS) e de estado de chamada (VOICE) sem ter de fazer *polling*. Configura uma vez a URL para onde devem ser enviadas as atualizações e, a partir daí, a Clint faz `POST` de cada atualização para essa URL. Estes endpoints vivem em `/v2` e dependem da *feature* **SMS & Voice**: se ela não estiver ativa na sua conta, obtém `403` com a mensagem `"This feature is not available for your account."`.

O guia oficial (`tag_description`) descreve o fluxo completo:

**Configurar** — `PUT /v2/webhooks`

```json
{ "url": "https://yoursite.com/clint-status", "events": ["sms.delivered", "sms.error", "voice.answered"] }
```

A lista `events` é o **conjunto completo** de subscrições — inclua os eventos que quer e omita os restantes. A resposta devolve um `secret` (mostrado **apenas uma vez** — guarde-o de imediato).

**O que vai receber** (corpo de cada callback POST enviado pela Clint para a sua URL):

```json
{ "channel": "SMS", "message_id": "abc123", "event": "sms.delivered", "phone": "+5511999998888", "status": "DELIVERED", "timestamp": "..." }
```

**Confirmar a autenticidade.** Para confirmar que o callback é mesmo da Clint, verifique o header `X-Clint-Signature` usando o seu `secret`. A assinatura é calculada assim: `X-Clint-Signature = "sha256=" + HMAC-SHA256` sobre a string `"<X-Clint-Timestamp>.<rawBody>"` usando o `secret` devolvido na configuração. Ou seja, concatena o valor do header `X-Clint-Timestamp` com o corpo bruto (`rawBody`) separados por um ponto, aplica HMAC-SHA256 com o `secret` e compara o resultado (prefixado por `sha256=`) com o header `X-Clint-Signature` recebido.

**Eventos disponíveis:**

- **SMS** — `sms.delivered`, `sms.undelivered`, `sms.error`
- **Voice** — `voice.answered`, `voice.no_answer`, `voice.failed`

## Índice de endpoints

| Método | Caminho | O que faz |
|---|---|---|
| GET | `/v2/webhooks` | Devolve a configuração de webhook atual (URL, eventos subscritos e se há *secret*). |
| PUT | `/v2/webhooks` | Define/atualiza a URL e as subscrições de eventos; corpo vazio remove o webhook. |

---

## `GET` `/v2/webhooks` — Get webhook config

**O que faz.** Devolve a URL de webhook atualmente configurada, os eventos subscritos e se existe um *signing secret* definido.

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.** Nenhum.

**Parâmetros de query.** Nenhum.

**Corpo da requisição.** Nenhum.

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `200` | Webhook configuration | `object` (`url`, `events`, `has_secret`) |
| `401` | Invalid or missing API token | — |
| `403` | The SMS & Voice feature is not enabled for your account | `object` (`status`, `message`) |

**Campos da resposta `200`.**

| Nome | Tipo | Descrição |
|---|---|---|
| `url` | `string` (nullable) | URL de destino atualmente configurada. `null` se não houver webhook configurado. |
| `events` | `array<string>` | Os eventos que tem atualmente subscritos. Cada item é um dos enums: `sms.delivered`, `sms.undelivered`, `sms.error`, `voice.answered`, `voice.no_answer`, `voice.failed`. |
| `has_secret` | `boolean` | Indica se já existe um *signing secret* definido para esta configuração. |

**Campos da resposta `403`.**

| Nome | Tipo | Descrição |
|---|---|---|
| `status` | `integer` | Código de estado (ex.: `403`). |
| `message` | `string` | Mensagem de erro (ex.: `"This feature is not available for your account."`). |

**Exemplo — requisição**
```bash
curl -X GET "https://api.clint.digital/v2/webhooks" \
  -H "api-token: SUA_API_KEY"
```

**Exemplo — resposta (`200`)**
```json
{
  "url": "https://yourserver.example.com/webhook",
  "events": [
    "sms.delivered",
    "voice.answered"
  ],
  "has_secret": true
}
```

**Exemplo — resposta (`403`)**
```json
{
  "status": 403,
  "message": "This feature is not available for your account."
}
```

**Notas (inteligência LL Mídia).** Endpoint de leitura, ideal para verificar o estado atual antes de reconfigurar. Repare que `has_secret` só diz **se** existe um *secret* — o valor do *secret* nunca é devolvido aqui (só surge uma vez, na resposta ao `PUT` de primeira configuração). Se `url` vier `null` e `events` vazio, o webhook não está configurado. O `403` costuma indicar que a *feature* **SMS & Voice** não está ativa na conta; confirme o *gating* de escopo/feature flag antes de assumir erro de integração.

---

## `PUT` `/v2/webhooks` — Set webhook config

**O que faz.** Configura a URL de webhook e as subscrições de eventos. Forneça `url` e `events` para configurar; envie um corpo vazio (omita `url`) para **remover** o webhook por completo. Quando `url` é fornecido, `events` é obrigatório. Um novo *HMAC secret* é gerado e devolvido **uma única vez** na primeira configuração — guarde-o de imediato. Os callbacks de webhook são assinados: header `X-Clint-Signature = "sha256=" + HMAC-SHA256` sobre `"<X-Clint-Timestamp>.<rawBody>"` usando o *secret* devolvido.

**Autenticação.** Header `api-token` (obrigatório).

**Parâmetros de caminho.** Nenhum.

**Parâmetros de query.** Nenhum.

**Corpo da requisição.** `application/json` (obrigatório). Schema `object`:

| Nome | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `url` | `string` | Não | URL de destino. Omita (envie um corpo vazio) para remover o webhook. Quando presente, torna `events` obrigatório. |
| `events` | `array<string>` | Condicional | Lista dos eventos que quer receber. A lista é o **conjunto completo** de subscrições: os eventos incluídos são entregues, os omitidos não (reenvie a lista completa para alterar). Obrigatório quando `url` é fornecido. Cada item é um dos enums: `sms.delivered`, `sms.undelivered`, `sms.error`, `voice.answered`, `voice.no_answer`, `voice.failed`. Nota: os envios de SMS são também faturados; isto apenas controla que *callbacks* de estado chegam ao seu webhook. |

**Respostas.**

| Código | Descrição | Corpo/Schema |
|---|---|---|
| `200` | Config saved; secret returned once | `object` (`success`, `secret`) |
| `400` | Validation error | — |
| `401` | Invalid or missing API token | — |
| `403` | The SMS & Voice feature is not enabled for your account | `object` (`status`, `message`) |

**Campos da resposta `200`.**

| Nome | Tipo | Descrição |
|---|---|---|
| `success` | `boolean` | Indica se a configuração foi guardada com sucesso. |
| `secret` | `string` | *HMAC signing secret* — guarde imediatamente, não será mostrado novamente. |

**Campos da resposta `403`.**

| Nome | Tipo | Descrição |
|---|---|---|
| `status` | `integer` | Código de estado (ex.: `403`). |
| `message` | `string` | Mensagem de erro (ex.: `"This feature is not available for your account."`). |

**Exemplo — requisição (configurar)**
```bash
curl -X PUT "https://api.clint.digital/v2/webhooks" \
  -H "api-token: SUA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://yourserver.example.com/webhook",
    "events": [
      "sms.delivered",
      "sms.error",
      "voice.answered"
    ]
  }'
```

**Exemplo — requisição (remover)**
```bash
curl -X PUT "https://api.clint.digital/v2/webhooks" \
  -H "api-token: SUA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{}'
```

**Exemplo — resposta (`200`)**
```json
{
  "success": true,
  "secret": "whsec_a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6"
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
- **A lista é substitutiva, não incremental.** Cada `PUT` reescreve o conjunto completo de subscrições. Para acrescentar ou remover um evento, reenvie sempre a lista inteira com o estado desejado.
- **O `secret` só aparece uma vez.** É devolvido na resposta ao `PUT` de primeira configuração. Guarde-o num local seguro no momento — não há forma de o obter de novo (o `GET` só devolve `has_secret`). Se o perder, terá de reconfigurar para gerar um novo.
- **Validação da assinatura.** Ao receber cada callback, calcule `HMAC-SHA256` sobre `"<X-Clint-Timestamp>.<rawBody>"` (o `rawBody` é o corpo **bruto**, não re-serializado) com o `secret`, prefixe com `sha256=` e compare com o header `X-Clint-Signature`. Use comparação de tempo constante para evitar *timing attacks*.
- **Remoção.** Enviar `{}` (corpo vazio, sem `url`) remove o webhook por completo.
- **Faturação.** A subscrição de eventos apenas controla que *callbacks* de estado recebe; não altera a faturação dos envios de SMS, que são cobrados independentemente.
- **Gating.** Tal como o `GET`, depende da *feature* **SMS & Voice**; `403` indica *feature flag*/escopo em falta.

## Objetos relacionados

Esta categoria não referencia schemas nomeados na spec — os corpos de pedido e de resposta são objetos *inline*. Para o modelo de dados geral, consulte [`../03-modelo-de-dados.md`](../03-modelo-de-dados.md). Categorias irmãs de mensageria: [SMS](./sms.md) e [Voice](./voice.md).
