# Decisão — Modelo de Monetização

**Data:** 2026-05-20 · **Status:** ✅ decidido

Voltar para [[00 - Home]].

## Contexto

Precisávamos escolher entre 3 modelos: Freemium puro, Free + Premium (paywall por feature), ou Híbrido. A decisão afeta o que cada sprint entrega para qual público e como o paywall do [[Sprints/Sprint 07 - Alerts e Paywall]] é desenhado.

## Decisão

**Híbrido:** free com produto útil sozinho + features premium-only + limites de quantidade nas features compartilhadas.

## Matriz de plano (referência única)

| Feature | Free | Pro |
|---|---|---|
| Screener | completo, sem export | completo + CSV export |
| Stock Detail | completo | completo + histórico de anos |
| Watchlist | 10 tickers | ilimitado |
| Rankings | top 10 free | top 50 + filtros avançados |
| Compare | 2 ativos | 4 ativos + radar chart |
| Alerts | 3 alertas (preço apenas) | ilimitado + DY/P/L/variação |
| Análise IA | LOCKED | ilimitada |
| CSV Export | LOCKED | OK |

**Preço definido:** R$ 34,90/mês ou **R$ 299/ano** (= R$ 24,92/mês equivalente, ~29% off no anual).

## Por quê este modelo

- **Free tem produto útil sozinho** — usuário consegue fazer análise básica sem pagar, vira recomendação orgânica.
- **Dois vetores de upgrade** — limite de quantidade (Watchlist, Alerts) e features-killer (IA, CSV, Compare 4 ativos). Cada usuário converte pelo gatilho que faz mais sentido pra ele.
- **Modelo validado no mercado** — Status Invest, Suno, TradingView pro. Tem benchmark de preço e comportamento.

## Implicações nos sprints

- [[Sprints/Sprint 04 - Watchlist]] — implementar limite de 10 tickers no free desde o início; modal de upgrade quando excedido.
- [[Sprints/Sprint 05 - Rankings]] — free vê top 10, pro vê até top 50. Filtros avançados (multi-critério) só pro.
- [[Sprints/Sprint 06 - Compare]] — free comparar até 2; tentar 3+ pede upgrade. Radar chart fica gated pro pro.
- [[Sprints/Sprint 07 - Alerts e Paywall]] — limite de 3 alertas no free, tipos só `PRICE_ABOVE/BELOW`. Demais tipos e quantidade ilimitada no pro.
- **Stock Detail (Sprint 03):** versão atual fica igual para todos; histórico/série temporal entra como add-on pro depois.
- **Análise IA:** quando entrar (backlog), já nasce como pro-only. Free nem vê o card (ou vê tarjado "Disponível no Pro").

## Decisões derivadas (consolidadas em 2026-05-20)

- ✅ **Preço:** R$ 34,90/mês ou R$ 299/ano (~29% off no anual).
- ✅ **Plano anual:** sim, com desconto. Cliente paga adiantado, churn anual é menor.
- ✅ **Provedor de pagamento:** **Asaas** como primeiro provedor — menor taxa do mercado (1,99% + R$ 0,49 cartão recorrente, 0,99% PIX). Migrar para Stripe se a base crescer e o DX começar a doer.
- ✅ **Sem trial — usar garantia de 7 dias** (paga, devolve se pedir). Evita explosão de estados em `Subscription.status` e impede abuse fácil. Mesmo efeito psicológico de "risco zero".

## Implicações da escolha do Asaas + garantia

- `Subscription.status` fica simples: `active | cancelled | past_due | refunded`.
- Webhook do Asaas atualiza status; quando `refunded`, marcar `refundedAt` no DB e desativar acesso imediatamente.
- Política de garantia: visível na `/planos` e na `/settings/subscription`. Aceitar reembolso só se solicitado em ≤ 7 dias da primeira cobrança (o anual também).
- Anti-abuse opcional: um user só pode pedir garantia uma vez (campo `refundedAt` no User ou checar histórico).

## Helper técnico esperado

Em `apps/web/src/lib/plan.ts`:

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

Função `getEffectivePlan(session)` lê `Subscription.plan` e retorna a chave correspondente. Toda checagem de limite passa por este mapa.
