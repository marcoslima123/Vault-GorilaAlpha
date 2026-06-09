# 2026-05-20 — Sprint 02 (Screener Funcional) concluído

← [[00 - Home]] · sessão anterior: [[Diario/2026-05-20 - Sprint 01 Hardening concluido]]

## Banco repopulado (limpeza necessária antes do sprint)

- Estado anterior: schema do Prisma à frente do banco. Banco tinha `subsector`, mas Prisma esperava `industry`. `/api/stocks` retornava 500.
- 3 migrations pendentes (`yahoo_migration`, `auth_tables`, `profile_and_reset`).
- A `yahoo_migration` adicionava `fetched_at NOT NULL` que conflitaria com indicators populados.
- Decisão: `prisma migrate reset --force` (DB em dev, sem dado precioso) + repopular via seed.
- Seed.ts refatorado pra chamar `runSync` direto em vez de fetch HTTP — agora não depende do `/api/sync` que virou admin-only no Sprint 01.
- Pós-seed: **671 stocks** populados (B3: 146, BDR: 35, US: 490). 44 erros (tickers que sumiram do Yahoo Finance).

## Sprint 02 entregue

Arquivos criados/modificados em `gorilaAlpha-web/apps/web/src`:

| Arquivo | Mudança |
|---|---|
| `lib/format.ts` | reescrito com `formatPrice/Large/Ratio/Percent/Change` parametrizados por `Currency` (BRL/USD) |
| `services/stocks/index.ts` | **novo** — tipos `Stock`, `StockIndicator`, `StocksFilters` + `fetchStocks()`, `buildStocksQuery()` |
| `components/app/screener-filters.tsx` | reescrito — URL state via `useSearchParams` + `router.replace`, debounce 300ms no search, Lupa icon, botão Limpar |
| `components/app/screener-table.tsx` | reescrito — props `{stocks, isLoading}`, skeleton 8 rows, empty state, link `/stock/[ticker]`, formatação por currency |
| `app/(app)/screener/page.tsx` | reescrito — lê filtros da URL, dispara `fetchStocks`, paginação "carregar mais" via offset, gerencia loading/error |
| `app/api/stocks/route.ts` | **bug crítico consertado** — filtros numéricos rodavam em memória DEPOIS do Prisma cortar em `take`. Agora busca até 2500, filtra, depois pagina via slice |
| `apps/web/scripts/seed.ts` | refatorado pra chamar `runSync` direto (sem HTTP, sem auth dependency) |

## Validações (via PowerShell + curl)

| Cenário | Resultado |
|---|---|
| `?market=B3&maxPl=10` | 49 stocks ✅ |
| `?minRoe=0.15` | 308 stocks ✅ |
| `?market=B3&minDy=0.05&maxPl=15` | 33 stocks (BBSE3, BLAU3, BRAP4, ...) ✅ |
| `?search=PETR` | PETR3, PETR4 + petro* ✅ |
| `?limit=3&offset=2` (B3) | 3 stocks começando em ALPA4 ✅ |
| `/screener` deslogado | 307 redirect (middleware Sprint 01) ✅ |

## Achados e backlog

- ⚠️ **Filtros rodam em memória** (até 2500 stocks). Funciona pro tamanho atual; pra escalar precisa mover para `where` no Prisma com 1-to-1 `currentIndicator` ou snapshot. Tem 671 stocks hoje, então sem urgência.
- ⚠️ **Yahoo Finance perde tickers**: 44 erros no seed. Lista de tickers padrão tem stocks que não existem mais (KCBR3, ZAMP3, etc.). Backlog: limpar lista de defaults.
- ⚠️ **Cache do Next dev**: ao salvar mudanças no `/api/stocks/route.ts`, Next.js manteve módulo cacheado e filtro não atualizou. Resolveu com `docker restart app`. Em produção não é problema.
- ⚠️ **`StatsCards` ainda hardcoded** ("8,512 ativos analisados"). Backlog rápido pro próximo sprint.
- ⚠️ **Sort por coluna** não implementado (fora do escopo). Backlog pequeno.

## Próximo

[[Sprints/Sprint 03 - Stock Detail Real]] — `/stock/[ticker]` ainda usa mock PETR4 hardcoded. Trocar pelo `/api/stocks/[ticker]` real + implementar análise heurística da [[Decisoes/2026-05-20 - Mercados e IA]].
