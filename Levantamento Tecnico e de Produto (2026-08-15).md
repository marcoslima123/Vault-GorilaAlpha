# Levantamento Técnico e de Produto — GorilaAlpha

> **Gerado em 2026-08-15** por leitura direta do código em `C:\Users\Marcosdeli\Desktop\Projetos\GorilaAlpha\gorilaAlpha-web` (branch `dev`).
> Destinado a análise estratégica de produto e planejamento de features.
> Marcações **[VERIFICAR]** indicam informação que não pôde ser confirmada no código.
>
> ⚠️ **Números de banco são do ambiente LOCAL.** Produção não foi consultada nesta sessão.
> ⚠️ **A working tree está suja e nada foi commitado** — parte do que está descrito aqui existe só na máquina local.

Ligações: [[00 - Home]] · [[03 - Roadmap]] · [[Deploy - Railway (Producao)]]

---

# 1. VISÃO GERAL DO PROJETO

## 1.1 Formato do repositório

**Monorepo único** (pnpm workspaces + Turborepo). Não são repos separados.

```
gorilaAlpha-web/
├── apps/
│   ├── web/          @gorila/web    :3000  — desktop + TODA a API + Prisma + integrações
│   └── pwa/          @gorila/pwa    :3001  — mobile/PWA, sem banco e sem API própria
├── packages/
│   ├── ui/           @gorila/ui             — ícones, tokens, componentes compartilhados
│   ├── core/         @gorila/core           — client HTTP, sessão, tipos, scoring
│   └── config/       @gorila/config         — tsconfig e eslint base
├── docker/Dockerfile                        — multi-stage do docker-compose local
├── Dockerfile / Dockerfile.pwa / Dockerfile.whatsapp   — Railway (3 serviços)
├── railway.json / railway.pwa.json / railway.whatsapp.json
└── .github/workflows/   check-alerts.yml · sync-stocks.yml · quality-gate.yml
```

**Regra arquitetural central:** o web é o único backend. O PWA não tem rotas de API — `apps/pwa/src/app/api/[...path]/route.ts` é um proxy catch-all que repassa tudo para `WEB_INTERNAL_URL` preservando cookies (inclusive `set-cookie`).

`packages/ui` e `packages/core` são publicados como **TypeScript cru** (`main` → `src/index.ts`), por isso ambos os apps declaram `transpilePackages`.

## 1.2 Stack técnica

| Camada | Tecnologia | Versão |
|---|---|---|
| Runtime | Node.js | `>=22.0.0` |
| Gerenciador | pnpm | `10.33.2` |
| Orquestração | Turborepo | `^2.5.0` |
| Framework | Next.js (App Router) | `16.2.2` |
| UI | React / React DOM | `19.2.4` |
| Linguagem | TypeScript | `5.9.3` |
| Estilo | Tailwind CSS | `^4` |
| ORM | Prisma + @prisma/client | `^5.22.0` |
| Banco | PostgreSQL | `16-alpine` |
| Auth (hash) | @node-rs/argon2 | `^2.0.2` |
| Auth (JWT) | jose | `^6.2.3` |
| IA | @anthropic-ai/sdk | `^0.90.0` |
| Gráficos | lightweight-charts | `^5.2.0` |
| WhatsApp | @whiskeysockets/baileys | `6.7.23` (pinado) |
| PDF | unpdf | `^1.6.2` |
| Email | resend | `^6.12.2` |
| Dados US | yahoo-finance2 | `^3.14.0` |
| Validação | zod | `^4.4.2` |
| Testes | vitest | `^4.1.10` |
| Mutação | @stryker-mutator/core | `^9.6.1` |
| Fronteiras | dependency-cruiser | `^18.2.0` |

**Dependências presentes mas de uso duvidoso:**

| Pacote | Situação |
|---|---|
| `next-auth` `5.0.0-beta.31` + `@auth/prisma-adapter` | Instalados, mas a auth é **JWT próprio**. Provável resíduo. **[VERIFICAR]** se algo ainda importa |
| `lucide-react` `^1.7.0` | Instalado, mas o `CLAUDE.md` **proíbe** ícones de libs externas |
| `shadcn` `^4.2.0` | Instalado como dependência de runtime (normalmente é CLI de dev) |
| `xlsx` `^0.18.5` | Em devDependencies do web. **[VERIFICAR]** uso |
| `@react-oauth/google` | Login Google — este é usado de fato |

## 1.3 Como o projeto roda

**Scripts na raiz:**

| Comando | O que faz |
|---|---|
| `pnpm dev` | web (3000) + pwa (3001) |
| `pnpm dev:web` / `pnpm dev:pwa` | apenas um dos dois |
| `pnpm build` / `build:web` / `build:pwa` | build via Turbo |
| `pnpm lint` / `pnpm type-check` | eslint / tsc em todos os workspaces |
| **`pnpm gate`** | **quality gate completo** (ver 6.6) |
| `pnpm gate:slow` | gate com etapas lentas |
| `pnpm clean` | limpa e remove node_modules |

**Por workspace:**

```bash
pnpm --filter @gorila/web type-check
pnpm --filter @gorila/web db:migrate      # prisma migrate dev
pnpm --filter @gorila/web db:studio
pnpm --filter @gorila/web seed            # tsx scripts/seed.ts
pnpm --filter @gorila/web whatsapp        # worker Baileys
pnpm --filter @gorila/web test            # vitest run
```

**Docker local:**

```bash
docker compose up -d
# web http://localhost:3000 · pwa http://localhost:3001 · postgres localhost:5433
```

O `docker-compose.override.yml` roda o target `dev` com bind-mount `.:/app` e **volumes anônimos** para todos os `node_modules` e `.next`.

> ⚠️ **Armadilha conhecida e recorrente:** os volumes anônimos sobrevivem a `up -d` e `restart` e têm precedência sobre a imagem. Rebuildar não resolve. O comando correto é `docker compose up -d --force-recreate --renew-anon-volumes app`.

**Scripts operacionais** (`apps/web`, via `pnpm exec tsx scripts/<arquivo>.ts`): `create-admin`, `seed`, `seed-reports`, `warm-charts`, `clear-chart-cache`, `cleanup-reports`, `list-reports`, `inspect-report`, `translate-reports`.

## 1.4 Camadas dentro de `apps/web/src`

| Pasta | Papel |
|---|---|
| `app/(app)/` | telas autenticadas |
| `app/api/` | 49 rotas REST |
| `app/` (resto) | páginas públicas (landing, login, planos, registrar…) |
| `services/` | regra de negócio e I/O externo, um subdiretório por domínio |
| `lib/` | infra transversal (db, jwt, guards, plan, anthropic, rate-limit, formatação, scoring) |
| `models/` | tipos de domínio server↔client |
| `types/` | tipos de fixed-income, reports, ícones |
| `viewmodels/` | hooks `"use client"` que fazem fetch e expõem `{ data, loading, error, refetch }` |
| `components/` | `ui/` primitivos · `app/` telas autenticadas · `auth/` · `landing/` |

O PWA espelha de forma enxuta: `components/{app,shell,ui}`, `lib/` (helpers de apresentação **duplicados de propósito**) e consome sessão/HTTP de `@gorila/core`.

---

# 2. FEATURES IMPLEMENTADAS

## 2.1 Autenticação e onboarding

**a) Implementado e funcionando**

JWT próprio (não NextAuth) em cookies httpOnly `ga_access` / `ga_refresh`. Cadastro por email+senha com verificação por código OTP, login Google (OAuth), recuperação de senha, refresh automático de sessão.

