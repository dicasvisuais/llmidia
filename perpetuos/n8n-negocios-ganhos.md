# Automação N8N — Negócios ganhos da Clint → Google Sheets (diário 04:00)

Traz, uma vez por dia (**04:00**), todos os **negócios ganhos** (`status = WON`) de uma
origem da Clint para uma folha de cálculo do Google, **sem duplicar linhas** (upsert pelo
`id` do negócio).

- **Origem monitorizada:** `07fc7c4b-82d2-427d-b09e-04a7f90f16f1` (PIPELINE_COMERCIAL-V3)
- **Planilha:** <https://docs.google.com/spreadsheets/d/1LPZ4rVl6hEOCPsu3gxMPqKJr1SN5H-OuVQYIZaDeEgM/edit>
- **Workflow importável:** [`n8n-negocios-ganhos.json`](./n8n-negocios-ganhos.json)
- **Endpoint usado:** [`GET /v1/deals`](../_docs/clint-api/endpoints/deals.md) · Auth: header `api-token`

---

## Fluxo

```
┌──────────────────────┐   ┌────────────────────────────┐   ┌───────────────────────┐   ┌────────────────────────────┐
│ Schedule 04:00       │──▶│ HTTP: Clint GET /v1/deals  │──▶│ Code: mapear → colunas │──▶│ Google Sheets: Append/Update│
│ (cron 0 4 * * *)     │   │ ?origin_id=…&status=WON    │   │ (valor + utm + nativos)│   │ (upsert por id_negocio)     │
└──────────────────────┘   └────────────────────────────┘   └───────────────────────┘   └────────────────────────────┘
```

Como escolheste **Upsert por ID**, o workflow busca **todos** os negócios ganhos da origem
(`limit=1000`, chega e sobra para os ~30 atuais) e, para cada um, **cria a linha se for
nova ou atualiza a existente** com base na coluna `id_negocio`. Assim a planilha está
sempre completa e atual, sem linhas repetidas.

> **Nota sobre a resposta real da API.** A resposta de `GET /v1/deals` traz **mais campos
> do que a spec OpenAPI oficial documenta**. Em particular, o **valor do negócio é nativo**
> (`deal.value`, no topo do objeto — não está em `fields`), acompanhado de `currency`,
> `currency_symbol` e `stage` (o **nome** legível da etapa). Os **UTM** e outros campos
> personalizados é que vivem em `deal.fields`.

---

## Colunas da planilha

O nó Google Sheets usa **auto-mapeamento**: casa cada campo que sai do Code com a coluna
de **mesmo nome** na folha. Por isso a **linha de cabeçalho (linha 1) tem de existir com
estes nomes exatos**. Cria uma aba chamada **`Negócios Ganhos`** e cola isto em `A1`
(são valores separados por TAB):

```
id_negocio	estado	contacto_nome	contacto_email	contacto_telefone	responsavel	valor_fechado	moeda	criado_em	ganho_em	dias_para_ganhar	etapa	etapa_id	origem_id	utm_source	utm_medium	utm_campaign	utm_content	utm_term	utm_id
```

| Coluna | Origem na API | Notas |
|---|---|---|
| `id_negocio` | `id` | **Chave do upsert.** Não apagar/editar. |
| `estado` | `status` | Traduzido: `WON`→`Ganho`. |
| `contacto_nome` | `contact.name` | É o "título" que aparece no card da Clint. |
| `contacto_email` | `contact.email` | |
| `contacto_telefone` | `contact.phone` | |
| `responsavel` | `user.full_name` | Dono do negócio. |
| `valor_fechado` | **`value`** (nativo, topo) | Numérico (ex.: `499`). **Não** está em `fields`. |
| `moeda` | `currency` | Ex.: `EUR`. |
| `criado_em` | `created_at` | Formatado `dd/mm/aa, hh:mm` (fuso `Europe/Lisbon`). |
| `ganho_em` | `won_at` | Data em que passou a Ganho. |
| `dias_para_ganhar` | calculado | Número de dias entre `criado_em` e `ganho_em` (inteiro). Vazio se sem data de ganho. |
| `etapa` | **`stage`** (nativo) | Nome legível da etapa (ex.: `Reunião agendada`). |
| `etapa_id` | `stage_id` | UUID da etapa (para referência técnica). |
| `origem_id` | `origin_id` | Constante (é a origem filtrada). |
| `utm_source`…`utm_id` | campos `fields` com "utm" | Preenchidos automaticamente por qualquer campo cujo nome contenha `utm`. |

