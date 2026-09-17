# Automação N8N — Conversão cruzada do WGT · v2 (taxa + uma linha por pessoa)

Responde à **Pergunta 1 da Direção** — *"os leads do WGT convertem noutros funis?"* — e
agora dá a **taxa de conversão cruzada** (com o denominador certo) e um CSV **por pessoa**.

> **100% leitura.** Só lê da Clint e gera um ficheiro. Não altera nada.

- **Origem WGT:** `7403f4aa-7a29-4863-b44e-b4f5a6ae6094`
- **Workflow:** [`n8n-conversao-cruzada-wgt.json`](./n8n-conversao-cruzada-wgt.json)
- **Endpoints:** [`GET /v1/deals`](../_docs/clint-api/endpoints/deals.md) · [`GET /v1/origins`](../_docs/clint-api/endpoints/origins.md)

---

## O que muda face à v1

A v1 dava a **lista** de quem converteu (numerador). Faltava o **denominador** para a taxa:
*de quantas pessoas que entraram primeiro pelo WGT?* Isso obriga a verificar o "1.º toque"
de **todos** os contactos do WGT, não só dos que ganharam. Por isso a v2 faz o **loop por
contacto sobre a base WGT inteira** — cada histórico dá, ao mesmo tempo:

- **Denominador** → o 1.º negócio (mais antigo) desse contacto é WGT? (originado no WGT)
- **Numerador** → desses, ganhou algo numa origem ≠ WGT?

---

## Fluxo

```
Executar manualmente
        ▼
1. GET /v1/origins                         mapa origem_id → nome
        ▼
2. GET /v1/deals?origin_id=WGT             (~8 páginas)  → todos os negócios WGT
        ▼
3. Contactos WGT únicos (Code)             dedup → ~milhares de contactos  (⚙ LIMITE_TESTE)
        ▼
4. GET /v1/deals?contact_id=…  ⟳ LOOP 15/1s   histórico completo de CADA contacto WGT
        ▼
5. Analisar (Code)                         1.º toque = WGT? ganhou fora? → agrega por pessoa
        ▼
6. Gerar CSV                               1 linha por pessoa
```

> **É a etapa mais demorada de todas as automações** — são milhares de chamadas (uma por
> contacto WGT). A ~15/s, alguns milhares de contactos levam **vários minutos**. Mas vês o
> nó 4 a avançar (progresso real) e a memória fica baixa. Para um ensaio rápido, poe
> `LIMITE_TESTE = 50` no topo do nó **"3. Contactos WGT únicos"**.

---

## O resultado

**No log do nó 5** (a resposta à Direção):

```
>> DENOMINADOR - originados no WGT (1.o negocio = WGT): N
>> NUMERADOR   - desses, converteram noutra origem: M
>> TAXA DE CONVERSAO CRUZADA: X%
   dos convertidos, com PRODUTO real (IGT/FGRS): P
   destinos: IGT 22=.. | Sessão Estratégica=.. | FGRS=..
```

**No CSV — uma linha por pessoa:**

| Coluna | Conteúdo |
|---|---|
| `contacto_id`, `nome`, `email`, `telefone` | quem é |
| `data_entrada_wgt` | 1.º negócio (WGT) |
| `n_ganhos_fora` | quantos negócios ganhou fora do WGT |
| `funis_ganhos` | lista dos funis (ex.: `IGT 22 \| FGRS 6`) |
| `ganhou_produto_real` | `SIM` se ganhou em IGT/FGRS (produto pago) |
| `produtos_reais` | quais |
| `valor_ganhos_eur` | soma dos valores **em EUR** (⚠️ pode conter prestações) |
| `moedas` | moedas envolvidas |
| `primeiro_ganho`, `ultimo_ganho`, `dias_ate_1o_ganho` | datas e tempo até converter |
| `total_negocios_contacto` | negócios totais no CRM |
| `flag_importacao_suspeita` | `SIM` se algum ganho é anterior à entrada |

---

## Leitura correta dos valores

- **`166.33` é uma prestação** de 499€ (÷3) — não somes valores às cegas como se fossem
  vendas inteiras.
- **Produto real** = `IGT*` e `FGRS*`. Os restantes (**Sessão Estratégica**, LDP, Cobrança,
  Lista de espera, Mentoria) são etapas internas — a coluna `ganhou_produto_real` separa-os.
- Há ganhos em **BRL** e alguns com **valor 0** (funis internos).

---

## ⚠️ Contaminação de janeiro

Ganhos anteriores à entrada no WGT ficam com `flag_importacao_suspeita`. O número seguro é
**M menos as flags**. A verificação de 1.º toque (loop) é precisamente o que separa quem
*entrou* pelo WGT de quem só *passou* por lá — na 1.ª execução, isto reduziu 262 "candidatos"
para 55 reais.

---

## Configuração

1. **Credencial:** `Clint api-token` (Header Auth, header `api-token`) — a única necessária.
2. **Importar** [`n8n-conversao-cruzada-wgt.json`](./n8n-conversao-cruzada-wgt.json).
3. Ligar a credencial nos **3 nós HTTP** (origens, WGT, histórico).
4. (Opcional) `LIMITE_TESTE = 50` no nó 3 para ensaiar; depois `0` para correr tudo.
5. **Executar** e acompanhar o nó 4. No fim, **"6. Gerar CSV"** → **Binary** → download.

> Para várias origens WGT, acrescenta os IDs ao array `WGT_ORIGINS` no nó 5.

---

_Documento interno LL Mídia. Fonte: OpenAPI oficial da Clint (`@clint-api/v1.0`).
Responde à Pergunta 1 (Direção) do dossiê "Nova Estrutura WGT 3.0"._
