# Sprint 03 — Stock Detail Real

**Esforço:** 2–3 dias · **Status:** ✅ concluído em 2026-05-20

← [[Sprints/Sprint 02 - Screener Funcional]] · próximo: [[Sprints/Sprint 04 - Watchlist]]

## Objetivo

Substituir o mock hardcoded de PETR4 em `/stock/[ticker]` por dados reais. Qualquer um dos 1410 tickers deve renderizar.

## Entregáveis

- [ ] Página `apps/web/src/app/(app)/stock/[ticker]/page.tsx` vira server component que chama `/api/stocks/[ticker]` (ou diretamente Prisma com `getCurrentSession()`).
- [ ] `notFound()` quando ticker não existe.
- [ ] Componentes recebem props do `StockIndicator` ao invés de constantes:
  - `StockHeaderCard` — ticker, name, price, change %, marketCap, signal/score
  - `StockInvestmentSummary` — P/L, P/VP, DY, ROE, etc.
  - `StockScoreModules` — cards de Valuation / Qualidade / Saúde Financeira / Crescimento (cada um lê N campos do indicator)
  - `StockBalanceSheet` — tabela com ativoTotal, patrimLiq, divBruta, divLiquida, caixa
  - `StockPriceChart` — placeholder por ora (não temos série histórica no schema; backlog)
  - `StockAiAnalysis` — implementar **versão heurística** já neste sprint (de [[Decisoes/2026-05-20 - Mercados e IA]]). Criar `apps/web/src/lib/analysis-heuristic.ts` com 15-20 regras descritivas (não prescritivas). Versão IA via Claude fica para o [[Sprints/Sprint 07 - Alerts e Paywall]] (pro-only).
- [ ] Remover qualquer `const mockStocks = { PETR4: ... }` do código.
- [ ] Loading skeleton via `loading.tsx`.
- [ ] Sinalizar visualmente se `StockIndicator.fetchedAt` está stale (> 15min) — chamar refresh inline?

## Critério de pronto

- `/stock/VALE3`, `/stock/AAPL`, `/stock/ITUB4` etc. renderizam com dados reais.
- Nenhum hardcoded de ticker em componentes.
- Tela carrega em < 500ms (cache do `StockIndicator`).

## Notas técnicas

- Score modules precisam de uma função de scoring. Sugestão: criar `apps/web/src/lib/scoring.ts` puro, recebe `StockIndicator` retorna `{ valuation: 0-100, quality: 0-100, ... }`. Mais fácil de testar.
- Gráfico de preço requer série histórica. Não temos hoje. Decidir: salvar histórico em novo model `StockPriceHistory` (cron diário) ou consumir Yahoo on-demand?
- Análise heurística: aproveitar `scoring.ts` + camada extra `analysis-heuristic.ts` que transforma scores em descrições.
- Formatação de moeda multi-mercado: criar `apps/web/src/lib/format-currency.ts` que respeita `stock.currency` (BRL vs USD).
- Disclaimer obrigatório no rodapé do `StockAiAnalysis`: "Material informativo. Não constitui recomendação de investimento."

## Riscos

- Tickers BDR têm nomes com pontos (`BERK34.SA`, `MSFT34.SA`). Conferir que rota dinâmica aceita.
