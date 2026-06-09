# Sprint 09 — Renda Fixa (BR + Internacional)

**Esforço:** ~1 dia · **Status:** 🟢 código completo (2026-06-09), tsc+lint OK, dados reais testados · 🟡 falta validação visual no browser

← [[Sprints/Sprint 08 - Feed de Relatorios de Corretoras]] · diário: [[Diario/2026-06-09 - Modulo Renda Fixa]]

## Objetivo

Módulo de Renda Fixa cobrindo Brasil e Internacional: comparar produtos, simular rendimento, ver curva de juros e ter análise IA — explicando o efeito dos juros globais para o investidor PF.

## Entregáveis

### Banco
- [x] `FixedIncomeRate` + `FixedIncomeCache` · migration `add_fixed_income`.

### Dados (`src/services/fixed-income/`)
- [x] BCB (Selic 432 / CDI 4389 / IPCA 13522) — **real**.
- [x] FRED treasuries (DGS3MO…DGS30) — **real** (`FRED_API_KEY`).
- [x] Produtos BR derivados do BCB (Tesouro Direto bloqueado por Cloudflare).
- [x] Câmbio (AwesomeAPI) + Europa (Yahoo, best-effort).
- [x] cache.ts (FixedIncomeCache), index.ts (getAllRates/syncRates).

### Lógica + API
- [x] `comparator.service` (IR progressivo, isenção, US→BRL, risco/liquidez, calculate).
- [x] `GET /rates`, `GET /compare`, `GET /calculator`, `POST /ai-analysis`.

### UI
- [x] RateCard, CurveChart, ComparatorTable, RateCalculator, AIAdvisor, tooltips educacionais.
- [x] Página `/fixed-income` com 4 tabs + link no sidebar.

### IA
- [x] `ai-advisor.service` — narrativa descritiva com simulação real. Validado com Claude.

## Critério de pronto

- [x] tsc + lint limpos.
- [x] BCB + FRED + comparador + advisor testados com dados reais.
- [ ] Validação visual no browser (4 abas) — pendente (religar dev).
- [ ] US products + câmbio confirmados no app real (sandbox filtrou).

## Notas técnicas

- **Tesouro Direto bloqueado (Cloudflare 403)** → produtos BR via BCB (Selic/CDI exatos; IPCA+/Prefixado "referência"). Decisão do Marcos.
- **Internacional via FRED** (2026-06-09): trocou Yahoo (Node ≥22, instável) por séries OCDE no FRED → DE/FR/IT/GB/JP/CA reais, **3M + 10Y cada**. **27 produtos / 8 países**. Câmbio ampliado (EUR/GBP/JPY/CAD). `international.service.ts`; `europe.service.ts` virou re-export. CurveChart reescrita: plota **todos os países como linhas** (TREASURY/BOND) com legenda. Cache (`FixedIncomeCache`) precisa ser limpo ao mudar a forma dos dados (TTL 1h).
- **Aba Histórico** (2026-06-09): `history.service.ts` (FRED `observation_start`+`frequency=m` / BCB `dataInicial`, downsample mensal, cache 6h) + `GET /api/fixed-income/history`. Componente `historico.tsx` com toggle **Taxa única** (seletor: Selic/CDI/IPCA/US/DE/JP… + 3M) e **Países** com sub-toggle **10Y/3M** (comparação no tempo), filtro de período 1A/5A/10A/Máx, eixo X = datas reais. FRED não tem BR 10Y → BR usa Selic (10Y) / CDI (3M) como proxy. Distinção importante: **curva = prazo (taxas de hoje)**; **histórico = tempo**.
- **Logo TradingView** removido de todos os gráficos lightweight-charts (`attributionLogo: false` em `FiLineChart` + `StockChart`). 2 erros de lint pré-existentes no `stock-chart.tsx` corrigidos (setState em effect → função async; ref no render → state `containerWidth`).
- Global fetch (sem node-fetch). guardSession em todas as rotas.
- Câmbio = 0 no sandbox de teste; funciona no app (igual /api/quotes).

## Backlog/evolução

- Fonte exata do Tesouro (CSV Tesouro Transparente ou taxas manuais admin).
- Subir Node ≥22 pra bonds europeus.
- Paywall (renda fixa internacional só no pro?).
- Cron pra `syncRates` periódico.
