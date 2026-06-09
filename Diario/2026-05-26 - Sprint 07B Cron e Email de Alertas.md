# 2026-05-26 — Sprint 07.B (Cron + Email de Alertas) concluído

← [[00 - Home]] · sessão anterior: [[Diario/2026-05-21 - Sprint 07A e 07C Alerts e Paywall]]

## Credenciais recebidas do Marcos

- **RESEND_API_KEY** = `re_TrE6...` (conta Resend free, 3000 emails/mês)
- **CRON_SECRET** = `jExWf...6Q=` (32 bytes base64)
- **ASAAS_API_KEY_SANDBOX** = `$aact_hmlg_...` (guardado pro 7.D, não usado ainda)
- **Host do cron:** GitHub Actions

⚠️ Chaves vieram no chat. `.env` é gitignored. Recomendado rotacionar a Resend antes de produção.

## O que entregou

### Env
- `.env` + `.env.example` ganharam `CRON_SECRET`, `ASAAS_API_KEY_SANDBOX`, `ASAAS_ENV`, `ASAAS_WEBHOOK_TOKEN`, `ASAAS_API_KEY_PROD`.
- `turbo.json` `globalEnv` += as 5 vars (senão o `next-server` filho do turbo não enxerga).
- ⚠️ A chave Asaas começa com `$` → docker compose emite warning de interpolação. Inofensivo no 7.B (cron não usa Asaas). Corrigir no 7.D escapando `$$` ou movendo pra secret do host.

### Lib pura
- `apps/web/src/lib/alert-eval.ts` — `evaluateAlert(type, threshold, snapshot)` retorna `{triggered, currentValue}`. Cobre os 5 tipos:
  - PRICE_ABOVE: `price >= threshold`
  - PRICE_BELOW: `price <= threshold`
  - DY_ABOVE: `dy >= threshold`
  - PE_BELOW: `pl <= threshold` (descarta pl ≤ 0)
  - VARIATION_ABOVE: `(price/low52w - 1) >= threshold` (alta desde a mínima de 52 semanas — definido assim por ser o dado disponível; documentar pro usuário)

### Email
- `lib/email.ts` += `sendAlertEmail()` + templates HTML/text. Reusa o `deliver()` lazy (sem RESEND_API_KEY ele só loga). Disclaimer no rodapé. Avisa que o alerta foi desativado após disparo.

### Endpoint
- `POST /api/cron/check-alerts` — protegido por `Authorization: Bearer ${CRON_SECRET}` (401 sem/errado).
- Busca alerts `active:true` com user + stock + indicator.
- Avalia cada um. Pros disparados: marca `active=false`, `triggeredAt`, `lastValue`, e envia email.
- **Disparo único** (desativa após disparar) — evita spam. Usuário reativa/recria se quiser.
- Retorna `{ok, checked, triggered, emailsSent, errors?, at}`.

### GitHub Actions
- `.github/workflows/check-alerts.yml` — schedule `*/15 * * * *` + `workflow_dispatch` (disparo manual). Faz `curl POST` com Bearer. Usa secrets `APP_URL` e `CRON_SECRET` do repo. Falha o job se HTTP != 200.

## Validações (local)

| Cenário | Resultado |
|---|---|
| cron sem auth | 401 ✅ |
| cron Bearer errado | 401 ✅ |
| cron Bearer certo, sem alerts | `{checked:0, triggered:0}` ✅ |
| Criar PETR4 PRICE_ABOVE 1 + rodar cron | `{checked:1, triggered:1, emailsSent:1}` ✅ |
| Alerta pós-disparo | `active=false, triggeredAt=..., lastValue=45.05` ✅ |

**Email enviado com sucesso** (emailsSent=1, sem erros). ✅ **Marcos CONFIRMOU recebimento no inbox em 2026-05-26** — pipeline validado ponta a ponta. Confirma também que `marcoslima_1212@hotmail.com` é o email dono da conta Resend (por isso a entrega passou mesmo sem domínio verificado).

## Pra ativar o cron em PRODUÇÃO (falta)

O GitHub Actions só funciona quando:
1. **Código no GitHub** (push do repo)
2. **App deployado em URL pública** (Vercel? Railway? VPS?) — localhost não é alcançável pelo GitHub
3. **Secrets no repo** (Settings → Secrets and variables → Actions):
   - `APP_URL` = URL pública do app (ex: `https://gorila-alpha.vercel.app`)
   - `CRON_SECRET` = mesmo valor do `.env`

Enquanto não tem deploy, dá pra testar manualmente via PowerShell (como fizemos) ou rodar o workflow com `workflow_dispatch`.

## Limitações conhecidas

- **Resend `onboarding@resend.dev`** só entrega pro email dono da conta Resend. Pra enviar pra qualquer usuário, precisa **verificar um domínio próprio** no Resend (DNS: SPF + DKIM).
- **VARIATION_ABOVE** usa "alta desde a mínima 52w" — não é "variação no ano". Limitação do dado disponível.
- **Sem deduplicação temporal sofisticada** — disparo único resolve, mas se o usuário quer alerta recorrente, precisa recriar. Backlog: modo "recorrente com lock".
- **GitHub Actions cron tem atraso** — pode rodar com 5-15min de atraso vs o horário agendado (limitação do GitHub, não é garantido em ponto).

## Status Sprint 07

- [x] 7.A — Alerts CRUD + UI (2026-05-21)
- [x] 7.C — UpgradeModal + /settings/subscription (2026-05-21)
- [x] **7.B — Cron + email (2026-05-26)**
- [ ] 7.D — Integração Asaas (precisa conta validada + corrigir `$` no env)
- [ ] 7.E — Garantia 7 dias (depende 7.D)

## Próximo

7.D (Asaas) quando a conta sair do KYC. A chave sandbox já está no `.env`. Vou precisar:
- Corrigir o escape do `$` no docker (usar `$$` ou env do compose direto)
- Definir URL pública pro webhook (ngrok pra testar sandbox, ou deploy)
- Texto da política de garantia (7.E)
