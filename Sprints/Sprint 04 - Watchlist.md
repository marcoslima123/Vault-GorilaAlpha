# Sprint 04 — Watchlist

**Esforço:** 3–4 dias · **Status:** ✅ concluído em 2026-05-21

← [[Sprints/Sprint 03 - Stock Detail Real]] · próximo: [[Sprints/Sprint 05 - Rankings]]

## Objetivo

Primeira feature que cria razão real para o usuário voltar: marcar ativos favoritos e ver lista persistente.

## Entregáveis

### Banco
- [ ] Model `Watchlist` em `apps/web/prisma/schema.prisma`:
  ```
  model Watchlist {
    id        String   @id @default(cuid())
    userId    String
    stockId   String
    note      String?
    addedAt   DateTime @default(now())
    user      User     @relation(...)
    stock     Stock    @relation(...)
    @@unique([userId, stockId])
    @@index([userId])
  }
  ```
- [ ] Adicionar `watchlist Watchlist[]` em `User`.
- [ ] Migração: `pnpm --filter @gorila/web prisma migrate dev --name add-watchlist`.

### API
- [ ] `GET /api/watchlist` — retorna watchlist do user logado (com join no Stock + StockIndicator).
- [ ] `POST /api/watchlist` — body `{ ticker, note? }`, idempotente.
- [ ] `DELETE /api/watchlist/[ticker]`.
- [ ] `PATCH /api/watchlist/[ticker]` — atualizar nota.
- [ ] Todos com `getCurrentSession()` guard.

### UI
- [ ] `WatchlistTable` consome `/api/watchlist` (server component + cache invalidate on mutation).
- [ ] Botão "Estrela" no `ScreenerTable` e `StockHeaderCard`:
  - cinza = não na lista, verde/dourado = na lista
  - clique faz POST/DELETE, optimistic update
- [ ] `WatchlistDetailPanel` mostra detalhe + nota editável.
- [ ] Sort por: adicionado mais recente, maior variação dia, maior DY.
- [ ] Empty state com CTA "ir para Screener".

## Critério de pronto

- Adicionar PETR4 → reload → continua na lista.
- Remover funciona com 1 clique e desfaz visualmente antes do server confirmar.
- Free plan: **10 tickers** (decidido em [[Decisoes/2026-05-20 - Modelo de Monetizacao]]). 11º clique abre `<UpgradeModal />`.
- Pro plan: ilimitado.

## Notas técnicas

- Usar Server Actions ou Route Handlers? Recomendo Server Actions (Next 16, integra bem com `revalidatePath`).
- Limite por plano via `PLAN_LIMITS.{free,pro}.watchlistMax` (helper a criar em `apps/web/src/lib/plan.ts` — definido na decisão de monetização). Checar contagem antes do INSERT.

## Riscos

- Optimistic update + falha de rede precisa de rollback. Usar `useOptimistic` do React 19.
