# Próximos Passos — Recomendação

> Baseado no [[01 - Estado Atual]] em 2026-05-20.

## Estratégia em uma frase

> **Conectar primeiro, expandir depois.** A maior parte do valor está em ligar UIs prontas a APIs já existentes, antes de criar features novas.

## Princípios que guiaram a priorização

1. **Fechar buracos de segurança antes de polir UX** — endpoint público de admin e rotas sem guard server-side são riscos baratos de mitigar.
2. **Plug-the-wires antes de build-from-scratch** — Screener e Stock Detail já têm API funcionando; só falta wiring. ROI alto.
3. **Construir features que retêm usuário cedo** — Watchlist é o primeiro motivo real do usuário voltar ao app. Vem antes de Rankings/Compare (que são "nice to have").
4. **Paywall só faz sentido com feature paywallável** — deixar para depois que Watchlist + Alerts estiverem prontos.

## Ordem recomendada

### 🥇 Imediato (1 semana)

1. **[[Sprints/Sprint 01 - Hardening de Auth]]** — middleware Next.js, proteger `/api/sync` e `/api/fix-markets`, decidir paywall conceitual para dados de mercado. (1–2 dias)
2. **[[Sprints/Sprint 02 - Screener Funcional]]** — conectar `ScreenerFilters` + `ScreenerTable` ao `/api/stocks`. URL state, paginação, debounce. (2–3 dias)
3. **[[Sprints/Sprint 03 - Stock Detail Real]]** — remover mock PETR4, consumir `/api/stocks/[ticker]`. Renderizar todos os score modules a partir do `StockIndicator`. (2–3 dias)

### 🥈 Curto prazo (2–3 semanas)

4. **[[Sprints/Sprint 04 - Watchlist]]** — model `Watchlist` no Prisma, migração, CRUD `/api/watchlist`, ação "favoritar" no Screener e Stock Detail. (3–4 dias)
5. **[[Sprints/Sprint 05 - Rankings]]** — `GET /api/rankings?criteria=&market=&limit=` com agregação SQL no `StockIndicator`. Conectar `RankingsPodium` + `RankingsContenders`. (2 dias)
6. **[[Sprints/Sprint 06 - Compare]]** — sem persistência (URL state). Compor N chamadas a `/api/stocks/[ticker]`. Renderizar matrix/benchmark. (2 dias)

### 🥉 Médio prazo (1 mês+)

7. **[[Sprints/Sprint 07 - Alerts e Paywall]]** — model `Alert`, worker (Vercel Cron ou Inngest), entrega via Resend, integração com `SettingsAlerts`. Depois ativar gates de `Subscription.plan` nos endpoints. (5–7 dias)

## Por que **não** começar pelos outros caminhos

- **"Reescrever auth para NextAuth"** — auth atual funciona, está testada e tem features que NextAuth pediria horas pra replicar (rate limit por email, OTP via Resend). Custo > benefício hoje.
- **"Construir Compare antes de Watchlist"** — usuário entra no Compare uma vez por semana; entra na Watchlist toda sessão. Retenção > exploração.
- **"Alerts primeiro"** — depende de scheduler/cron e quota de email. Mais infra, menos resultado visível por semana.
- **"Refatorar UI inteira"** — design system está OK. Refator é caro e não desbloqueia nada.

## Métricas para considerar "pronto"

| Sprint | Critério de pronto |
|---|---|
| 01 — Hardening | Usuário deslogado redireciona em `/screener` no SSR. `/api/sync` retorna 401 sem auth admin. |
| 02 — Screener | Filtros mudam URL, URL hidrata filtros, paginação funciona, < 200ms p50 com 1410 stocks. |
| 03 — Stock Detail | Qualquer ticker dos 1410 renderiza sem mock. Sem `PETR4` hardcoded no código. |
| 04 — Watchlist | Usuário adiciona/remove ticker e persiste entre sessões. Star aparece colorido no Screener. |
| 05 — Rankings | 10 critérios (DY, ROE, P/L, etc.) renderizam top 10. Resposta < 100ms. |
| 06 — Compare | Comparar 4 tickers via URL `?tickers=PETR4,VALE3,ITUB4,BBDC4`. |
| 07 — Alerts | Alerta dispara email quando preço cruza threshold. Free plan limitado a 3 alertas. |

## Riscos e dependências

- **Quota Yahoo Finance** — `/api/sync` é a única fonte. Se cair, todo o app fica obsoleto. Considerar cache mais agressivo + fallback (Brapi, AlphaVantage).
- **Vercel Cron grátis** tem limites. Avaliar Inngest ou um worker dedicado se alerts crescer.
- **GDPR/LGPD** — `User.password` hash está OK (argon2), mas falta política de retenção de logs (`SyncLog`, `RefreshToken` expirados).

## Open questions para validar com você

- ✅ ~~Modelo de monetização~~ → **Híbrido** com R$ 34,90/mês ou R$ 299/ano, Asaas, garantia 7d. Detalhes em [[Decisoes/2026-05-20 - Modelo de Monetizacao]].
- ✅ ~~Mercados~~ → **B3 + BDR + US** desde o início. Detalhes em [[Decisoes/2026-05-20 - Mercados e IA]].
- ✅ ~~IA heurística vs Anthropic~~ → **Híbrido** (heurística free, Claude pro) + tom **descritivo**. Detalhes em [[Decisoes/2026-05-20 - Mercados e IA]].