Onboarding em 3 passos: `registrar` → `registrar/perfil` (experiência, objetivo, patrimônio, data de nascimento) → `registrar/preferencias` (mercados, setores, volume mínimo, alertas).

No client, `SessionProvider`/`useSession` de `@gorila/core/session` carrega `/api/auth/me`, faz refresh a cada 50min, no `visibilitychange` e em qualquer 401.

**b) Arquivos**

| Camada | Arquivos |
|---|---|
| Libs | `lib/jwt.ts`, `auth-cookies.ts`, `auth-session.ts`, `auth-tokens.ts`, `auth-validation.ts`, `argon.ts`, `otp.ts`, `email.ts`, `google-id-token.ts`, `rate-limit.ts`, `signup-draft.ts`, `api-guards.ts` |
| Serviço | `services/auth/index.ts`, `services/profile/index.ts` |
| Componentes web | `components/auth/` — `login-screen`, `register-screen`, `register-profile-screen`, `register-preferences-screen`, `verify-email-screen`, `forgot-password-screen`, `reset-password-screen`, `welcome-screen`, `google-sign-in-button`, `password-field`, `pill`, `step-indicator`, `auth-shell`, `auth-provider` |
| Componentes PWA | `(auth)/` rotas espelhadas + `components/ui/otp-input.tsx`, `auth-header.tsx`, `text-field.tsx` |
| Core | `packages/core/src/auth.ts`, `session.tsx` |

**c) API routes** — todas públicas por natureza

`POST /api/auth/register` · `login` · `logout` · `refresh` · `verify` · `resend` · `forgot-password` · `reset-password` · `google` · `GET /api/auth/me`

**d) Models Prisma** — `User`, `Account`, `EmailVerification`, `RefreshToken`, `PasswordReset`, `UserPreferences`

**e) Integrações** — Resend (envio de OTP e reset), Google OAuth

**f) Incompleto / placeholder**

- **"Continuar com Apple"** existe nas telas de login e registro, mas está `disabled` com badge "em breve" (`login-screen.tsx:159`, `register-screen.tsx:184`)
- Magic-link consta como pendência no vault, não implementado

**g) Limitações**

- `next-auth` e `@auth/prisma-adapter` instalados mas não usados — peso morto e fonte de confusão
- `rate-limit.ts` é **[VERIFICAR]** se in-memory (se for, não funciona com múltiplas instâncias)
- Model `Account` existe para OAuth, mas só Google está implementado

---

## 2.2 Screener de ações

**a)** Filtros por mercado, setor, P/L (min/max), ROE mínimo, DY mínimo, volume mínimo e busca textual. Paginação por `limit`/`offset`. Estado na URL.

**b)** Web: `app/(app)/screener/page.tsx`, `components/app/screener-filters.tsx`, `screener-table.tsx`, `viewmodels/useStocks.ts`, `services/stocks/index.ts`
PWA: `app/(app)/screener/page.tsx`, `components/app/screener-filter-sheet.tsx`, `stock-row.tsx`

**c)** `GET /api/stocks` — **rota PÚBLICA, sem guard**. Limites: `DEFAULT_LIMIT=50`, `MAX_LIMIT=500`, `MAX_BUFFER=2500`

**d)** `Stock`, `StockIndicator`

**e)** Nenhuma direta (lê do banco)

**f)** Sem itens marcados

**g) Limitações**

- **`/api/stocks` é público** — qualquer um pode extrair a base inteira em páginas de 500. O middleware `proxy.ts` protege páginas, não a API
- `MAX_BUFFER = 2500` sugere filtragem em memória, não no SQL — **[VERIFICAR]** impacto de performance conforme a base cresce

---

## 2.3 Stock detail (página da ação)

**a)** Página completa por ticker: header com preço, variação e frescor do dado; gráfico de preço com médias móveis; score em 5 pilares com anel visual; módulos de score detalhados; balanço; resumo de investimento; insights MM200; consenso de relatórios; botões de favoritar, criar alerta e forçar refresh; análise IA.

Desde 07/08 exibe **frescor da cotação** (`PriceFreshness`): fresco → `Cotação agora · atraso de 15min da bolsa`; velho → `⚠ Cotação de Xd atrás`; sem dado → `Cotação indisponível`.

**b)** Web: `app/(app)/stock/[ticker]/page.tsx` (com `requireSession`), `components/app/stock-header-card.tsx`, `stock-chart.tsx`, `stock-price-chart.tsx`, `stock-score-modules.tsx`, `stock-balance-sheet.tsx`, `stock-investment-summary.tsx`, `stock-detail-panel.tsx`, `stock-ai-analysis.tsx`, `score-ring.tsx`, `mm200-insights.tsx`, `consensus-widget.tsx`, `create-alert-button.tsx`, `favorite-toggle-button.tsx`, `refresh-stock-button.tsx`; `lib/stock-presenter.ts`
PWA: `app/(app)/stock/[ticker]/page.tsx`, `components/app/stock-chart.tsx`, `score-module-card.tsx`, `mm200-insights.tsx`; `lib/stock-presenter.ts`, `lib/score.ts`

**c)**

| Rota | Método | Guard |
|---|---|---|
| `/api/stocks/[ticker]` | GET | **PÚBLICA** |
| `/api/stocks/[ticker]/history` | GET | guardSession |
| `/api/stocks/[ticker]/price-chart` | GET | guardSession |
| `/api/stocks/[ticker]/refresh` | POST | guardSession |
| `/api/stocks/[ticker]/technical-insights` | GET | guardSession |
| `/api/stocks/[ticker]/ai-analysis` | POST | guardSession + plano |

**d)** `Stock`, `StockIndicator`, `StockQuote`, `StockPriceHistory`, `TechnicalInsight`, `AiAnalysis`

**e)** brapi PRO (B3/BDR), Yahoo Finance (US), Twelve Data (candles US)

**g) Limitações**

- Preço tem **teto de 15min de atraso** para B3 por regra da bolsa. Tempo real exigiria feed licenciado
- `stock/page.tsx` (sem ticker) existe e **não tem `requireSession`** — **[VERIFICAR]** o que renderiza
- **`/api/stocks/[ticker]` é público** — expõe fundamentos completos sem login

---

## 2.4 Análise IA (Claude API)

**a)** Três usos distintos do Claude, todos com `AI_MODEL = "claude-haiku-4-5-20251001"`:

| Uso | Onde | Gate |
|---|---|---|
| Análise fundamentalista da ação | `lib/analysis-ai.ts` | `aiAnalysis` (Pro) |
| Consultor de renda fixa | `services/fixed-income/ai-advisor.service.ts` | guardSession |
| Extração estruturada de PDFs | `services/reports/report-processor.ts` | pipeline interno |

**Estratégia híbrida:** `analysis-heuristic.ts` gera análise determinística (free) e `analysis-ai.ts` gera via Claude (pro). Se `hasAnthropicKey()` falhar, retorna `null` e cai na heurística.

**Cache:** `AiAnalysis` com TTL de **7 dias** (`CACHE_TTL_MS` em `analysis-ai.ts`), unique em `[stockId, model]`.

**b)** `lib/anthropic.ts` (client via Proxy lazy no `globalThis`), `lib/analysis-ai.ts`, `lib/analysis-heuristic.ts`, `components/ai-analysis.tsx`, `components/app/stock-ai-analysis.tsx`, `ai-analysis-loader.tsx`

**c)** `POST /api/stocks/[ticker]/ai-analysis` (guardSession + plano) · `POST /api/analysis` (**guardAdmin**) · `POST /api/fixed-income/ai-analysis` (guardSession)

