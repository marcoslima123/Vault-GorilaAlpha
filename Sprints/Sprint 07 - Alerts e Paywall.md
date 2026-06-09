# Sprint 07 — Alerts e Paywall

**Esforço:** 5–7 dias · **Status:** 🟡 parcial (7.A + 7.C concluídos em 2026-05-21 · 7.B/7.D/7.E pendentes)

← [[Sprints/Sprint 06 - Compare]]

## Objetivo

Última peça do MVP. Alerts é a feature mais retenedora (notificação ativa) e o gatilho natural para monetização.

## Entregáveis

### Banco
- [ ] Model `Alert`:
  ```
  model Alert {
    id          String   @id @default(cuid())
    userId      String
    stockId     String
    type        AlertType
    threshold   Float
    triggeredAt DateTime?
    active      Boolean  @default(true)
    createdAt   DateTime @default(now())
    user        User     @relation(...)
    stock       Stock    @relation(...)
    @@index([userId, active])
  }

  enum AlertType {
    PRICE_ABOVE
    PRICE_BELOW
    DY_ABOVE
    PE_BELOW
    VARIATION_ABOVE
  }
  ```
- [ ] Migração.

### Worker / Scheduler
- [ ] Decidir: Vercel Cron, Inngest, ou job próprio?
  - Vercel Cron: grátis até 2 jobs/dia no plano hobby, melhor no Pro. Simples.
  - Inngest: mais robusto pra retries/dead-letter, learning curve maior.
  - **Recomendado:** começar com Vercel Cron (1 job a cada 15min) por simplicidade.
- [ ] Endpoint `POST /api/cron/check-alerts` protegido por header `Authorization: Bearer ${CRON_SECRET}`.
- [ ] Lógica:
  1. Buscar `Alert.where({ active: true })`.
  2. Agrupar por stock para minimizar fetch.
  3. Comparar com `StockIndicator` (já cacheado 15min — bate certinho com a cadência).
  4. Disparar `Resend.emails.send(...)` para alerts cruzados.
  5. Setar `triggeredAt`, opcionalmente desativar (alerta de "1 disparo" vs recorrente).

### API
- [ ] `GET/POST/PATCH/DELETE /api/alerts` — CRUD.
- [ ] Limites do plano híbrido (de [[Decisoes/2026-05-20 - Modelo de Monetizacao]]):
  - Free: **3 alerts** ativos, tipos `PRICE_ABOVE` e `PRICE_BELOW` apenas.
  - Pro: **ilimitado**, todos os tipos (`PRICE_ABOVE/BELOW`, `DY_ABOVE`, `PE_BELOW`, `VARIATION_ABOVE`).
- [ ] `POST` valida tipo+contagem contra `PLAN_LIMITS` (de `apps/web/src/lib/plan.ts`) antes do INSERT.

### UI
- [ ] `SettingsAlerts` consome `/api/alerts`:
  - Lista de alerts ativos.
  - Adicionar novo (escolher ticker + tipo + threshold).
  - Toggle active.
  - Ver histórico de disparos (último `triggeredAt`).
- [ ] Botão "Criar alerta" no `StockHeaderCard` pré-preenche o stock.

### Paywall (sobre tudo o que existe)
- [ ] Criar `apps/web/src/lib/plan.ts` com mapa `PLAN_LIMITS` (definido em [[Decisoes/2026-05-20 - Modelo de Monetizacao]]):

```ts
export const PLAN_LIMITS = {
  free: {
    watchlistMax: 10,
    alertsMax: 3,
    alertTypes: ["PRICE_ABOVE", "PRICE_BELOW"],
    compareMax: 2,
    rankingsTopN: 10,
    csvExport: false,
    aiAnalysis: false,
  },
  pro: {
    watchlistMax: Infinity,
    alertsMax: Infinity,
    alertTypes: "*",
    compareMax: 4,
    rankingsTopN: 50,
    csvExport: true,
    aiAnalysis: true,
  },
} as const;
```

- [ ] Helper `getEffectivePlan(session)` em `lib/auth-session.ts` lê `Subscription.plan` e retorna `'free' | 'pro'`.
- [ ] Componente `<UpgradeModal />` reutilizável, recebe `feature` para customizar copy.
- [ ] Página `/settings/subscription` com botão "Fazer upgrade" — preços: **R$ 34,90/mês ou R$ 299/ano**.
- [ ] Integração com **Asaas** (decidido em [[Decisoes/2026-05-20 - Modelo de Monetizacao]]):
  - Criar customer no Asaas no primeiro upgrade.
  - Subscription com cobrança recorrente (cartão ou PIX).
  - Webhook em `POST /api/webhooks/asaas` para sincronizar `Subscription.status`.
  - Salvar `Subscription.providerCustomerId` = Asaas customer ID.
- [ ] **Política de garantia de 7 dias** (em vez de trial):
  - Botão "Cancelar e pedir reembolso" na `/settings/subscription` visível só nos primeiros 7 dias.
  - Backend chama Asaas refund API + marca `Subscription.status = 'refunded'` + `Subscription.refundedAt = now()`.
  - Acesso pro revogado imediatamente.
  - Anti-abuse: user com `refundedAt` não-nulo não pode pedir garantia de novo num plano novo.
- [ ] Estados de `Subscription.status`: `active | cancelled | past_due | refunded` (sem `trialing`).

## Critério de pronto

- Criar alerta de PETR4 acima de R$ 40 → preço cruza → email chega em < 30 min.
- Free user tentando criar 4º alerta vê modal de upgrade.
- Webhook de checkout (Stripe) atualiza `Subscription.plan` automaticamente.

## Notas técnicas

- `Resend` já está em deps. Template de email reusa o que existe em verificação (mesma estética).
- Considerar Inngest se a base crescer rápido — `bun.sh` de eventos.

## Riscos

- Disparar email duas vezes pelo mesmo cruzamento: implementar lock via `triggeredAt` (não notifica de novo enquanto preço continua "acima").
- Quota Resend: free plan = 3000 emails/mês. Monitorar.
- Stripe webhook = complexidade. Considerar Lemon Squeezy se brasileiro for OK com pagamento internacional.

## Dependências

- ✅ Decisão de produto: free vs paid limits — em [[Decisoes/2026-05-20 - Modelo de Monetizacao]].
- [ ] Conta criada no **Asaas** (sandbox + produção). API key salva em env.
- [ ] `CRON_SECRET` configurado no env.
- [ ] Política de garantia escrita pra colocar em `/planos` e `/settings/subscription`.
