# 2026-05-27 — Inteligência Histórica da MM200

← [[00 - Home]] · sessão anterior: [[Diario/2026-05-27 - Grafico interativo Lightweight Charts]]

## Objetivo

Feature pedida pelo Marcos: analisar o **comportamento histórico do preço quando toca a MM200** (suporte) e mostrar estatísticas de retorno em linguagem acessível. Reaproveita os 5 anos de histórico já persistidos pelo gráfico.

## O que entregou (8 itens do spec)

### 1+2+3. Lógica (lib pura)
- `apps/web/src/lib/mm200-insights.ts` — `computeMM200Insights(series)`:
  - **Detecção de toque de suporte**: preço a ≤1.5% da MM200 + veio de cima (≥60% dos 5 dias anteriores acima da média) + recuperou em 5 dias (preço sobe ou volta acima da MM200). Cooldown de 15 dias evita contar o mesmo evento múltiplas vezes.
  - **Retornos por toque**: 30d, 60d, 90d + retorno máximo na janela de 90d.
  - **Agregação**: total de toques, % que subiu em 30d (successRate), médias 30/60/90d, melhor caso, pior caso, data do último toque.

### 7. Schema
- Model `TechnicalInsight` (ticker, indicator, touchCount, successRate, avgReturn30/60/90d, bestCase, worstCase, lastTouchDate, payload Json, calculatedAt, expiresAt). `@@unique([ticker, indicator])`. Migration `add_technical_insight`.

### 8. API
- `GET /api/stocks/[ticker]/technical-insights` — guardSession. Usa `getDailySeries` (exportada do chart-data, reusa cache 24h do Postgres). Calcula insights, cacheia em `TechnicalInsight` (expiresAt 24h, payload com toques individuais). Campos escalares pra query + payload Json pro detalhe.

### 4+5. Card no Stock Detail
- `components/app/mm200-insights.tsx` — card client com texto acessível: "Nas últimas N vezes que X tocou a MM200, em M (P%) o papel subiu nos 30 dias seguintes." 3 boxes de retorno médio (30/60/90d) coloridos, melhor/pior caso, último toque. Empty state se 0 toques. **Disclaimer obrigatório** sempre visível: "Desempenho passado não garante resultados futuros. Análise de caráter exclusivamente informativo e educacional."
- Integrado em `/stock/[ticker]` abaixo dos score modules.

### 6. Marcadores no gráfico
- `StockChart` agora busca `technical-insights` e marca os **toques de suporte** (não mais os cruzamentos genéricos) com círculos `belowBar`, **texto = retorno 30d** (ex: "+5.2%"). Filtra só os toques dentro do range visível do período.
- Nota: hover-tooltip específico do marcador não é nativo do lightweight-charts; o retorno aparece no texto do próprio marcador (visível direto).

## Validações

| Ticker | Toques | Sucesso 30d | Médias 30/60/90d | Melhor / Pior | Último |
|---|---|---|---|---|---|
| AAPL34 | 8 | 71% | +4.5% / +5.9% / +10.9% | +30.2% / -9.1% | 24/04/2026 |
| PETR4 | 6 | 33% | -0.4% / +1.1% / +5.7% | +54.1% / -23.2% | 26/11/2025 |
| VALE3 | 5 | 40% | -1.4% / -2.6% / +1.2% | +37.4% / -15.9% | 29/08/2025 |

- Página `/stock/AAPL34`: 200, card "COMPORTAMENTO HISTÓRICO NA MM200" presente, disclaimer presente, sem erros ✅
- Cache: AAPL34 retornou em 0.1s (banco) ✅

## Observações de design

- **successRate usa retorno 30d** como referência (se houver toques com 30d completo). Para o último toque, se ainda não passaram 30/60/90 dias, esse retorno é `null` e não entra na média daquela janela (honesto).
- Os números são insight estatístico de amostra pequena (5-8 toques em 5 anos) — por isso o disclaimer é obrigatório e proeminente.
- Detecção de suporte ≠ cruzamento simples. O gráfico antes marcava qualquer cruzamento; agora marca só os toques de suporte qualificados (com retorno).

## Resíduos / backlog (continua dos anteriores)

- Remover `/api/ping` (debug)
- Remover `StockPriceChart` SVG antigo + `/api/.../history` órfão
- `successRate` no model é Float não-null → salvo 0 quando null (o payload Json guarda o valor real com null). Funciona, mas é um detalhe.

## Lembrete de infra confirmado de novo

Criar a rota nova `/technical-insights` exigiu **rebuild da imagem** (não só restart) — mesmo padrão documentado em [[Diario/2026-05-27 - Grafico interativo Lightweight Charts]]. Rebuild + force-recreate resolveu.

## Próximo

Marcos vai testar no browser (/stock/AAPL34): o card de comportamento histórico + os marcadores no gráfico com os retornos.