**d)** `AiAnalysis`

**e)** Anthropic Messages API. Usa `cache_control: { type: "ephemeral" }` nos system prompts.

**g) Limitações**

- **Modelo único e barato** (Haiku 4.5) para todos os usos, inclusive os que mais se beneficiariam de um modelo forte
- Tom **descritivo obrigatório** — nunca prescritivo (decisão de produto registrada no vault)
- `AiAnalysis.content` é `Json` sem schema versionado — mudar o formato do prompt invalida caches antigos silenciosamente **[VERIFICAR]**

---

## 2.5 Gráficos e indicadores técnicos (MM50/200/400)

**a)** Gráfico interativo com `lightweight-charts`. Períodos: `1D`, `5D`, `1M`, `3M`, `6M`, `1A`, `5A`. Sobrepõe MM50, MM200 e MM400, além de marcar os toques na MM200.

**b)** `lib/chart-data.ts` (orquestração, `ChartResult` com `price`/`mm50`/`mm200`/`mm400`/`touchesMM200`), `lib/moving-average.ts` (`computeMM`), `components/app/stock-chart.tsx`, `stock-price-chart.tsx`; PWA `components/app/stock-chart.tsx`

**c)** `GET /api/stocks/[ticker]/price-chart`, `GET /api/stocks/[ticker]/history`

**d)** `StockPriceHistory` (unique `[stockId, interval, date]`)

**e)** brapi PRO (B3/BDR, `BRAPI_HISTORY_RANGE`), Twelve Data (`fetchTwelveDataCandles`, até 5000 candles diários)

**f)** Modo intraday retorna arrays vazios de MM (`chart-data.ts:104`) — comportamento esperado, mas significa que em `1D`/`5D` **não há médias móveis**

**g)**

- `MIN_CACHE_SPAN_MS = 150 dias` — abaixo disso refaz o fetch
- Script `warm-charts.ts` existe para pré-aquecer, e `clear-chart-cache.ts` para limpar — indica que o cache de gráfico **exige operação manual**

---

## 2.6 Inteligência histórica MM200

**a)** Feature diferenciadora: detecta historicamente quando o preço **tocou a MM200** e mede o que aconteceu depois — retorno em 30/60/90 dias, taxa de acerto, melhor e pior caso.

**b)** `lib/mm200-insights.ts` (`computeMM200Insights`, tipos `DailyClose`, `MM200Touch`, `MM200Insights`), `services/technical-insights/index.ts` (fetch com dedup de 30s in-flight), `components/app/mm200-insights.tsx` (web e PWA)

**c)** `GET /api/stocks/[ticker]/technical-insights` — cache de **24h** via `TechnicalInsight.expiresAt`

**d)** `TechnicalInsight` — unique `[ticker, indicator]`, campos `touchCount`, `successRate`, `avgReturn30d/60d/90d`, `bestCase`, `worstCase`, `lastTouchDate`, `payload` (Json com os toques)

**g)**

- Depende de histórico suficiente em `StockPriceHistory` — ações com pouco histórico não geram insight
- Local tem apenas **7 registros** de `TechnicalInsight` — a cobertura é sob demanda, não pré-computada
- O model é chaveado por `ticker` (string), **não** por `stockId` — sem FK, órfãos possíveis

---

## 2.7 Watchlist

**a)** Adicionar/remover ações, nota por entrada, tabela com painel de detalhe, botão de favoritar integrado ao Screener e ao Stock Detail. Limite por plano.

**b)** Web: `app/(app)/watchlist/page.tsx`, `components/app/watchlist-table.tsx`, `watchlist-detail-panel.tsx`, `favorite-toggle-button.tsx`, `viewmodels/useWatchlist.tsx`, `services/watchlist/index.ts`
PWA: `app/(app)/watchlist/page.tsx`, `components/app/watchlist-row.tsx`
Core: `packages/core/src/watchlist.ts`

**c)** `GET/POST /api/watchlist` · `DELETE/PATCH /api/watchlist/[ticker]` — todas guardSession, com checagem de plano no POST

**d)** `Watchlist` — unique `[userId, stockId]`, índices em `userId` e `stockId`

**g)** Retorna `403 { error: "limit_reached", plan, limit }` ao exceder — o front usa isso para abrir o modal de upgrade

---

## 2.8 Alertas

**a)** 5 tipos (`PRICE_ABOVE`, `PRICE_BELOW`, `DY_ABOVE`, `PE_BELOW`, `VARIATION_ABOVE`), com threshold, ativação/desativação e entrega por email. Verificação por cron a cada 15min.

**b)** `services/alerts/index.ts`, `lib/alert-eval.ts`, `lib/email.ts`, `components/app/settings-alerts.tsx`, `create-alert-button.tsx`
PWA: `app/(app)/alertas/page.tsx`, `components/app/alert-row.tsx`, `create-alert-sheet.tsx`
Core: `packages/core/src/alerts.ts`

**c)** `GET/POST /api/alerts` · `PATCH/DELETE /api/alerts/[id]` · `POST /api/cron/check-alerts` (Bearer `CRON_SECRET`)

**d)** `Alert` + enum `AlertType` — índices `[userId, active]` e `stockId`

**e)** Resend (email), GitHub Actions (`check-alerts.yml`, `*/15 * * * *`)

**g)**

- Free limitado a **3 alertas** e só tipos de preço
- **Entrega só por email** — não há push nem in-app
- Banco local tem **0 alertas** — a feature não está exercitada localmente

---

## 2.9 Rankings

**a)** Pódio + lista de contenders por critério (DY, ROE, P/L etc.), com agregação SQL sobre `StockIndicator`. Top N limitado por plano (free 10, pro 50).

**b)** `app/(app)/rankings/page.tsx`, `components/app/rankings-podium.tsx`, `rankings-contenders.tsx`, `rankings-insights.tsx`, `services/rankings/index.ts`

**c)** `GET /api/rankings` — guardSession + plano

**d)** `Stock`, `StockIndicator`

**f)** `rankings-insights.tsx:109` — *"Você está vendo o Top {limit}. Filtros multi-critério em breve."* → **filtro multi-critério não implementado**

**g)** 🔴 **Não existe no PWA.** É a única tela do web sem equivalente mobile.

---

## 2.10 Comparador de ações

**a)** Compara N tickers lado a lado com matriz de saúde, benchmarking e painéis. Estado na URL (`?tickers=PETR4,VALE3,...`), **sem persistência**. Free compara 2, Pro compara 4.

**b)** Web: `app/(app)/compare/page.tsx` (`requireSession` + plano), `components/app/compare-benchmarking.tsx`, `compare-health-matrix.tsx`, `compare-sidebar-panels.tsx`, `compare-ticker-selector.tsx`, `services/compare/index.ts` (`parseTickersParam`, `serializeTickers`, `COMPARE_COLORS`, `colorFor`)
PWA: `app/(app)/compare/page.tsx`, `components/app/compare-add-sheet.tsx`, `compare-charts.tsx`, `compare-fundamentals.tsx`, `lib/compare.ts`

**c)** **Não tem rota própria.** Compõe N chamadas a `/api/stocks/[ticker]` (PWA usa `fetchStock` do core)

**g)** N chamadas sequenciais/paralelas por comparação — sem endpoint em lote, custo de rede multiplicado

---

## 2.11 Feed de relatórios (WhatsApp/Baileys)

**a)** Pipeline assíncrono completo, **funcionando end-to-end em produção desde 2026-06-18**:

