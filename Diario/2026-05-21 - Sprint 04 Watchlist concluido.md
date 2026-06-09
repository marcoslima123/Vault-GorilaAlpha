# 2026-05-21 — Sprint 04 (Watchlist) concluído

← [[00 - Home]] · sessão anterior: [[Diario/2026-05-21 - Botao atualizar dados]]

## O que entregou

### Banco
- **Model `Watchlist`** em `apps/web/prisma/schema.prisma` com `userId`, `stockId`, `note?`, `addedAt`, `@@unique([userId, stockId])` + índices.
- Adicionado relation `watchlist Watchlist[]` em `User` e `watchlistEntries Watchlist[]` em `Stock`.
- Migration `20260521031503_add_watchlist` aplicada via `prisma migrate dev`.

### Lib novo
- `apps/web/src/lib/plan.ts` — define `PLAN_LIMITS` (free/pro) conforme [[Decisoes/2026-05-20 - Modelo de Monetizacao]]. Função `getEffectivePlan(userId)` que lê `Subscription.plan` e retorna `"free" | "pro"` (default `free` se sem subscription ou não ativa).

### Endpoints
| Endpoint | Comportamento |
|---|---|
| `GET /api/watchlist` | Lista entries do user com Stock+Indicator. Retorna `{plan, limit, count, data}`. Idempotente. |
| `POST /api/watchlist` | Body `{ticker, note?}`. Idempotente — retorna `alreadyExists:true` se já estiver. Checa `PLAN_LIMITS.watchlistMax` ANTES de criar. 403 com `limit_reached` se excede. 404 se ticker inválido. |
| `DELETE /api/watchlist/[ticker]` | Remove. Retorna `{removed}`. |
| `PATCH /api/watchlist/[ticker]` | Atualiza nota (slice 280 chars). |
| Todos | `guardSession()`. 401 sem cookie. |

### Estado client
- `apps/web/src/services/watchlist/index.ts` — fetchWatchlist, addToWatchlist, removeFromWatchlist, `WatchlistError` com `code/status`.
- `apps/web/src/viewmodels/useWatchlist.tsx` — Context API com `Set<string>` de tickers favoritos. Optimistic update no `add`/`remove` (UI atualiza imediatamente, reverte em caso de erro). Trata 401 silenciosamente (não loga, só zera estado).
- Provider montado em `(app)/layout.tsx` → toda rota autenticada tem acesso.

### UI
- **Estrela funcional no Screener**: cinza quando não na lista, dourada quando está. Click toggle via hook. `stopPropagation` pra não navegar.
- **`FavoriteToggleButton`** no `/stock/[ticker]` ao lado do botão Atualizar. Texto muda: "Adicionar à watchlist" ↔ "Na watchlist". Cor laranja quando ativo.
- **`/watchlist` reescrito**: lista real do user, contador "X de 10 · plano FREE", empty state com CTA pro Screener, botão remover com 1 clique.
- Removidos componentes mock `WatchlistTable` e `WatchlistDetailPanel` da página (continuam exportados mas sem uso).

## Validações via API

| Cenário | Resultado |
|---|---|
| GET vazio | `{plan:"free", limit:10, count:0, data:[]}` ✅ |
| POST PETR4 | 200, retorna `id` ✅ |
| POST PETR4 duplicado | 200 com `alreadyExists:true` ✅ |
| DELETE PETR4 | 200, `{removed:1}` ✅ |
| Adicionar até 10 | 200 cada ✅ |
| 11º ticker | 403 `{error:"limit_reached", message:"Limite de 10 ativos no plano free. Faça upgrade…"}` ✅ |
| Ticker inexistente | 404 ✅ |
| Sem auth | 401 ✅ |

## Achados e limitações

- **Modal de upgrade ainda placeholder** — hoje o erro do limite cai num `window.alert()` simples no Screener. Sprint 07 implementa `<UpgradeModal />` adequado.
- **`useWatchlist` carrega a watchlist toda na primeira renderização** — para 10 tickers tá ok, mas se virar 1000 (pro), vai pesar. Backlog: paginar ou usar Set lazy.
- **Botão estrela está sempre visível** mesmo deslogado — porque o contexto retorna `isFavorite()=false` quando 401. Tecnicamente clicar logado vs deslogado se comporta diferente (deslogado: erro silencioso). A rota `/screener` já é protegida pelo middleware, então usuário deslogado nem chega lá.

## Próximo

[[Sprints/Sprint 05 - Rankings]] ou Sprint 3.5 (gráfico histórico). Faltam 3 sprints até MVP (Rankings + Compare + Alerts/Paywall).
