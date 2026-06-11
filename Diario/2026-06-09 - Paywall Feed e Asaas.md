# 2026-06-09 — Paywall do Feed + Monetização Asaas (Sprint 07.D/E)

← [[00 - Home]] · sessão anterior: [[Diario/2026-06-09 - Modulo Renda Fixa]] · sprint: [[Sprints/Sprint 07 - Alerts e Paywall]]

## Objetivo

Fechar a monetização: paywall do Feed (free vs pro) + integração Asaas (cobrança recorrente, webhook, garantia 7 dias). Resolve o 7.D/7.E que estavam pendentes.

## Frente A — Paywall do Feed

- `PLAN_LIMITS`: novos `feedMax` (free 6 / pro ∞), `feedRealtime` (free false), `feedConsensus` (free false).
- `GET /api/reports`: free recebe só os últimos 6, sem paginação, com `plan`/`locked` na resposta.
- `GET /api/reports/stream` (SSE): 403 se não-pro.
- `GET /api/reports/consensus/[ticker]`: 403 se não-pro (ConsensusWidget já se esconde no 403).
- `ReportFeed`: lê `plan`/`locked`; só conecta SSE/mostra "ao vivo" se pro; card de upgrade (abre `useUpgradeModal`) quando `locked`.
- Decisão do Marcos: Free vê limitado; Pro vê tudo + tempo real + consenso.

## Frente B — Asaas

- `lib/asaas.ts`: cliente (base por `ASAAS_ENV`, header `access_token`). createCustomer/createSubscription/getSubscriptionPayments/getSubscription/refundPayment/cancelSubscription.
- `POST /api/billing/checkout`: cria/recupera customer (exige CPF/CNPJ na 1ª vez), cria subscription (mensal R$34,90 / anual R$299, billingType UNDEFINED), retorna `invoiceUrl` (checkout hospedado Asaas — cartão/PIX/boleto, evita PCI). Marca status `pending`.
- `POST /api/webhooks/asaas`: valida `asaas-access-token` (se `ASAAS_WEBHOOK_TOKEN` setado); PAYMENT_CONFIRMED/RECEIVED → pro/active + endsAt (nextDueDate); OVERDUE → past_due; REFUNDED → free/refunded.
- `POST /api/billing/refund`: garantia 7d — reembolsa o pagamento confirmado, cancela a subscription, status refunded. (Anti-abuse persistente via `refundedAt` ficou de fora pra não precisar de migration com dev rodando — hardening futuro.)
- `SettingsSubscription` virou client: seletor mensal/anual + CPF → checkout → redireciona pro Asaas; botão "Cancelar e pedir reembolso" se pro <7d. Tirou o placeholder "em breve".

## Bug crítico do .env (resolvido)

- A `ASAAS_API_KEY_SANDBOX` estava como `"$aact_..."` **sem escape**. O Next (`@next/env`) expande `$VAR` → a chave virava vazia (401). Corrigi pra `"\$aact_..."`. Validado: chave un-escaped = 200 no `myAccount`.
- Fluxo testado no sandbox: createCustomer + createSubscription (status ACTIVE) + invoiceUrl (`https://sandbox.asaas.com/i/...`). Funciona.

## Pendências pra ligar de verdade

1. **Reiniciar o `dev:web`** pra pegar a `.env` corrigida (senão checkout dá 401).
2. **Webhook precisa de URL pública** (Asaas não alcança localhost): expor via ngrok/deploy, configurar no painel Asaas (Webhooks → `/api/webhooks/asaas`) + setar `ASAAS_WEBHOOK_TOKEN` (no .env e no painel).
3. **Rotas novas exigem rebuild da imagem Docker** (regra conhecida) — em dev funciona após restart.
4. Testar checkout como usuário **free** (o do Marcos é pro/active → "already_pro").
5. tsc + lint limpos em tudo.

## Testes realizados + /planos + cadastro (2026-06-10)

- **Testes end-to-end no sandbox** (script chamando o handler do webhook direto + curl nas rotas com dev no ar): createCustomer+subscription+invoiceUrl ✅; webhook PAYMENT_CONFIRMED → free/pending vira **pro/active** ✅; checkout pela rota HTTP autenticada (login curl→cookie→/api/billing/checkout) → 200 + invoiceUrl ✅.
- Usuário de teste `teste-pro@gorila.local` / `Teste@1234` deixado no banco (Pro) pra teste no browser. Webhook precisa de URL pública (ngrok) pra ativar sozinho; localmente simulei.
- **`/planos` atualizada**: estava com Pro R$89 + Elite R$249 fictício. Agora Free (R$0) + Pro (R$34,90/mês ou R$299/ano) reais, toggle mensal/anual, features reais. `PlanosCards` CTA acionável (logado→`/settings?plan=pro&cycle`; deslogado→`/registrar?plan=pro`). `PlanosComparison` real (Free×Pro). `SettingsSubscription` rola pro `#billing` com `?plan=pro`.
- **Escolha de plano no fim do cadastro**: bloco na welcome-screen (`/bem-vindo`) → "Conhecer o Pro" (garantia 7d) ou "Continuar no Free". (A welcome-screen já persistia as preferências do onboarding via `profileService.save`.)

## Primeiro pagamento real no sandbox + correções de retorno/sync (2026-06-10)

Marcos fez um pagamento real no sandbox (cartão, R$ 34,90, confirmado) e levantou 2 problemas:

1. **"Me levou pra fora da aplicação"** → é o comportamento padrão do **checkout hospedado** do Asaas (`sandbox.asaas.com/i/...`); cartão é digitado no ambiente PCI deles. É o recomendado pra MVP. Alternativa (futuro): **Checkout Transparente** (tokeniza cartão via JS do Asaas e a gente envia) — mais trabalho + PCI.
2. **"Não voltou pra aplicação depois de pagar"** + plano ficou `pending` (o webhook não alcança localhost, então a app nunca soube que pagou).

**Correções implementadas:**
- `createSubscription` agora aceita `callback: { successUrl, autoRedirect }`. O checkout passa `successUrl = ${origin}/settings?billing=success` (origem do request) → ao pagar, o **navegador do usuário** redireciona de volta. **Funciona em localhost** (quem redireciona é o browser, não o servidor do Asaas — diferente do webhook).
- **Novo endpoint `POST /api/billing/sync`**: consulta `getSubscriptionPayments` no Asaas; se algum pagamento é `CONFIRMED/RECEIVED/RECEIVED_IN_CASH`, marca `pro/active` (mesma lógica do webhook, com `getSubscription` p/ `endsAt`). É a **alternativa ao webhook em dev**.
- `SettingsSubscription`: ao voltar com `?billing=success` chama o sync **automaticamente** (e `window.location.replace("/settings")` se virou Pro); + botão manual **"Já paguei · atualizar status"** com mensagens de status (`notice`/`error`).

**Insight central:** webhook é a fonte da verdade em produção (precisa URL pública), mas o **`/api/billing/sync` + `successUrl/autoRedirect`** cobrem o fluxo localmente e servem de fallback robusto mesmo em prod (se o webhook atrasar/falhar). Ver também [[Backlog - Suitability (Passo 4 do Cadastro)]] (não relacionado, só índice).

**Pra destravar um pagamento preso em `pending`:** logar na conta que pagou → `/settings` → "Já paguei · atualizar status".
