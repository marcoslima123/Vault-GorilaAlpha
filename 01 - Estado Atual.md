# Estado Atual — GorilaAlpha

> Snapshot em **2026-05-20**. Atualizar conforme o projeto evolui.

Voltar para [[00 - Home]].

## Visão Geral em 1 minuto

Área autenticada ~40% funcional:
- **Auth completa** (custom JWT + Google OAuth + OTP por email + refresh tokens)
- **Dados de mercado completos** (1410 stocks B3/BDR/US via Yahoo Finance)
- **Layout pronto** (sidebar, header, design system)
- **UI da área logada**: 27 componentes existem, mas a maioria são esqueletos sem dados
- **Backend**: só `/api/profile` e `/api/auth/me` são autenticados; endpoints de stock são públicos
- **Sem middleware**: rotas `(app)/*` só são protegidas client-side (gap de segurança)

---

## 1. Rotas

### Públicas
- `/` — Landing (hero, ticker tape, features, CTA)
- `/login`, `/registrar`, `/registrar/perfil`, `/registrar/preferencias`
- `/verificar-email` (OTP), `/esqueci-senha`, `/redefinir-senha`
- `/bem-vindo` (welcome pós-onboarding)
- `/contato`, `/planos`, `/produtos`

### Autenticadas (grupo `(app)`)
- `/screener` — tabela de stocks com filtros fundamentalistas
- `/stock/[ticker]` — detalhe individual (redirect `/stock` → `/stock/PETR4`)
- `/watchlist` — ativos monitorados
- `/compare` — comparador de 2–4 stocks
- `/rankings` — rankings por critério (DY, ROE, etc.)
- `/settings` — perfil, assinatura, preferências, alertas

**Proteção:** ⚠️ não há `middleware.ts`. Guarda é **client-side** via `useAuth()` checando `status === "authenticated"`. Gap a fechar.

---

## 2. Autenticação

Custom JWT (não usa NextAuth).

### Arquivos-chave
- `apps/web/src/lib/auth-tokens.ts` — `issueSession`, `rotateRefreshToken`, `revokeRefreshToken`
- `apps/web/src/lib/auth-session.ts` — `getCurrentSession()` (server)
- `apps/web/src/lib/auth-cookies.ts` — `setAuthCookies()`
- `apps/web/src/lib/auth-validation.ts` — Zod schemas
- `apps/web/src/viewmodels/useAuth.tsx` — AuthContext (client)
- `apps/web/src/services/auth.ts` — HTTP client

### Strategies
- ✅ Email + senha (argon2 hash) com rate limit (9/15min por IP, 5/15min por email)
- ✅ Google OAuth (`@react-oauth/google`)
- ✅ Verificação de email via OTP (Resend)
- ✅ Reset de senha por token

### Endpoints `/api/auth/*`
- `POST /register` · `POST /login` · `POST /logout`
- `POST /verify` · `POST /resend`
- `GET /me` · `POST /refresh`
- `POST /forgot-password` · `POST /reset-password`
- `POST /google`

---

## 3. Schema do banco (`apps/web/prisma/schema.prisma`)

11 models. Todos com propósito ativo — nenhum morto.

### Auth (8 models)
- **User** — id, email, name, password (argon2), emailVerified, createdAt, updatedAt
- **Account** — OAuth social (provider + providerId)
- **EmailVerification** — OTP codes
- **RefreshToken** — tokens (userId, tokenHash, expiresAt, revokedAt)
- **PasswordReset** — tokens de reset
- **UserPreferences** — locale, currency, theme, experience, goal, patrimony, birthDate, markets[], sectors[], minVolume, alerts
- **Subscription** — userId, plan, status, startedAt, endsAt, providerCustomerId

### Stocks (3 models)
- **Stock** — ticker (unique), name, sector, industry, country, market, currency — **1410 registros**
- **StockIndicator** — 40+ campos: price, marketCap, PL, PVP, ROE, ROA, ROIC, EBITDA, margens, dividendos, dívida, etc. — fetch cacheado 15min
- **SyncLog** — logs de sync (status, provider, market, stockCount, duration, error)