```
Worker Baileys escuta grupos do WhatsApp
        ↓ salva PDFs em REPORTS_TEMP_PATH
POST /api/reports/process  (auth: REPORTS_PROCESS_SECRET)
        ↓ enfileira em lib/report-queue.ts
report-processor.ts: unpdf extrai texto → Claude → JSON estruturado
        ↓ classifica STOCK | SECTOR | MACRO | IGNORE
persiste em Report → recalcula ReportConsensus → emite em report-events
        ↓
GET /api/reports/stream (SSE)  — realtime é feature Pro
```

**b)** `services/whatsapp/whatsapp.service.ts` (com watchdog de reconexão), `src/scripts/start-whatsapp.ts`, `services/reports/report-processor.ts`, `lib/report-queue.ts`, `lib/report-consensus.ts`, `lib/report-events.ts`, `instrumentation.ts`
Web: `components/app/report-feed.tsx`, `report-card.tsx`, `consensus-widget.tsx`
PWA: `(app)/page.tsx` (**é a home**), `components/app/report-feed.tsx`, `report-card.tsx`, `report-detail.tsx`, `home-header.tsx`

**c)**

| Rota | Guard |
|---|---|
| `GET /api/reports` | guardSession + plano (`feedMax`) |
| `GET /api/reports/[id]` | guardSession + plano (`fullText` só Pro) |
| `GET /api/reports/consensus/[ticker]` | guardSession + plano (`feedConsensus`) |
| `POST /api/reports/process` | `REPORTS_PROCESS_SECRET` |
| `GET /api/reports/stream` | guardSession + plano (`feedRealtime`) |

**d)** `Report` (unique `sourceMessageId`, índices em `ticker`, `broker`, `reportType`, `createdAt`), `ReportConsensus` (unique `ticker`)

**e)** WhatsApp via Baileys (`6.7.23` pinado), Anthropic

**f)** Fila com **3 tentativas**, quarentena em `pending-image/` (PDF que é imagem, sem texto) e `failed/`. Backoff de 30s em rate limit.

**g) Limitações sérias**

- 🔴 **Fila e EventEmitter são in-process** (`globalThis`). Só funciona com **uma única instância** do web. Trava escala horizontal
- 🔴 **Validação de ticker frouxa** — o extrator cria entradas de menções soltas (`AZZAS2154`, `AUGO`, `PICS`) e consensos com `buy+neutral+sell = 0` que nunca renderizam
- 🔴 **SSE não funciona pelo PWA** — o proxy catch-all faz `await upstream.arrayBuffer()` (`[...path]/route.ts:37`), que aguarda o corpo inteiro. Um stream SSE nunca termina → a requisição pendura. Na prática o PWA usa polling paginado (`fetchReports`), então o realtime é **web-only**
- PDF só de imagem não é processado (sem OCR)

---

## 2.12 Renda fixa (BR + Internacional)

**a)** O domínio mais fatiado do projeto. Taxas de Brasil, EUA, Europa e internacionais; comparador com IR e câmbio; calculadora; curva de juros; histórico de séries; consultor IA.

**b)** `services/fixed-income/` — `brazil.service.ts`, `usa.service.ts`, `europe.service.ts`, `international.service.ts`, `fx.service.ts`, `history.service.ts`, `comparator.service.ts`, `ai-advisor.service.ts`, `cache.ts`, `types.ts`, `index.ts`
Web: `app/(app)/fixed-income/page.tsx`, `components/app/fixed-income/` — `ai-advisor`, `comparator-table`, `currency-input`, `curve-chart`, `fi-line-chart`, `historico`, `rate-calculator`, `rate-card`, `shared`
PWA (**novo, 2026-08-15**): `app/(app)/renda-fixa/page.tsx`, `components/app/fixed-income-screen.tsx` (5 abas em rolagem horizontal), `-overview`, `-calculator`, `-compare`, `-history`, `-ai`, `-fields`
Core: `packages/core/src/fixed-income.ts`

**c)** `GET /api/fixed-income/rates` · `compare` · `calculator` · `history` · `POST /api/fixed-income/ai-analysis` — todas guardSession

**d)** `FixedIncomeRate` (unique `[country, ticker, maturity]`), `FixedIncomeCache` (unique `cacheKey`)

**e)** BCB SGS (séries 432 Selic, 4389 CDI, 13522 IPCA 12m), FRED (`FRED_API_KEY`, treasuries e bonds internacionais), AwesomeAPI (câmbio)

**f)** Tesouro Direto **bloqueado por Cloudflare** — produtos vêm do BCB como alternativa

**g) Limitações**

- ✅ **Corrigido em 2026-08-14:** `cached()` gravava fallback de falha (`ipca12m ?? 0`) como dado bom por 1h, subestimando todo produto `IPCA+`. Agora `cached()` aceita predicado `shouldCache`
- 🔴 **`usa`, `international` e `fx` usam o mesmo `cached()` e NÃO foram auditados** para o mesmo bug
- Histórico multi-país dispara 8 chamadas ao FRED — cache frio pode retornar vazio
- Formato de erro das rotas (`{ error: "codigo", message }`) é incompatível com o `apiFetch` do core (`{ error: { code, message } }`) → o PWA sempre mostra mensagem genérica

---

## 2.13 Sistema de planos e pagamentos

Ver seção 5 completa.

---

# 3. INTEGRAÇÕES EXTERNAS

| # | Serviço | Para quê | Status | Lib/SDK | Endpoints consumidos |
|---|---|---|---|---|---|
| 1 | **brapi.dev PRO** | **Única fonte** de B3/BDR: preço, intraday, histórico 5y, fundamentos | 🟢 Funcionando | `fetch` nativo (`services/providers/brapi.ts`) | `/api/quote/{tickers}` com e sem `fundamental`/`modules` |
| 2 | **Yahoo Finance** | Fundamentos de ações US | 🟡 Parcial | `yahoo-finance2@^3.14.0` | via SDK |
| 3 | **Twelve Data** | Cotação e candles de US | 🟢 Funcionando | `fetch` (`services/providers/twelveData.ts`) | `/quote`, candles `1day` até 5000 |
| 4 | **Anthropic (Claude)** | Análise de ação, consultor de renda fixa, extração de PDF | 🟢 Funcionando | `@anthropic-ai/sdk@^0.90.0` | Messages API, modelo `claude-haiku-4-5-20251001` |
| 5 | **BCB (SGS)** | Selic, CDI, IPCA 12m e séries históricas | 🟢 Funcionando | `fetch` | `api.bcb.gov.br/dados/serie/bcdata.sgs.{id}/dados` |
| 6 | **FRED (St. Louis Fed)** | Treasuries US e bonds internacionais | 🟢 Funcionando | `fetch` | `api.stlouisfed.org/fred/series/observations` |
| 7 | **AwesomeAPI** | Câmbio (USD/EUR/GBP/JPY/CAD → BRL) | 🟢 Funcionando | `fetch` | cotação de moedas |
| 8 | **WhatsApp (Baileys)** | Recebe PDFs de relatórios dos grupos | 🟢 No ar em prod | `@whiskeysockets/baileys@6.7.23` | socket não-oficial |
| 9 | **Asaas** | Cobrança recorrente, reembolso, webhook | 🟢 Funcionando (sandbox testado) | `fetch` (`lib/asaas.ts`) | `/customers`, `/subscriptions`, `/subscriptions/{id}`, `/subscriptions/{id}/payments`, refund, cancel |
| 10 | **Resend** | OTP, reset de senha, alertas | 🟢 Funcionando | `resend@^6.12.2` | envio transacional |
| 11 | **Google OAuth** | Login social | 🟢 Funcionando | `@react-oauth/google` + `lib/google-id-token.ts` | verificação de ID token |
| 12 | **Tesouro Nacional** | Produtos do Tesouro Direto | 🔴 **Bloqueado** | — | Cloudflare barra o acesso |
| 13 | **GitHub Actions** | Cron de alertas e de sync | 🟡 sync ainda não deployado | — | `check-alerts` `*/15`, `sync-stocks` `*/30` |

