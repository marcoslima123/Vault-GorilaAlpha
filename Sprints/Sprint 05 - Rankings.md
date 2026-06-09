# Sprint 05 — Rankings

**Esforço:** 2 dias · **Status:** ✅ concluído em 2026-05-21

← [[Sprints/Sprint 04 - Watchlist]] · próximo: [[Sprints/Sprint 06 - Compare]]

## Objetivo

Listas "Top N" por critério, usando o `StockIndicator` que já está populado. Feature de baixo custo e alto valor de marketing.

## Entregáveis

### API
- [ ] `GET /api/rankings?criteria=<key>&market=<B3|BDR|US>&limit=10`
- [ ] Critérios suportados (no mínimo):
  - `dy` — maior Dividend Yield
  - `roe` — maior ROE
  - `roic` — maior ROIC
  - `pl-asc` — menor P/L
  - `pvp-asc` — menor P/VP
  - `crescimento-lucro` — maior crescLucro
  - `liquidez-corrente` — maior liqCorrente
  - `margem-liquida` — maior margemLiquida
- [ ] Filtros de qualidade obrigatórios (evitar lixo): `volMedioDiario > X`, descartar nulls do critério.
- [ ] Cache de 1h via `unstable_cache` ou response headers.

### UI
- [ ] `RankingsPodium` — top 3 do critério ativo (visual diferenciado).
- [ ] `RankingsContenders` — posições 4 a 10 (free) ou 4 a 50 (pro).
- [ ] `RankingsInsights` — texto curto explicando o critério (ex: "DY alto sem checar payout pode mascarar empresa em declínio").
- [ ] `RankingsSidebarCard` — trocar critério (lista clicável).
- [ ] URL: `/rankings?criteria=dy&market=B3` (shareable).

### Paywall (do [[Decisoes/2026-05-20 - Modelo de Monetizacao]])
- [ ] Free: top 10. Pro: top 50.
- [ ] Filtros avançados (combinar 2+ critérios, ex: "DY alto E ROE alto"): pro-only.
- [ ] Banner discreto após posição 10 no free com CTA "Ver top 50 no Pro".

## Critério de pronto

- 8 critérios funcionam.
- Trocar critério não recarrega página inteira (client navigation).
- Resposta < 100ms (índices no Postgres em `StockIndicator.dy`, `.roe`, etc.).

## Notas técnicas

- Adicionar `@@index([dy])`, `@@index([roe])` etc. no `StockIndicator` se ainda não existem.
- Insights podem ser hardcoded inicialmente (1 parágrafo por critério). LLM-generated fica como upgrade futuro.

## Riscos

- Indicadores com `null` falseiam ranking. SQL precisa `WHERE indicator.dy IS NOT NULL AND volMedioDiario > 1000000`.