> O Code apanha **qualquer** campo UTM (inclui `utm_id` e `utm_term`). Se tiveres um UTM
> fora do padrão, ele cria a chave automaticamente — basta acrescentares a coluna com esse
> nome ao cabeçalho.

---

## Passo-a-passo

### Pré-requisitos
- Acesso ao teu N8N.
- **Chave de API da Clint** (painel da Clint → definições/integrações).
- Conta Google com acesso de edição à planilha.

### 1. Credencial da Clint (header `api-token`) — a chave fica só aqui
No N8N: **Credentials → New → "Header Auth"**
- **Name:** `Clint api-token`
- **Header Name:** `api-token`
- **Header Value:** a tua chave da Clint

> A chave vive **apenas nesta credencial** do N8N — nunca no JSON nem na planilha.

### 2. Credencial do Google Sheets
**Credentials → New → "Google Sheets OAuth2 API"** e autoriza a conta Google que tem
acesso à planilha.

### 3. Importar o workflow
**Workflows → Import from File →** escolhe [`n8n-negocios-ganhos.json`](./n8n-negocios-ganhos.json).

### 4. Ligar credenciais e a aba
- Nó **"Clint: GET /v1/deals (WON)"** → escolhe a credencial `Clint api-token`.
- Nó **"Google Sheets: Append or Update"** → escolhe a credencial Google, confirma o
  *Document* (já vem com o ID da planilha) e seleciona a **aba `Negócios Ganhos`**.

### 5. Valor do negócio — já vem resolvido ✅
O valor é o campo **nativo** `deal.value`, por isso o Code preenche `valor_fechado`
automaticamente — **não precisas de configurar nada**. A constante `VALUE_FIELD_LABEL` no
topo do código pode ficar `''`; só a usarias se, num caso atípico, o valor estivesse noutro
campo. Para inspecionar a resposta real:

```bash
curl -s "https://api.clint.digital/v1/deals?origin_id=07fc7c4b-82d2-427d-b09e-04a7f90f16f1&status=WON&limit=1" \
  -H "api-token: SUA_API_KEY" | python3 -m json.tool
```

### 6. Testar e ativar
- **"Execute Workflow"** e confirma as linhas na planilha (agora com `valor_fechado`, `moeda` e `etapa`).
- Corre outra vez: as linhas **não** duplicam (upsert por `id_negocio`).
- Ativa o toggle **Active** — corre sozinho todos os dias às 04:00.

---

## Notas e ajustes

- **Fuso horário.** O trigger e as datas usam `Europe/Lisbon`. Para horário do Brasil, muda
  o *timezone* do workflow (Settings) e a constante `TIMEZONE` no Code node para
  `America/Sao_Paulo`.
- **Valor numérico.** `valor_fechado` entra como número (ex.: `499`) para permitir somas na
  planilha (o nó está com *convert to string* desligado).
- **Volume > 1000 ganhos.** Hoje um único pedido (`limit=1000`) chega. Se ultrapassares,
  ativa a **paginação** no nó HTTP: *Options → Pagination →* "Update a parameter in each
  request", parâmetro `page` = `{{ $pageCount + 1 }}`, *Stop condition* =
  `{{ $response.body.hasNext === false }}`. O Code já junta várias páginas.
- **Só leitura.** O workflow apenas lê da Clint e escreve na planilha; não altera nada na Clint.

---

_Documento interno LL Mídia. Fonte da API: OpenAPI oficial da Clint (`@clint-api/v1.0`),
complementada com campos reais observados na resposta (`value`, `currency`, `stage`)._