## 3.1 Problemas conhecidos por integração

**brapi.dev**
- Ficou **fora do ar por ~1h em 2026-08-07** (site 200, `/api/*` em timeout). Não há circuit breaker: a rota cai em `cache-stale` e devolve preço velho
- Atraso estrutural de **15 minutos** para B3, por regra da bolsa
- Timeout configurado: 8s (`QUOTE_TIMEOUT_MS`), lote de 15 tickers (`QUOTE_CHUNK`)

**Yahoo Finance**
- **Bloqueado nos IPs de datacenter da Railway** — por isso B3/BDR migrou 100% para brapi. Continua sendo usado só para fundamentos US **[VERIFICAR]** se funciona em prod hoje

**Baileys**
- Cliente **não oficial** do WhatsApp — risco de banimento e de quebra a cada mudança do protocolo. Versão pinada em `6.7.23` justamente por isso
- Tem watchdog de reconexão (`STALE_TIMEOUT_MS`, mínimo 60s; `WATCHDOG_INTERVAL_MS` 60s)
- Limite de tamanho de PDF via `WHATSAPP_MAX_PDF_MB`

**Asaas**
- `ASAAS_ENV` decide entre sandbox e prod. Timeout de 15s
- **[VERIFICAR]** se o webhook está com URL pública configurada em produção (consta como pendência no vault)

---

# 4. BANCO DE DADOS

PostgreSQL 16. Prisma 5.22. Migrations em `apps/web/prisma/migrations/` — **14 migrations**, da `20260410040147_init` até `20260807195143_add_stock_quote`.

## 4.1 Models

### Domínio de mercado

| Model | Tabela | Campos-chave | Para quê | Índices |
|---|---|---|---|---|
| **Stock** | `stocks` | `ticker` (unique), `name`, `sector`, `industry`, `country`, `market` (B3\|BDR\|US), `currency` | Entidade central. Relaciona indicadores, cotação, watchlist, alertas, análises IA e histórico | unique `ticker` |
| **StockQuote** | `stock_quotes` | `stockId` (unique), `price`, `previousClose`, `changePercent`, `volume`, `source`, `quotedAt`, `fetchedAt` | **Cotação separada dos fundamentos** (criado 2026-08-07). Uma linha por ação, upsert. TTL 60s | unique `stock_id`, índice `fetched_at` |
| **StockIndicator** | `stock_indicators` | ~40 campos: valuation (pl, pvp, evEbitda, pEbit, lpa, vpa, beta), qualidade (roe, roa, roic, margens), saúde (divLiqEbitda, liqCorrente…), dividendos (dy, payout), operacional (ebitda, capex, receita, lucro, crescimentos), balanço (ativoTotal, patrimLiq, dívidas, caixa) | Fundamentos. Alimenta Screener, Rankings, Compare e scoring | `stock_id`, `fetched_at` |
| **StockPriceHistory** | `stock_price_history` | `stockId`, `interval`, `date`, OHLC, `volume` | Séries para gráficos, MM50/200/400 e insights MM200 | unique + índice `[stock_id, interval, date]` |
| **TechnicalInsight** | `technical_insights` | `ticker`, `indicator`, `touchCount`, `successRate`, `avgReturn30d/60d/90d`, `bestCase`, `worstCase`, `lastTouchDate`, `payload` (Json), `expiresAt` | Cache de 24h da inteligência histórica MM200 | unique + índice `[ticker, indicator]` |
| **AiAnalysis** | `ai_analyses` | `stockId`, `model`, `content` (Json), `generatedAt` | Cache de 7 dias da análise Claude por ação e modelo | unique `[stock_id, model]`, índice `stock_id` |
| **SyncLog** | `sync_logs` | `status`, `provider`, `market`, `stockCount`, `duration`, `error`, `startedAt`, `finishedAt` | Auditoria das sincronizações | — |

### Domínio de usuário

| Model | Tabela | Campos-chave | Para quê | Índices |
|---|---|---|---|---|
| **User** | `users` | `email` (unique), `name`, `password` (argon2, opcional), `emailVerified` | Conta | unique `email` |
| **Account** | `accounts` | `provider`, `providerId` | Vínculo OAuth (só Google hoje) | unique `[provider, provider_id]`, índice `user_id` |
| **EmailVerification** | `email_verifications` | `email`, `code`, `expiresAt`, `consumed` | OTP de verificação | `email`, `expires_at` |
| **UserPreferences** | `user_preferences` | `locale`, `currency`, `theme`, `experience`, `goal`, `patrimony`, `birthDate`, `markets[]`, `sectors[]`, `minVolume`, `alerts` | Onboarding + preferências. **Base para a feature de Suitability** | unique `user_id` |
| **Subscription** | `subscriptions` | `plan`, `status`, `startedAt`, `endsAt`, `canceledAt`, `providerCustomerId`, `providerSubscriptionId` | Plano efetivo e vínculo com o Asaas | unique `user_id` |
| **RefreshToken** | `refresh_tokens` | `tokenHash` (unique), `expiresAt`, `revokedAt` | Rotação de sessão | `user_id`, `expires_at` |
| **PasswordReset** | `password_resets` | `tokenHash` (unique), `expiresAt`, `consumedAt` | Recuperação de senha | `user_id`, `expires_at` |

### Domínio de produto

| Model | Tabela | Campos-chave | Para quê | Índices |
|---|---|---|---|---|
| **Watchlist** | `watchlist_entries` | `userId`, `stockId`, `note`, `addedAt` | Favoritos | unique `[user_id, stock_id]`, índices em ambos |
| **Alert** | `alerts` | `type` (enum), `threshold`, `active`, `triggeredAt`, `lastValue` | Alertas de preço/indicador | `[user_id, active]`, `stock_id` |
| **Report** | `reports` | `reportType`, `ticker`, `company`, `broker`, `recommendation`, `targetPrice`, `upside`, `analystName`, `reportDate`, `summary`, `fullText`, `strengths[]`, `risks[]`, `companiesMentioned[]`, `sourceMessageId` (unique) | Relatório de corretora extraído de PDF | `ticker`, `broker`, `report_type`, `created_at` |
| **ReportConsensus** | `report_consensus` | `ticker` (unique), `buyCount`, `neutralCount`, `sellCount`, `avgTargetPrice` | Consenso agregado por ação | unique `ticker` |
| **FixedIncomeRate** | `fixed_income_rates` | `country`, `category`, `name`, `ticker`, `maturity`, `rate`, `rateType`, `currency`, `minAmount` | Produtos de renda fixa | unique `[country, ticker, maturity]`, índices `country` e `category` |
| **FixedIncomeCache** | `fixed_income_cache` | `cacheKey` (unique), `data` (Json), `expiresAt` | Cache genérico das fontes externas | unique `cache_key` |

**Enum:** `AlertType` = `PRICE_ABOVE`, `PRICE_BELOW`, `DY_ABOVE`, `PE_BELOW`, `VARIATION_ABOVE`

## 4.2 Observações sobre o schema

