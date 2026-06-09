# 2026-06-09 — Módulo Renda Fixa (BR + Internacional)

← [[00 - Home]] · sessão anterior: [[Diario/2026-06-08 - Feed teste visual e wiring WhatsApp]] · sprint: [[Sprints/Sprint 09 - Renda Fixa]]

## Objetivo

Módulo completo de Renda Fixa cobrindo Brasil e Internacional: dados reais (Tesouro/BCB/FRED/Yahoo), comparador com IR e câmbio, calculadora, curva de juros, análise IA e tooltips educacionais. Stack: Next.js 16 + Prisma + Postgres + Claude API.

## Entregue (7 partes, na ordem do brief)

### Parte 2 — Schema
- Models `FixedIncomeRate` (country/category/name/ticker/maturity/rate/rateType/currency/minAmount, unique [country,ticker,maturity]) e `FixedIncomeCache` (cacheKey unique, data Json, expiresAt). Migration `add_fixed_income`.

### Parte 1 — Serviços de dados (`src/services/fixed-income/`)
- `cache.ts` — helper `cached(key, ttl, fetcher)` via `FixedIncomeCache` (1h taxas / 5min câmbio).
- `brazil.service.ts` — BCB **funciona** (Selic série 432, CDI 4389, IPCA 13522). **Tesouro Direto bloqueado por Cloudflare (403)** → decisão do Marcos: **produtos derivados do BCB** (Tesouro Selic, CDB 100/110/120% CDI, LCI 95%, Poupança = exatos; Tesouro Prefixado/IPCA+ = "referência" marcada).
- `usa.service.ts` — **FRED funciona** (DGS3MO…DGS30, 7 treasuries). `FRED_API_KEY` no .env.
- `europe.service.ts` — Yahoo (GDBR10/GUKG10) best-effort; `yahoo-finance2` exige Node ≥22 (temos 20.11) → retorna vazio sem quebrar.
- `fx.service.ts` — câmbio via AwesomeAPI (USD/EUR/GBP-BRL). 0 no sandbox de teste, funciona no app (já usado em /api/quotes).
- `index.ts` — `getAllRates`, `getBrazilIndicators`, `getFxRates`, `syncRates` (upsert no FixedIncomeRate).
- Usei **global fetch** (não instalei node-fetch — Node 20 já tem).

### Parte 4 — Comparador (`comparator.service.ts`)
- `compareFixedIncome(amount, years)`: por produto calcula bruto/líquido, IR progressivo (22,5/20/17,5/15%), isenção LCI/LCA/Poupança, US→BRL via câmbio (filtra se câmbio=0), risco/liquidez. Compounding correto: IPCA+/SELIC+ = ((1+base)(1+spread))^anos.
- `calculate(...)` p/ a calculadora (série mensal + benchmark CDI).
- Validado: R$10k/1ano → CDB 120% líquido R$11.426, LCI isenta R$11.368, etc.

### Parte 3 — API Routes
- `GET /rates` (filtros country/category, dispara syncRates), `GET /compare`, `GET /calculator`, `POST /ai-analysis`. Todas guardSession.

### Parte 6 — Componentes + página
- `components/app/fixed-income/`: shared (CountryBadge texto/sem emoji, RiskBadge, **InfoTooltip + tooltips educacionais da Parte 7**), RateCard, CurveChart (SVG US+BR), ComparatorTable (ordenável), RateCalculator (gráfico+CDI), AIAdvisor.
- Página `(app)/fixed-income` com tabs [Visão Geral][Calculadora][Comparativo][Análise IA]. Link **Renda Fixa** no sidebar (após Rankings).

### Parte 5/7 — IA advisor + tooltips
- `ai-advisor.service.ts` — alimenta o Claude com a simulação real (comparação calculada) → narrativa em tom descritivo (4 parágrafos). Validado com Claude real: cita números reais, explica efeito dos juros americanos, FGC, etc.
- Tooltips educacionais (Selic/CDI/Fed/Duration/IPCA+) em `shared.tsx`, usados na Visão Geral.

## Caveats honestos

- **Tesouro Direto**: feed oficial bloqueado por Cloudflare. Produtos BR vêm do BCB (Selic/CDI exatos; IPCA+/Prefixado como referência). Pra taxas exatas por título no futuro: CSV do Tesouro Transparente (pesado) ou taxas manuais.
- **Câmbio (US→BRL)** e **Europa (Yahoo)**: funcionam no app real; no sandbox de teste o câmbio veio 0 (US filtrado) e Yahoo exige Node ≥22.
- Tudo passa **tsc + lint**. APIs/serviços testados com dados reais; **falta validar visualmente no browser** (dev estava parado).

## Próximo

- Marcos religar `pnpm dev:web` e abrir `/fixed-income` pra ver as 4 abas.
- Eventual: subir Node ≥22 (Europa) e/ou fonte exata do Tesouro.
