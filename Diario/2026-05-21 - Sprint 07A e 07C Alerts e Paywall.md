# 2026-05-21 — Sprint 07.A + 07.C (Alerts + UpgradeModal) concluídos

← [[00 - Home]] · sessão anterior: [[Diario/2026-05-21 - Sprint 3.5 Grafico Historico concluido]]

## Escopo desta sessão

Atacamos só **7.A (Alerts CRUD + UI)** e **7.C (UpgradeModal + /settings/subscription mockada)** porque não precisam de dependências externas (Asaas, RESEND_API_KEY, CRON_SECRET).

Backlog do Sprint 07:
- **7.B** Cron de alerts + email (depende Resend + CRON_SECRET + decisão de host)
- **7.D** Integração Asaas (depende conta + API key)
- **7.E** Política de garantia 7d (depende 7.D)

## O que entregou

### Banco
- Model `Alert` em `apps/web/prisma/schema.prisma`: `userId, stockId, type, threshold, active, triggeredAt, lastValue, createdAt, updatedAt`.
- Enum `AlertType`: `PRICE_ABOVE | PRICE_BELOW | DY_ABOVE | PE_BELOW | VARIATION_ABOVE`.
- Relations adicionadas em `User.alerts` e `Stock.alerts`.
- Migration `20260521131613_add_alerts`.

### Service + tipos
- `apps/web/src/services/alerts/index.ts` — `fetchAlerts/createAlert/deleteAlert/toggleAlert`, `ALERT_TYPES` config (cada tipo com `label/unit/hint/freeAllowed`), `AlertError` com `code/status`.

### Endpoints
| Endpoint | Comportamento |
|---|---|
| `GET /api/alerts` | Lista alerts do user com Stock+Indicator. Retorna `{plan, limit, count, data}`. |
| `POST /api/alerts` | Body `{ticker, type, threshold}`. Valida tipo contra `PLAN_LIMITS.alertTypes` (free só `PRICE_ABOVE/BELOW`) → 403 `type_locked`. Conta alerts atuais vs `alertsMax` → 403 `limit_reached`. |
| `PATCH /api/alerts/[id]` | Toggle `active`. 404 se não for do user. |
| `DELETE /api/alerts/[id]` | Remove. 404 se não for do user. |
| Todos | `guardSession()`. |

### Component `UpgradeModal` reutilizável
- `apps/web/src/components/ui/upgrade-modal.tsx` — `<UpgradeModalProvider>` + `useUpgradeModal()` hook.
- 9 `FeatureKey` configurados com título/descrição específicos (watchlist, compare-count, compare-radar, rankings-top50, alerts-count, alerts-type, ai-analysis, csv-export, generic).
- Lista 6 benefícios Pro + caixa de preço (R$ 34,90/mês ou R$ 299/ano · –29%) + botão "Fazer upgrade" → `/settings/subscription`.
- Provider montado em `(app)/layout.tsx` envolvendo `WatchlistProvider`.

### UI atualizada
| Componente | Mudança |
|---|---|
| `SettingsAlerts` | Reescrito do zero. Lista alerts reais, form expansível pra criar (ticker autocomplete, type select com flag Pro, threshold), toggles funcionais, botão remover. Contador "X de 3 · plano FREE". |
| `CreateAlertButton` | **Novo** componente — popover compacto no `/stock/[ticker]` que cria alerta com o ticker pré-preenchido. Validação de tipo Pro abre o `UpgradeModal`. |
| `SettingsSubscription` | Refatorado. Aceita `plan/startedAt/endsAt` via props. Tabela comparativa Free vs Pro com 7 linhas (Watchlist, Compare, Rankings, Alertas, IA, CSV, Garantia). Banner "Asaas · INTEGRAÇÃO EM BREVE". Botão "Fazer upgrade" disabled (sem Asaas ainda). |
| `/settings/page.tsx` | Virou server component que busca plano do user e passa pro `SettingsSubscription`. |
| `/settings/subscription/page.tsx` | **Nova sub-rota** focada apenas no card de plano. Breadcrumb para `/settings`. |
| `ScreenerTable` (Watchlist toggle) | `window.alert()` substituído por `upgradeModal.open({ feature: "watchlist" })` quando bate no limite. |
| `CompareTickerSelector` | Mesma substituição: `compare-count` modal quando tenta passar do limite no plano free. |

### Botão Criar Alerta no stock detail
- `/stock/[ticker]/page.tsx` virou async (era), agora também chama `requireSession()` + `getEffectivePlan()` pra passar `plan` ao botão.
- Botão fica entre `FavoriteToggleButton` e `RefreshStockButton`.

## Validações

| Cenário | Resultado |
|---|---|
| `GET /api/alerts` vazio | `{plan:free, limit:3, count:0}` ✅ |
| `POST PRICE_ABOVE PETR4 40` | 200 ✅ |
| `POST DY_ABOVE` (free) | 403 `type_locked` ✅ |
| 3 alerts | aceita ✅ |
| 4º alert | 403 `limit_reached` ✅ |
| `PATCH active=false` | 200 ✅ |
| `DELETE` | 200 ✅ |
| `/settings/subscription` | 200, tabela comparativa renderiza ✅ |

## Achados

- **`docker --force-recreate` + Turbopack cache**: depois do recreate, o Next.js retornou 404 em tudo (até em rotas que existiam antes). Resolveu com `docker restart` simples. Provavelmente Turbopack precisa reaquecer.
- O modal `<UpgradeModalProvider>` precisa estar no client component layout. Como o `(app)/layout.tsx` é server component, mas os filhos imports do hook (`useUpgradeModal`) funcionam porque o provider em si é um client component declarado com `"use client"`.

## Backlog do Sprint 07

- [ ] **7.B Cron + email**: `POST /api/cron/check-alerts` (Bearer secret), workflow Vercel Cron, template de email via Resend. Pressupõe `RESEND_API_KEY` real e `CRON_SECRET` no env.
- [ ] **7.D Asaas**: customer + subscription recorrente + webhook em `POST /api/webhooks/asaas`. Pressupõe conta Asaas (sandbox + prod).
- [ ] **7.E Garantia 7d**: botão "Cancelar e reembolsar" só nos primeiros 7d, marca `Subscription.refundedAt`, revoga acesso. Anti-abuse: bloqueio se já pediu garantia antes.

## Estado do user

Continua em `plan=free` (mexi por último em [[Diario/2026-05-21 - Sprint 06 Compare concluido]]). Pra testar interface Pro:

```sql
UPDATE subscriptions SET plan='pro' WHERE user_id IN (
  SELECT id FROM users WHERE email='marcoslima_1212@hotmail.com'
);
```

## Próximo

Quando rolar tempo de configurar:
- conta Asaas → atacamos 7.D
- ou pedir `RESEND_API_KEY` + decidir host (Vercel?) → atacamos 7.B

Ou seguir polish: melhorar a UI das outras telas (PWA app vazio, SettingsProfile/SettingsPreferences que ainda são mock), schema de histórico (Sprint 3.5+) etc.