- `StockIndicator` **não tem unique em `stockId`** — decisão deliberada. Antes fazia `create` a cada refresh (churn: prod chegou a 1978 linhas para 680 ações). Virou update-or-create, mas o unique não foi adicionado porque a migration teria que apagar ~1300 linhas históricas em produção
- `TechnicalInsight` e `ReportConsensus` são chaveados por **`ticker` (string), sem FK para `Stock`** — permite órfãos
- `Report.ticker` também é string livre — origem dos tickers-lixo

## 4.3 Seeds e dados iniciais

`apps/web/scripts/seed.ts` → chama `runSync({ useDefaults: true, market })`, que usa `DEFAULT_TICKERS` de `services/providers/yahooFinance.ts`. Aceita argumento de mercado (`B3`, `BDR`, `US`).

`seed-reports.ts` existe para popular o Feed. Não há seed de usuários — `create-admin.ts` cria admin manualmente.

---

# 5. SISTEMA FREE vs PRO

## 5.1 Limites (fonte única: `apps/web/src/lib/plan.ts`)

| Recurso | Free | Pro |
|---|---|---|
| Watchlist | 10 ações | Ilimitado |
| Alertas | 3 ativos | Ilimitado |
| Tipos de alerta | só `PRICE_ABOVE` e `PRICE_BELOW` | todos os 5 |
| Comparador | 2 ações | 4 ações |
| Rankings | Top 10 | Top 50 |
| Export CSV | ❌ | ✅ |
| Análise IA | ❌ | ✅ |
| Feed — relatórios visíveis | 6 | Ilimitado |
| Feed — realtime (SSE) | ❌ | ✅ |
| Feed — consenso | ❌ | ✅ |
| Feed — texto completo do relatório | ❌ | ✅ |

## 5.2 Como o gating é implementado

**Não há middleware de plano.** O padrão é explícito por rota:

```ts
const plan = await getEffectivePlan(userId)   // consulta Subscription
const limits = getPlanLimits(plan)
if (!limits.aiAnalysis) return 403
```

`getEffectivePlan` retorna `"pro"` **somente** se existir `Subscription` com `status === "active"` **e** `plan === "pro"`. Qualquer outro caso → `"free"`.

**Onde é aplicado (11 arquivos):**

| Arquivo | Recurso |
|---|---|
| `api/watchlist/route.ts` | `watchlistMax` → 403 `limit_reached` |
| `api/alerts/route.ts` | `alertsMax`, `alertTypes` → 403 `limit_reached` |
| `api/rankings/route.ts` | `rankingsTopN` |
| `api/reports/route.ts` | `feedMax` |
| `api/reports/[id]/route.ts` | `fullText` → `fullTextLocked` |
| `api/reports/consensus/[ticker]/route.ts` | `feedConsensus` → 403 `upgrade_required` |
| `api/reports/stream/route.ts` | `feedRealtime` → 403 `upgrade_required` |
| `api/stocks/[ticker]/ai-analysis/route.ts` | `aiAnalysis` |
| `(app)/compare/page.tsx` | `compareMax` (server component) |
| `(app)/stock/[ticker]/page.tsx` | análise IA (server component) |
| `(app)/reports/[id]/page.tsx` | texto completo (server component) |

**Dois contratos de erro distintos** — vale unificar:
- `403 { error: "limit_reached", plan, limit }` — consumido por `create-alert-button.tsx` e `settings-alerts.tsx`
- `403 { error: "upgrade_required", feature }` — usado no Feed

## 5.3 Preço

Definido em `api/billing/checkout/route.ts:16`:

```ts
const PRICE: Record<BillingCycle, number> = { MONTHLY: 34.9, YEARLY: 299 }
```

**R$ 34,90/mês** ou **R$ 299/ano** (equivale a R$ 24,92/mês, ~29% de desconto).

## 5.4 Pagamento (Asaas)

**Fluxo:**

1. `POST /api/billing/checkout` (guardSession) → cria `customer` no Asaas se não existir, cria `subscription` com `billingType: "UNDEFINED"` (o cliente escolhe boleto/pix/cartão), retorna `invoiceUrl`
2. `POST /api/webhooks/asaas` (auth por `ASAAS_WEBHOOK_TOKEN`) → atualiza `Subscription`
3. `POST /api/billing/sync` (guardSession) → reconcilia manualmente com o Asaas
4. `POST /api/billing/refund` (guardSession) → **garantia de 7 dias**: valida `startedAt` dentro de `GUARANTEE_DAYS = 7`, acha o pagamento `CONFIRMED`/`RECEIVED`, chama `refundPayment` e `cancelSubscription`

**Telas:** `(app)/settings/subscription`, `components/app/settings-subscription.tsx`, `landing/planos-cards.tsx`, `planos-comparison.tsx`, `planos-faq.tsx`
**PWA:** `components/app/subscription-sheet.tsx`

**Sem trial** — o modelo é garantia de reembolso de 7 dias.

---

# 6. GAPS E PROBLEMAS CONHECIDOS

## 6.1 🔴 Bloqueadores de deploy

| # | Problema | Impacto |
|---|---|---|
| 1 | Migration `20260618120144_add_report_full_text` **não registrada** no `_prisma_migrations` de produção. O SQL é `ALTER TABLE "reports" ADD COLUMN "full_text" TEXT` sem guarda, e a coluna já existe (criada na mão) | `start:railway` roda `prisma migrate deploy && next start` — o `&&` impede o `next start`. **O serviço web não sobe.** Conserto: `prisma migrate resolve --applied 20260618120144_add_report_full_text` |
| 2 | `20260807195143_add_stock_quote` está **na fila atrás** da anterior | Só aplica depois de destravar a #1 |
| 3 | **Nada commitado** — a working tree acumula duas sessões (cotação + ScoreRing de 07/08; quality gate, cron, report detail e Renda Fixa PWA de 13–15/08) | Risco de perda; produção está muito atrás do local |

## 6.2 🔴 Segurança

| # | Problema | Onde |
|---|---|---|
| 1 | **Chave da Resend commitada em texto plano** no arquivo versionado | `.env.example:21` — `RESEND_API_KEY="re_BhaTCgsS_..."`. **Rotacionar** |
| 2 | Credencial do Postgres de produção circulou em texto plano no chat em 2026-08-07 | Railway → Postgres → Variables. **Rotacionar** |
| 3 | **`/api/stocks` e `/api/stocks/[ticker]` são públicas** — dá para extrair a base inteira em páginas de 500, sem login | O middleware protege páginas, não a API |
| 4 | `/api/tickers` pública — expõe a lista completa de tickers | `app/api/tickers/route.ts` |
| 5 | `/fixed-income` e `/reports` (web) **não estão em `PROTECTED_PREFIXES`** nem usam `requireSession` | `proxy.ts:11` — a página renderiza sem login (os dados 401am, mas a casca aparece) |

## 6.3 🔴 Arquitetura

| # | Problema | Consequência |
|---|---|---|
| 1 | Fila de relatórios e `EventEmitter` são **in-process** (`globalThis`) | Feed e SSE só funcionam com **uma instância** do web. Impede escala horizontal |
| 2 | **Proxy do PWA bufferiza a resposta** (`await upstream.arrayBuffer()`) | SSE não pode passar pelo PWA. Realtime é web-only por construção |
| 3 | `cached()` da renda fixa gravava fallback de falha como dado bom | ✅ Corrigido no `brazil.service`. 🔴 **`usa`, `international` e `fx` não auditados** |
| 4 | Sem circuit breaker nos providers | Provedor fora → `cache-stale` silencioso. Produção serviu preços de 15 dias sem sinalizar |
| 5 | Helpers de apresentação **duplicados de propósito** entre web e PWA (`stock-presenter.ts`, `format.ts`) | Divergência de comportamento entre plataformas (já aconteceu com scoring) |

