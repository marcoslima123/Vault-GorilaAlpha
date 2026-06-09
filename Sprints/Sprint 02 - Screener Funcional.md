# Sprint 02 — Screener Funcional

**Esforço:** 2–3 dias · **Status:** ✅ concluído em 2026-05-20

← [[Sprints/Sprint 01 - Hardening de Auth]] · próximo: [[Sprints/Sprint 03 - Stock Detail Real]]

## Objetivo

Transformar `/screener` de esqueleto para tela funcional usando os 1410 stocks já populados via `/api/stocks`.

## Entregáveis

- [x] **Banco repopulado** após reset de schema (`prisma migrate reset --force` aplicou as 4 migrations). 671 stocks (B3:146, BDR:35, US:490). Seed.ts refatorado pra chamar `runSync` direto (sem HTTP).
- [x] `apps/web/src/lib/format.ts` — `formatPrice/Large/Ratio/Percent/Change` lidando com `Currency = 'BRL' | 'USD'`. Formatação multi-mercado da [[Decisoes/2026-05-20 - Mercados e IA]].
- [x] `apps/web/src/services/stocks/index.ts` — tipos `Stock`, `StockIndicator`, `StocksFilters`, `StocksResponse` + `fetchStocks(filters)` e `buildStocksQuery()`.
- [x] `ScreenerFilters` reescrito:
  - Estado lido/escrito da URL (`useSearchParams` + `router.replace`)
  - SearchInput (debounce 300ms) com `LupaIcon`
  - Select Mercado (Todos/B3/BDR/US)
  - Setor (text), P/L max, ROE min, DY min (todos number inputs)
  - Botão "Limpar" só aparece quando há filtro ativo
- [x] `ScreenerTable` reescrito:
  - Recebe `stocks[]` e `isLoading` via props
  - Colunas: TICKER (link), SETOR, PREÇO, MARKET CAP, P/L, P/VP, ROE, DY, ⭐
  - Skeleton rows quando loading inicial
  - Empty state com mensagem
  - Link `/stock/[ticker]` no ticker
  - Botão estrela placeholder (entra no [[Sprints/Sprint 04 - Watchlist]])
- [x] `screener/page.tsx` coordena:
  - Parse de filtros da URL para `StocksFilters`
  - Reset de paginação quando filtros mudam
  - `fetchStocks()` com limit 50 + offset
  - Botão "Carregar mais" incrementa offset
  - Mostra erro se fetch falhar
  - Contador "N ações carregadas"
- [x] **Bug crítico do `/api/stocks` consertado** — filtros numéricos rodavam em memória depois do Prisma já ter cortado em `take`. Mudei pra buscar até 2500 stocks, aplicar filtros, depois paginar.

## Não fica neste sprint (intencional)

- ❌ `StockDetailPanel` lateral removido da página — clique no ticker navega para `/stock/[ticker]` (Sprint 03 vai consertar essa página).
- ❌ Sort por coluna — backlog; ordenação atual é alfabética por ticker.
- ❌ StatsCards com dados reais — ainda mock. Backlog rápido.
- ❌ Filtros numéricos no `where` do Prisma (server-side de verdade) — funciona com 671 stocks via in-memory; vira problema em escala >10k.

## Critério de pronto

- ✅ Filtrar `B3 + maxPl=10` retorna 49 stocks via API. `B3 + minDy=0.05 + maxPl=15` retorna 33 stocks.
- ✅ ROE >= 15% retorna 308 stocks.
- ✅ Search `PETR` retorna PETR3, PETR4 + outras "petro*".
- ✅ URL preserva filtros (compartilhável).
- ✅ Paginação via offset funciona (`limit=3&offset=2` retorna 3 stocks começando em ALPA4 dentre os 146 B3).
- ✅ Sem mock data no componente.
- ⏳ p50 < 200ms: não medi formalmente, mas resposta foi instantânea no PowerShell.

## Notas técnicas

- API já retorna paginado: `?limit=50&offset=0`. Adicionar `total` no payload se ainda não existe.
- Distinct de sectors: criar endpoint `GET /api/stocks/sectors?market=` (5 linhas, cache 1h).
- Sort: adicionar `?sortBy=&sortDir=` na API. Validar no Zod schema.
- Server component para SSR inicial + client component para interatividade dos filtros.

## Riscos

- Tabela com 1410 linhas sem virtualização trava. Usar paginação ou `@tanstack/react-virtual` (avaliar custo de adicionar).
