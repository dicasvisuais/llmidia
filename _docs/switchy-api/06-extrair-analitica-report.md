# Extrair analítica diária pela página pública de report

> Método LL Mídia, validado em 2026-09-17. Contorna a ausência de cliques por data na API ([`05-schema-graphql.md`](./05-schema-graphql.md)).

Cada link tem um **link público de report** (`Copy the report link` na UI): `https://www.switchy.io/report/<hash>:<token>`. Não exige login e serve toda a analítica — cliques por dia, países, browsers, OS, devices, referrers.

## Como funciona por baixo

- SPA Angular; os dados chegam por **Firebase Realtime Database** (projeto `urlshortener-f1125`, `https://urlshortener-f1125.firebaseio.com`) por **WebSocket** — não há XHR para interceptar.
- O gráfico é **amCharts v4**, empacotado via webpack (não exposto em `window`), por isso não se lê `chart.data` directamente.
- Não existe botão de export.

## Receita que funciona

Abrir o report num browser controlável e extrair do **SVG renderizado**:

1. **Calibrar o eixo Y** pelos rótulos numéricos → `escala = (vMax - vMin) / (yMin - yMax)` px→cliques.
2. **Apanhar as barras**: `<path>` com `fill: rgb(0, 120, 255)`, usando `getBoundingClientRect()` (o `getBBox()` devolve coordenadas locais, todas em 0,0 — não serve).
3. **Mapear barra→data pela posição x**, não pelos rótulos: em meses cheios o eixo só rotula dias alternados, e o casamento por rótulo perde barras.
4. **Aplicar offset de −1 dia**: as colunas do amCharts ficam centradas *entre* gridlines, pelo que o índice cru dá sempre um dia a mais.

### Duas armadilhas

**O gráfico só renderiza quando entra no viewport.** Sem `scrollIntoView` + `scrollBy(0, 420)` e ~2s de espera, não há barras nenhumas no DOM. Foi a causa de várias extracções vazias.

**O intervalo de datas é `[início, fim)` — exclui o último dia.** Um intervalo de um só dia devolve *"There is not click on your link"*. Verificado:

| Intervalo | Devolve |
|---|---|
| 2 Fev → 3 Fev | 3823 (só dia 2) |
| 2 Fev → 4 Fev | 4452 (dias 2+3) |
| 1 Fev → 3 Fev | 3825 (dias 1+2) |

Consequência: seleccionar "mês inteiro" no calendário **perde o último dia de cada mês**. Para o apanhar, escolher `último dia → dia 1 do mês seguinte`.

## Granularidade

O eixo adapta-se ao intervalo: "All time" agrega por **mês**; um intervalo de ~1 mês dá **dia**. Para série diária completa, iterar mês a mês pelo **Custom Range**.

## Validação

Dois controlos que confirmam a extracção:

1. A soma das barras diárias tem de bater com o *stat tile* **Clicks** do mesmo intervalo.
2. A soma de todos os meses tem de bater com o total "All time".

No `aula1-sms`: 147 dias com cliques, soma **13.814** = total "All time". Dados em [`dados/aula1-sms-cliques-diarios.csv`](./dados/aula1-sms-cliques-diarios.csv).

> O report público é frágil por natureza — é raspagem de UI, não API. Qualquer mudança no front-end da Switchy parte isto. E o link de report é **público**: quem o tiver vê a analítica toda.