## 6.4 🟡 Placeholders e features incompletas

| Onde | O quê |
|---|---|
| `landing/contato-form.tsx:20` | **`handleSubmit` faz `console.log`** — o formulário de contato não envia nada |
| `auth/login-screen.tsx:159` e `register-screen.tsx:184` | "Continuar com Apple" `disabled` com badge "em breve" |
| `app/rankings-insights.tsx:109` | "Filtros multi-critério em breve" |
| `pwa/components/app/coming-soon.tsx` | Componente existe e **não é importado em lugar nenhum** — morto |
| `chart-data.ts:104` | Intraday (`1D`/`5D`) retorna MMs vazias |
| Feed | Sem OCR — PDFs só de imagem vão para quarentena `pending-image/` |

Não há nenhum `TODO`/`FIXME` no código — coerente com a regra do projeto que proíbe comentários.

## 6.5 🟡 Dívida técnica

| Item | Detalhe |
|---|---|
| **`CLAUDE.md` está errado sobre testes** | Diz *"Não há framework de testes no projeto"*, mas existem **vitest configurado em 2 workspaces** e **6 arquivos de teste**: `lib/format.test.ts`, `lib/plan.test.ts`, `lib/report-consensus.test.ts`, `services/fixed-income/comparator.service.test.ts`, `services/providers/yahooFinance.test.ts`, `packages/core/src/scoring.test.ts`. Há até Stryker (mutation testing) configurado |
| **Cobertura de teste mínima** | 6 arquivos para ~49 rotas e ~30 services |
| Dependências mortas | `next-auth`, `@auth/prisma-adapter`, `lucide-react`, `shadcn`, `xlsx` **[VERIFICAR]** |
| ~1300 linhas paradas | `stock_indicators` histórico em produção (churn estancado, limpeza pendente) |
| Tickers-lixo | `AZZAS2154`, `AUGO`, `PICS`, `CPRI` em `report_consensus`; consensos com contagem zero |
| `HANDOFF.md` da raiz | Congelado em 2026-04-10, descreve estrutura pré-monorepo. **Contradiz a realidade** |
| Cache de gráfico manual | `warm-charts.ts` e `clear-chart-cache.ts` — operação manual |
| Formato de erro inconsistente | Rotas devolvem `{ error: "string", message }`; `apiFetch` espera `{ error: { code, message } }` |
| `routes.d.ts` corrompido | O dev server escreve parcialmente e quebra o type-check. Apagar + reiniciar |

## 6.6 Quality gate (o que o projeto verifica hoje)

`pnpm gate` → `.claude/scripts/quality-gate.sh`:

1. `prisma generate`
2. `type-check` dos 4 workspaces
3. `eslint`
4. `dependency-cruiser` (3 configs)
5. auditoria do `globalEnv` do Turbo
6. checagem de scoring duplicado

**Fronteiras vigiadas** (`.dependency-cruiser*.cjs`): `pwa-nao-importa-web`, `pwa-sem-banco`, `packages-nao-importam-apps`, `components-sem-banco`, `viewmodels-sem-servidor`, `scoring-so-do-core`, `no-circular`.

**Estado atual: PASS** — 0 erros, **390 warnings**, sendo os maiores blocos: `gorila/no-comments` (150), `max-lines-per-function` (62), `tailwindcss/no-custom-classname` (41), `react-hooks/set-state-in-effect` (34), `complexity` (34).

> Os 150 warnings de `no-comments` são o código legado admitido pelo `CLAUDE.md` (`services/providers/`, `services/sync/`, `schema.prisma`).

## 6.7 Paridade entre plataformas

**Existe no web e NÃO no PWA:**

| Tela | Situação |
|---|---|
| **Rankings** | 🔴 Único gap real. A API `/api/rankings` está pronta — é wiring |
| Settings completo | O PWA tem `perfil` com sheets (`edit-name-sheet`, `preferences-sheet`, `subscription-sheet`), mas não a estrutura de abas do web |
| Landing pages | `/`, `/planos`, `/produtos`, `/contato`, `/bem-vindo` — intencionalmente web-only |
| Feed realtime (SSE) | Bloqueado pela arquitetura do proxy (ver 6.3) |

**Existe no PWA e NÃO no web:**

| Tela | Situação |
|---|---|
| **`/alertas` como tela dedicada** | No web os alertas vivem dentro de `settings` |
| **Feed como home** | No PWA o Feed é a rota raiz `(app)/page.tsx`; no web é `/reports` |
| `welcome` screen | `(auth)/welcome` |

---

# 7. MÉTRICAS E DADOS

## 7.1 Base de ações — ambiente LOCAL (2026-08-15)

| Mercado | Ações |
|---|---|
| **US** | 490 |
| **B3** | 140 |
| **BDR** | 41 |
| **Total** | **671** |

> ⚠️ **Produção tem base diferente e maior.** O vault registra ~1410 ações no total e 680 com indicadores em 2026-08-07. **[VERIFICAR]** consultando produção.

**Contagens completas (local):**

| Entidade | Registros |
|---|---|
| `stocks` | 671 |
| `stock_indicators` | 676 |
| `stock_quotes` | 10 |
| `stock_price_history` | 19.939 |
| `technical_insights` | 7 |
| `ai_analyses` | 9 |
| `reports` | 69 |
| `report_consensus` | 14 |
| `fixed_income_rates` | 27 |
| `fixed_income_cache` | 46 |
| `users` | 3 |
| `watchlist_entries` | 3 |
| `alerts` | **0** |
| `sync_logs` | 8 |

**Top setores (local):** Technology (75), Industrials (70), Financial Services (64), Healthcare (61), Consumer Cyclical (56) — nomes em inglês, refletindo o predomínio de ações US.

## 7.2 Cache — o que e por quanto tempo

| O que | TTL | Onde fica | Configurável |
|---|---|---|---|
| **Cotação** | **60s** | `StockQuote` (Postgres) | `QUOTE_TTL_SECONDS` |
| **Fundamentos** | **12h** | `StockIndicator` | `FUNDAMENTALS_TTL_HOURS` |
| Análise IA da ação | **7 dias** | `AiAnalysis` | não |
| Insights MM200 | **24h** | `TechnicalInsight.expiresAt` | não |
| Indicadores BR (Selic/CDI/IPCA) | **1h** | `FixedIncomeCache` | não |
| Treasuries US | **1h** | `FixedIncomeCache` | não |
| Bonds internacionais | **1h** | `FixedIncomeCache` | não |
| Câmbio | **5min** | `FixedIncomeCache` | não |
| Histórico de séries (renda fixa) | **6h** | `FixedIncomeCache` | não |
| Lista de tickers | **24h** | memória do processo | não |
| Dedup de insights técnicos | **30s** | memória (in-flight) | não |
| Gráfico | span mínimo de **150 dias** | `StockPriceHistory` | não |

## 7.3 Jobs agendados

| Job | Frequência | Onde | Auth | Status |
|---|---|---|---|---|
| **Check alerts** | `*/15 * * * *` | GitHub Actions → `POST /api/cron/check-alerts` | Bearer `CRON_SECRET` | 🟢 No ar |
| **Sync stocks** | `*/30 * * * *` | GitHub Actions → `POST /api/cron/sync-stocks` | Bearer `CRON_SECRET` | 🟡 **Implementado, não deployado** |
| **Quality gate** | por push/PR | GitHub Actions | — | 🟢 No ar |
| **Watchdog WhatsApp** | 60s | dentro do worker | — | 🟢 No ar |
| **Scan de PDFs órfãos** | no boot | `instrumentation.ts` | — | 🟢 |

