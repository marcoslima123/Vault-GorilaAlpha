# 2026-05-20 — Sprint 03 (Stock Detail Real) concluído

← [[00 - Home]] · sessão anterior: [[Diario/2026-05-20 - Sprint 02 Screener concluido]]

## O que entregou

### Libs novas (puras, testáveis)
- `apps/web/src/lib/scoring.ts` — recebe indicadores e retorna scores 0-10 nos eixos: **valuation, qualidade, saúde financeira, dividendos, crescimento, geral**. Null-safe (retorna `null` quando sem dados). Função `ratingFromScore()` mapeia score → "FORTE/MÉDIO/FRACO/—".
- `apps/web/src/lib/analysis-heuristic.ts` — 12 regras descritivas com **tom neutro** (decisão da [[Decisoes/2026-05-20 - Mercados e IA]]). Retorna `{tese, pontosFortes, observacoes, pontosAtencao}`.
- `apps/web/src/lib/stock-presenter.ts` — transforma `Stock + Indicator` (Prisma) → props dos componentes (`SummaryRow`, `ScoreModule`, `AiAnalysisSection`, `BalanceRow`). Encapsula formatação multi-mercado (BRL/USD).

### Página `/stock/[ticker]` agora é server component real
- `apps/web/src/app/(app)/stock/[ticker]/page.tsx` — fetch via Prisma do stock + último indicator. `notFound()` se ticker não existe.
- `loading.tsx` — skeleton dos 6 cards principais.
- `not-found.tsx` — fallback amigável com link pro Screener.
- Disclaimer obrigatório no rodapé: "Material informativo. Não constitui recomendação de investimento."

### Componentes Stock* atualizados
| Componente | Mudança |
|---|---|
| `StockHeaderCard` | Trocou `signal: COMPRA/NEUTRO/VENDA` por `rating: FORTE/MÉDIO/FRACO/—`. Label "FUNDAMENTOS · {rating}" (decisão de tom descritivo). Score aceita `null`. 52w range no lugar de change-of-day (sem série histórica) |
| `StockInvestmentSummary` | Título "Investment Summary" → "Resumo". Removidos botões "Add to Watchlist" e "Deep Analysis" (entram no Sprint 04 e Sprint 07) |
| `StockScoreModules` | `score: number → number \| null`, mostra "—" quando null. Removido "6" hardcoded do título |
| `StockAiAnalysis` | "ANÁLISE IA · GORILLA LLM" → "ANÁLISE FUNDAMENTALISTA". Botão "Regenerar" só aparece se `onRegenerate` for passado |
| `StockBalanceSheet` | Mantido como está (só temos 1 snapshot atual — coluna "ATUAL") |
| `StockPriceChart` | Mantido com dados mock — não temos série histórica. Backlog: model `StockPriceHistory` + cron diário |

### Screener bonificado
- **Linha inteira é clicável** (não só o texto do ticker). Usa `router.push` no `onClick` da `<tr>`. Botão estrela e link do ticker fazem `stopPropagation` pra não duplicar navegação.
- `tabIndex={0}` + handler de teclado (Enter/Space) — acessibilidade.

## Validações

| Cenário | Resultado |
|---|---|
| `/stock/PETR4` (B3) | 200, renderiza com R$, P/L, ROE, scores reais ✅ |
| `/stock/AAPL` (US) | 200, renderiza com $, indicadores em US ✅ |
| `/stock/XYZ999INEXISTENTE` | not-found.tsx renderiza com link pro Screener ✅ |
| `/screener` → clicar linha | Navega pra `/stock/[ticker]` ✅ |
| Login com cookie persiste entre páginas | OK ✅ |

## Backlog descoberto

- **`notFound()` retorna status 200 em vez de 404** no Next 16 dev mode. Conteúdo correto, status HTTP errado. Não bloqueante mas vale investigar em produção.
- **`StockPriceChart` ainda mock** — precisa schema `StockPriceHistory` + cron diário. Backlog.
- **Sprint 04 vai trazer de volta o botão "Watchlist"** no `StockInvestmentSummary` (e funcional).
- **Sprint 07 (paywall)**: Botão "Regenerar análise IA" no `StockAiAnalysis` vai ativar quando análise IA pro estiver implementada.

## Decisões aplicadas

- **Tom descritivo, nunca prescritivo** — confirmado em todo lugar: rating "FORTE/MÉDIO/FRACO" em vez de "COMPRA/NEUTRO/VENDA", "ANÁLISE FUNDAMENTALISTA" em vez de "GORILLA LLM", regras heurísticas usam linguagem neutra ("indica", "observar"), disclaimer obrigatório.
- **Mercados multi-moeda funcionando** — `formatPrice/Large` respeitam `stock.currency` (BRL vs USD), demonstrado em `/stock/PETR4` (R$) vs `/stock/AAPL` ($).

## Próximo

[[Sprints/Sprint 04 - Watchlist]] — model + CRUD + ação "favoritar" funcional. Aí o botão estrela do Screener começa a fazer sentido.