### Faltando (referenciados pela UI mas sem model)
- ❌ **Watchlist** — não existe model
- ❌ **Alert** — `UserPreferences.alerts` é só uma string, não há estrutura real
- ❌ **AnalysisLog/AiAnalysis** — para o card de "análise IA" do stock detail

---

## 4. API Routes

### Autenticados
- `GET/POST /api/profile` — preferências do usuário (`getCurrentSession()` guard)
- `GET /api/auth/me`

### Públicos (deveriam ser autenticados?)
- `GET /api/stocks?market=&search=&minPl=&maxPl=&minRoe=&minDy=&minVol=&sector=&limit=&offset=`
- `GET /api/stocks/[ticker]` (cache 15min, live se expirado)
- `GET /api/tickers?market=&stats=`

### Admin (públicos sem auth — risco)
- `POST /api/sync` — sincroniza tickers (Yahoo Finance)
- `GET /api/sync` — status da última sync
- `POST /api/fix-markets`
- `POST /api/analysis` (stub)

### Docs
- `GET /api/docs` — Swagger UI
- `GET /api/docs/swagger.json` — spec OpenAPI

### Faltando
- ❌ `/api/watchlist` (CRUD)
- ❌ `/api/rankings`
- ❌ `/api/compare`
- ❌ `/api/alerts`

---

## 5. Componentes da área logada

### Layout
- `apps/web/src/app/(app)/layout.tsx` — renderiza `<AppSidebar />` + children
- `src/components/app/app-sidebar.tsx` — navegação esquerda

### 27 componentes em `src/components/app/`

| Componente | Status |
|---|---|
| AppHeader, StatsCards | esqueleto |
| ScreenerFilters, ScreenerTable | esqueleto (não consome `/api/stocks` ainda) |
| StockDetailPanel | esqueleto |
| WatchlistTable, WatchlistDetailPanel | esqueleto (sem backend) |
| CompareHealthMatrix, CompareBenchmarking, CompareTickerSelector, CompareSidebarPanels, CompareBottomPanels | esqueleto |
| RankingsPodium, RankingsContenders, RankingsInsights, RankingsSidebarCard | esqueleto |
| SettingsTabs, SettingsProfile, SettingsSubscription, SettingsPreferences, SettingsAlerts | esqueleto |
| StockHeaderCard, StockPriceChart, StockInvestmentSummary, StockScoreModules, StockAiAnalysis, StockBalanceSheet | **mock hardcoded (só PETR4)** |

### Ícones (`src/components/ui/icons/` — 33 ícones)

AlertIcon, AlertTriangleIcon, AppleIcon, ArrowRightIcon, BarChartIcon, ChartIcon, CheckIcon, ChevronDownIcon, CloseIcon, CreditCardIcon, DatabaseIcon, DocumentIcon, DownloadIcon, EmailIcon, EyeIcon, EyeOffIcon, FileTextIcon, FilterIcon, GlobeIcon, GoogleIcon, LupaIcon, MenuIcon, PlusIcon, SendIcon, SettingsIcon, ShareIcon, SparkleIcon, StarIcon, SyncIcon, TargetIcon, TrophyIcon, UserIcon, ZapIcon

---

## 6. Gaps críticos

1. **Sem proteção server-side das rotas `(app)`** — JS desabilitado bypassa
2. **Endpoints de stock públicos** — sem paywall, sem rate limit por user
3. **Stock Detail usa mock hardcoded** — só PETR4 renderiza; ignora `/api/stocks/[ticker]` que já funciona
4. **Watchlist, Compare, Rankings sem backend** — UI pronta, mas zero endpoints
5. **Alerts sem schema real** — UserPreferences.alerts é string solta
6. **`/api/sync` sem proteção** — qualquer um pode disparar sync e consumir quota do Yahoo

---

## Conclusão

Fundação sólida (auth + dados de mercado + design system), mas o **tecido conectivo está faltando**: UIs prontas que não falam com APIs, ou APIs prontas sem UI. O caminho mais eficiente é começar pelas conexões já viáveis ([[02 - Proximos Passos]]).