**Estratégia do sync rotativo:** seleciona tickers com indicador mais velho que 12h via `LEFT JOIN` no `MAX(fetched_at)`, lote padrão 60 (`SYNC_BATCH_SIZE`), teto 200. Devolve `requested`, `synced`, `failed`, `durationMs`, `market`, `oldestBefore`, `staleRemaining`. Medido: **200 ações em 29,5s**.

## 7.4 Frequência real de atualização

| Dado | Como atualiza hoje |
|---|---|
| Cotação | Sob demanda, TTL de 60s |
| Fundamentos | Sob demanda (TTL 12h) **+ cron a cada 30min quando deployar** |
| Relatórios | Push do WhatsApp, tempo real |
| Renda fixa | Sob demanda, TTLs de 5min a 6h |
| Insights MM200 | Sob demanda, TTL 24h |

> 🔴 **Enquanto o cron não for deployado, o preço em produção só atualiza quando alguém abre a página de uma ação.** Foi assim que produção acumulou 15 dias de defasagem em B3/BDR e 7 semanas em US.

---

# 8. NAVEGAÇÃO E UX

## 8.1 Mapa de rotas — apps/web (:3000)

**Públicas**

| Rota | Tela |
|---|---|
| `/` | Landing (hero, features, ferramentas, institucional, CTA, ticker tape) |
| `/planos` | Planos (cards, comparação, FAQ) |
| `/produtos` | Produtos |
| `/contato` | Contato (**formulário não envia**) |
| `/bem-vindo` | Boas-vindas |
| `/login` · `/registrar` · `/registrar/perfil` · `/registrar/preferencias` | Auth e onboarding |
| `/verificar-email` · `/esqueci-senha` · `/redefinir-senha` | Auth |

**Autenticadas — `(app)`**

| Rota | Guard |
|---|---|
| `/screener` | middleware |
| `/stock` | ⚠️ nenhum |
| `/stock/[ticker]` | middleware + `requireSession` |
| `/watchlist` | middleware |
| `/compare` | middleware + `requireSession` |
| `/rankings` | middleware |
| `/settings` · `/settings/subscription` | middleware + `requireSession` |
| `/reports` | ⚠️ **nenhum** |
| `/reports/[id]` | `requireSession` |
| `/fixed-income` | ⚠️ **nenhum** |

## 8.2 Mapa de rotas — apps/pwa (:3001)

**`(auth)`** — `/welcome`, `/login`, `/registrar`, `/registrar/perfil`, `/registrar/preferencias`, `/verificar-email`, `/esqueci-senha`

**`(app)`** — protegido no `layout.tsx` client-side via `useSession` (redireciona para `/welcome` se não autenticado, mostrando `Splash` enquanto carrega)

| Rota | Tela |
|---|---|
| `/` | **Feed de relatórios (home)** |
| `/screener` | Screener com sheet de filtros |
| `/stock/[ticker]` | Detalhe da ação |
| `/watchlist` | Watchlist |
| `/compare` | Comparador |
| `/alertas` | Alertas (tela dedicada) |
| `/renda-fixa` | **Renda Fixa com 5 abas** (novo) |
| `/reports/[id]` | Detalhe do relatório (novo) |
| `/perfil` | Perfil, preferências, assinatura |

**`/api/[...path]`** — proxy catch-all para o web

## 8.3 Roteamento entre os dois apps

`apps/web/src/proxy.ts` (middleware do Next):

1. Se `PWA_URL` está setado **e** o user-agent é mobile **e** não é bot → **redirect 307 para o PWA**
2. Mantém o path quando ele está em `SHARED_PREFIXES` = `/stock`, `/screener`, `/watchlist`, `/compare`, `/alertas`. Caso contrário, joga na raiz do PWA
3. `?view=web` grava cookie `ga_force_web=1` (30 dias) e desliga o redirect
4. Protege `PROTECTED_PREFIXES` = `/screener`, `/stock`, `/watchlist`, `/compare`, `/rankings`, `/settings` checando o cookie `ga_access`
5. Matcher exclui `api`, `_next/static`, `_next/image`, `favicon.ico`, `manifest.json`, `sw.js`, `icons`, `robots.txt`, `sitemap.xml`

> ⚠️ `SHARED_PREFIXES` inclui `/alertas`, que **não existe no web** — um usuário mobile em `/alertas` vem do PWA. Mas `/rankings`, `/reports` e `/fixed-income` **não** estão em `SHARED_PREFIXES`, então mobile nesses caminhos cai na home do PWA.

## 8.4 Navegação

| | Web | PWA |
|---|---|---|
| Estrutura | `app-sidebar.tsx` + `app-header.tsx` (com busca de ticker) | `bottom-nav.tsx` (7 itens, 44px cada, 333,6px total) + `app-shell.tsx` |
| Itens | Sidebar completa | Home, Screener, **Renda Fixa**, Watchlist, Compare, Alertas, Perfil |
| Padrão de interação | Painéis laterais e modais | **Bottom sheets** (`ui/sheet.tsx`) |

## 8.5 O que é realmente compartilhado

**`@gorila/ui`** — 23 ícones (`ArrowRight`, `Bell`, `Chart`, `Check`, `Close`, `Coin`, `Compare`, `CreditCard`, `Eye`, `EyeOff`, `FileText`, `Filter`, `Globe`, `Gorila`, `Home`, `LogOut`, `Lupa`, `Menu`, `Sparkle`, `Star`, `Sync`, `User`, `Zap`) + tokens.

> ⚠️ **Duplicação de ícones:** o web tem sua **própria** pasta `components/ui/icons/` com ~30 ícones, e não usa a de `packages/ui`. São dois conjuntos paralelos, com sobreposição (`SparkleIcon`, `FileTextIcon`, `ArrowRightIcon` existem nos dois).

**`@gorila/core`** — 12 módulos: `types`, `http` (`apiFetch`, `ApiError`), `session` (`SessionProvider`, `useSession`), `auth`, `reports`, `stocks`, `scoring` (`computeScores`, `ratingFromScore` — **fonte única**, vigiada pela regra `scoring-so-do-core`), `watchlist`, `technical`, `alerts`, `account`, `fixed-income`.

**Componentes de UI NÃO são compartilhados.** Cada app tem os seus (`apps/web/src/components/ui/` vs `apps/pwa/src/components/ui/`), com estilos diferentes: o web usa hex literais (`#0f1e2b`, `#00e5a0`), o PWA usa CSS custom properties (`var(--surface)`, `var(--primary)`).

---

# Apêndice — Regras do projeto (`CLAUDE.md`)

1. 🚫 **Proibido criar comentários no código** — nenhum `//`, `/* */`, JSDoc, TODO ou FIXME. Código deve ser autoexplicativo. Legado com comentários existe em `services/providers/`, `services/sync/` e `schema.prisma` — não replicar
2. 🎨 **Ícones sempre da pasta de ícones do projeto** — `@/components/ui/icons/` no web, `@gorila/ui/icons` no PWA e packages. Sem lucide-react, sem SVG inline. Ícone novo se cria na pasta primeiro
3. Toda env nova precisa entrar no `globalEnv` do `turbo.json`, senão o cache do Turbo serve build velho
4. Verificação antes de entregar: **`pnpm gate`**
