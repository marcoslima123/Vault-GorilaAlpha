# 2026-05-21 — Sprint 05 (Rankings) concluído

← [[00 - Home]] · sessão anterior: [[Diario/2026-05-21 - Sprint 04 Watchlist concluido]]

## O que entregou

### Service + tipos
- `apps/web/src/services/rankings/index.ts` — centraliza 8 critérios em `CRITERIA_CONFIG` com label, descrição descritiva, unidade (percent/ratio), campo do indicator e direção:
  - `dy` — Maior Dividend Yield
  - `roe` — Maior ROE
  - `roic` — Maior ROIC
  - `pl-asc` — Menor P/L
  - `pvp-asc` — Menor P/VP
  - `cresc-lucro` — Maior Crescimento de Lucro YoY
  - `liq-corrente` — Maior Liquidez Corrente
  - `margem-liquida` — Maior Margem Líquida
- Exporta `RankingsEntry`, `RankingsResponse`, `fetchRankings()`, validadores `isRankingCriteria/Market`.

### API
- `GET /api/rankings?criteria=&market=` — protegido por `guardSession()`.
- Aplica filtros de qualidade obrigatórios: `volMedioDiario ≥ 100k` e valor não nulo/finito.
- `pl-asc` / `pvp-asc` descartam valores ≤ 0 (empresas com prejuízo etc.).
- Lê `PLAN_LIMITS.rankingsTopN` (free=10, pro=50) via `getEffectivePlan`. Slice direto na resposta.
- Retorna `{criteria, criteriaLabel, criteriaDescription, market, plan, limit, entries}`.

### Componentes refatorados
| Componente | Mudança |
|---|---|
| `RankingsPodium` | Recebe `entries`, `criteriaLabel`, `criteriaDescription`, `onOpenCriteriaSelector`. Pódio 2-1-3 com cards `<Link href="/stock/[ticker]">`. Empty state se não há resultados |
| `RankingsContenders` | Lista rank 4 a N (linkável). Mostra setor, market cap, valor formatado |
| `RankingsInsights` | 3 cards: **Concentração por setor** (top 4 setores com % do total), **Como ler** (texto educativo neutro), **Por mercado** (B3/BDR/US counts) — todos calculados em tempo real a partir das entries |
| `RankingsSidebarCard` | Aceita `plan` + `limit`. Mostra paywall "FREE → Top 10, faça upgrade pra Top 50" ou banner Pro |

### Página `/rankings`
- Client component com URL state: `?criteria=&market=`
- Pills de mercado (Todos/B3/BDR/US) trocam filtro
- Popover com lista de critérios disparado pelo botão CRITÉRIO no podium
- Fetch ao mudar criteria ou market
- Loading state + error state
- Sidebar com plano (desktop only via `lg:flex`)

## Validações (via PowerShell)

| Cenário | Resultado |
|---|---|
| `criteria=dy` (top 10 free) | VULC3 48.45%, GRND3 29.15%, FESA4 18.64%... ✅ |
| `criteria=roe&market=B3` | BEEF3 83.75%, BBSE3 75.65%, CURY3 72.77%... ✅ |
| `criteria=pl-asc` | CI 2.51, BRSR6 3.44, JHSF3 3.45... ✅ |
| Critério inválido | 400 ✅ |
| Sem auth | 401 ✅ |
| Plano pro (após promoção via SQL) | top 50 entries retornadas ✅ |

## Decisão executada

[[Decisoes/2026-05-20 - Modelo de Monetizacao]] cumprida no Rankings:
- Free vê top 10
- Pro vê top 50
- Card lateral mostra a diferença e direciona pra `/settings` (placeholder de upgrade)

## Decisão executada (tom)

[[Decisoes/2026-05-20 - Mercados e IA]] cumprida:
- Descrições dos critérios são **observacionais** ("DY alto pode indicar boa distribuição, mas observe payout e sustentabilidade") — não prescritivas
- Card "Como ler" explica metodologia (vol min, sem prejuízo) sem dar recomendação
- Header da página tem o texto descritivo do critério, não promessa de retorno

## Estado atual do user (importante)

Para validar paywall do Sprint 05, promovi seu user `marcoslima_1212@hotmail.com` para **plan='pro'** via SQL direto. Pra voltar ao free e testar a paywall do Watchlist (limite 10), basta:

```sql
UPDATE subscriptions SET plan = 'free'
WHERE user_id = (SELECT id FROM users WHERE email = 'marcoslima_1212@hotmail.com');
```

## Próximo

[[Sprints/Sprint 06 - Compare]] (2 dias) — comparador 2-4 ativos via URL state, sem persistência. Reusa `lib/scoring.ts` do Sprint 03.

Ou **Sprint 3.5** (gráfico histórico com MM200/MM400 e filtros 1D/1W/3M/1Y/ALL) — backlog desde Sprint 03.
