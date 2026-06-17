# Deploy — Railway (Produção)

> Decidido em 2026-06-10. Host escolhido: **Railway** (container always-on → real-time/SSE funciona igual ao local, sem reescrever código). Alternativa mais barata era Fly.io/VPS; Vercel ficou de fora por ser serverless (quebraria o SSE e a ingestão do Feed).

---

## 🟢 HANDOFF — Estado atual (2026-06-17)

**Web + PWA no ar.** Repo `marcoslima123/gorilaAlpha-web`, branch **`dev`**. Detalhes da sessão: [[Diario/2026-06-17 - PWA em producao, brapi PRO e incidente de disco]].

### Serviços na Railway (agora 3)
1. **Postgres** — ⚠️ volume era **500 MB** e **encheu** (cache de cotações 5y); resize feito pra **20 GB** (Live resize, cobra só por uso). Plano **Pro**.
2. **web** (Next.js) — backend + desktop + SSE. `proxy.ts` detecta **mobile (UA) → 307 pra PWA** (bots ficam no web; escape `?view=web`).
3. **gorila-mobile** (PWA) — app mobile. **Config-as-code = `railway.pwa.json`** (usa `Dockerfile.pwa`), Root Directory vazio. Proxia `/api` pro web via **route handler runtime** (`apps/pwa/src/app/api/[...path]/route.ts`, lê `WEB_INTERNAL_URL`). Var `WEB_INTERNAL_URL` = URL pública do web. No web: `PWA_URL` = URL pública do PWA.

### Dados de mercado — brapi PRO (atualizado 2026-06-17)
- **brapi PRO assinado** (~R$117/mês). Web service: `BRAPI_TOKEN` + `BRAPI_HISTORY_RANGE=5y`. ⚠️ **Token PRO atual: `rpeGNpgbvm9Ugax8bDPedT`** — o antigo (`tMihVTBkEq...`) estava INVÁLIDO/INATIVO e causava 502, Balanço vazio e gráfico de 3 meses. **As DUAS variáveis precisam estar no serviço web.**
- **B3/BDR = brapi PRO é a ÚNICA fonte** (preço, intraday, histórico 5y, fundamentos). Yahoo foi **removido** desse caminho (só fazia barulho).
- **US**: gráfico via Twelve Data; **fundamentos via Yahoo** (brapi PRO NÃO dá fundamentos de US — só preço/gráfico). Yahoo é bloqueado na Railway → fundamentos US só sincronizam rodando `seed.ts US` do PC.
- Detalhes completos da sessão: [[Diario/2026-06-17 - brapi PRO 100% (token morto, graficos, MM200, Balanco)]].
- **Não precisa mais** do warm-charts-do-PC pra B3 (brapi PRO funciona no IP da Railway). Sync dos fundamentos: 647 ações com P/L. O token free FOI DESATIVADO ao assinar o PRO.
- ⚠️ Cache `stock_price_history` a 5y é pesado (~310MB cheio); o churn (delete+reinsert por refresh) é ineficiente — de olho no disco.

### Feed (WhatsApp) — deploy pronto, falta setup
- `Dockerfile.whatsapp` + `railway.whatsapp.json` commitados. Criar serviço **`gorila-whatsapp`** (config `railway.whatsapp.json`, **volume em `/data`**, `WHATSAPP_SESSION_PATH=/data/whatsapp-session`, `NEXT_PUBLIC_APP_URL`=web, `REPORTS_PROCESS_SECRET`). No web: `ANTHROPIC_API_KEY` + `REPORTS_PROCESS_SECRET`. 1º deploy: QR nos logs → escanear com **número dedicado** (Baileys = risco de ban). Feed atual populado com **seed manual** (10 relatórios).

### Atenção ao deployar
- Cada commit redeploya os serviços do mesmo repo. **Mudança no `apps/pwa` → confirmar no serviço `gorila-mobile`** (não no web). Ex: o fix do gráfico (`7edc156`) é PWA.

---

## HANDOFF anterior (2026-06-11) — histórico

**O APP ESTÁ NO AR e funcional em produção na Railway.** Repo: `marcoslima123/gorilaAlpha-web`, branch **`dev`** (é o default), último commit **`28bac920`**.

### Funcionando em produção ✅
- Deploy (Dockerfile na raiz → NIXPACKS não, **Dockerfile** single-stage), auth completo (login, **refresh proativo 50min + 30d**, logout, botão Sair).
- Screener, Rankings, Stock detail, Compare, Watchlist, Renda Fixa.
- **Dados**: 671 ações (US 490, B3 146, BDR 35) — seed rodado do PC.
- **Gráficos**: US ao vivo (Twelve Data), B3/BDR (~143 com cache full / ~38 restantes em brapi 3mo). Sem 502.
- **Billing/Asaas** + webhook + paywall + garantia 7d.
- **Cron de alertas**: GitHub Actions (`.github/workflows/check-alerts.yml`, a cada 15min) → `POST /api/cron/check-alerts`. Secrets `APP_URL` + `CRON_SECRET` no GitHub. Testado (HTTP 200). Alertas chegam por **e-mail**.
- Fixes recentes: toggle de alerta (flex), redirect pós-login (`from`→`next`), dedup technical-insights, merge cache B3.

### Pendências / próximos passos (na ordem)
1. **Worker do WhatsApp** — rodar Baileys fora da Railway (PC/VPS) apontando pra prod (`NEXT_PUBLIC_APP_URL`=domínio Railway, `REPORTS_PROCESS_SECRET` igual). Handoff já desacoplado (envia bytes).
2. **Sync agendado** no mesmo worker/host — rodar `seed` + `scripts/warm-charts.ts` periodicamente (Yahoo funciona em IP residencial) p/ manter fundamentos + B3 frescos e **completar os ~38 B3 que faltam** (VALE3, SUZB3, RENT3, SBSP3...).
3. **(Opcional) Resend domain** — verificar domínio no Resend p/ e-mails (verificação + alertas) chegarem a **qualquer** usuário (hoje só `marcoslima_1212@hotmail.com`).
4. **(Opcional) Magic-link** — login de 1 clique pelo link do e-mail de alerta.

### Pra continuar em OUTRA MÁQUINA
- `git clone` o repo, `git checkout dev`, `git pull`.
- **Node 22** (o projeto exige; `engines >=22`).
- `pnpm install`.
- **`.env` NÃO está no git** (segredos). Recriar local OU pegar valores no **Railway → Variables**. Pro `warm-charts`/`seed` contra prod, precisa: `DATABASE_URL` = a **`DATABASE_PUBLIC_URL`** (pegar em Railway → Postgres → Variables; host `*.proxy.rlwy.net`), `BRAPI_TOKEN`, `TWELVEDATA_API_KEY`.
- Rodar warm/seed: `$env:DATABASE_URL="<public_url>"; pnpm exec tsx scripts/warm-charts.ts` (na pasta `apps/web`).

---

## Arquitetura

- **Serviço 1 — Postgres** (plugin gerenciado da Railway).
- **Serviço 2 — web** (Next.js): serve tudo + **processa relatórios + emite o SSE** (real-time vive aqui). Domínio público → resolve o webhook do Asaas e o cron de alertas.
- **Worker do WhatsApp**: roda **no PC do Marcos por enquanto** (onde o QR do Baileys já autentica fácil), apontando pro domínio da Railway. Depois pode virar um 3º serviço.

### Por que o worker pôde sair de dentro do web
Antes o worker gravava o PDF em disco e mandava só o **caminho** (`pdfPath`) → web e worker precisavam do **mesmo disco**. Mudança feita em 2026-06-10: o worker agora **envia o PDF (bytes via multipart)** quando o `APP_URL` é remoto; `/api/reports/process` aceita upload e grava no próprio tmp. Local (localhost) continua usando `pdfPath` (zero mudança no dev). → web e worker **100% desacoplados**, real-time preservado (o web processa e emite o SSE no seu único processo).

## Mudanças no repo (já feitas)

- `apps/web/package.json`: `build` agora roda `prisma generate && next build`; novo script `start:railway` (`prisma migrate deploy && next start`).
- `apps/web/src/app/api/reports/process/route.ts`: aceita `multipart/form-data` (upload) além do `pdfPath` (compat local).
- `apps/web/src/services/whatsapp/whatsapp.service.ts`: `sendToProcessor` envia bytes quando remoto, `pdfPath` quando local (`isRemoteApp()`).
- `railway.json` (raiz): builder NIXPACKS; build `pnpm install --frozen-lockfile && pnpm build:web`; start `prisma migrate deploy && next start` (filtrado p/ @gorila/web).

## Passo a passo no painel da Railway

1. **Criar projeto** → "Deploy from GitHub repo" → escolher o repo `gorilaAlpha-web`.
2. **Adicionar Postgres**: New → Database → **PostgreSQL**.
3. No **serviço web**:
   - Railway lê o `railway.json` (build + migrate + start automáticos).
   - Em **Variables**, referenciar o banco: `DATABASE_URL = ${{Postgres.DATABASE_URL}}` (use a connection string interna do plugin).
   - Setar as demais env vars (lista abaixo).
   - Em **Settings → Networking → Generate Domain** → pega a URL pública (ex.: `https://gorilaalpha-web-production.up.railway.app`).
4. **Setar `NEXT_PUBLIC_APP_URL` e `APP_URL`** com essa URL e **redeploy** (o build embute a `NEXT_PUBLIC_*`).
5. **Migrations**: rodam sozinhas no start (`prisma migrate deploy`). Se quiser **seed**, rodar uma vez: `railway run pnpm --filter @gorila/web seed`.

## Env vars (setar no serviço web)

| Var | O que é | Observação |
|---|---|---|
| `DATABASE_URL` | Postgres | referência `${{Postgres.DATABASE_URL}}` |
| `JWT_ACCESS_SECRET` | segredo do JWT (auth) | gerar string longa aleatória |
| `ANTHROPIC_API_KEY` | Claude (IA + processamento de relatório) | |
| `RESEND_API_KEY` | e-mail (verificação, alertas, reset) | |
| `RESEND_FROM_EMAIL` | remetente | **precisa domínio verificado** no Resend p/ enviar a qualquer e-mail; senão só entrega ao dono da conta |
| `ASAAS_ENV` | `sandbox` ou `prod` | começar em `sandbox` |
| `ASAAS_API_KEY_SANDBOX` | chave Asaas sandbox | ⚠️ **SEM o `\` de escape** aqui (o `\$` era só pro dotenv local; na Railway vai a chave crua `$aact_...`) |
| `ASAAS_API_KEY_PROD` | chave Asaas produção | só quando for cobrar de verdade |
| `ASAAS_WEBHOOK_TOKEN` | valida o webhook | gerar string; configurar igual no painel Asaas |
| `CRON_SECRET` | protege `/api/cron/check-alerts` | gerar string |
| `REPORTS_PROCESS_SECRET` | protege `/api/reports/process` | gerar string; **o worker usa a mesma** |
| `FRED_API_KEY` | renda fixa internacional (FRED) | |
| `GOOGLE_CLIENT_ID` + `NEXT_PUBLIC_GOOGLE_CLIENT_ID` | login Google | adicionar a URL da Railway nas origens autorizadas no Google Console |
| `NEXT_PUBLIC_APP_URL` + `APP_URL` | URL pública | a do domínio Railway |
| `ADMIN_EMAILS` | e-mails admin | |
| `REPORTS_TEMP_PATH` | (opcional) pasta tmp dos PDFs | default `./tmp/reports` serve |

## Pós-deploy

- **Asaas**: painel → Webhooks → `https://<dominio>/api/webhooks/asaas` + setar o mesmo `ASAAS_WEBHOOK_TOKEN`. Agora a ativação do Pro fica **automática** (não precisa mais do botão "Já paguei").
- **Cron de alertas**: a rota é `POST /api/cron/check-alerts` com header `Authorization: Bearer $CRON_SECRET`. A Railway não tem cron nativo simples → opções: **cron-job.org** (grátis) ou **GitHub Actions** schedule chamando a URL; ou um Railway Cron service.
- **Worker (no PC)**: setar no `.env` local `NEXT_PUBLIC_APP_URL=https://<dominio-railway>` e `REPORTS_PROCESS_SECRET=<mesmo da Railway>`, rodar `pnpm --filter @gorila/web whatsapp`. Os PDFs do WhatsApp passam a cair no **banco de produção** e aparecem no Feed ao vivo.

## Fixes do build de produção (2026-06-10)

Primeiro `pnpm build:web` falhou; corrigido:
- **`/rankings` quebrava** (`useSearchParams()` sem `<Suspense>` no prerender estático). Era página `"use client"` no topo. Fix: `export const dynamic = "force-dynamic"` no `(app)/layout.tsx` → todas as páginas logadas viram dinâmicas (corretas, são auth + dados ao vivo). As telas de auth (login/redefinir/verificar) já usavam `<Suspense>`.
- **`yahoo-finance2` exige Node ≥ 22** (estava 20). Fix: `engines.node >=22` no `package.json` raiz + Dockerfile `node:22-slim`. ⚠️ Na Railway, garantir Node 22 (Nixpacks lê o `engines`).
- Resultado: **build OK**, páginas logadas como `ƒ (Dynamic)`.

## Gráficos / dados em produção (2026-06-11)

**Problema raiz:** o **Yahoo Finance bloqueia IPs de datacenter** (Railway). Então qualquer fetch ao vivo do Yahoo na Railway falha (`fetch failed` / 502). Isso afetou: seed/sync (fundamentos) e gráficos (price-chart, technical-insights).

**Solução de gráficos (`chart-data.ts`):**
- **B3/BDR** → **brapi.dev** (datacenter-friendly). ⚠️ token **free só permite range até `3mo`** (e tem limite de req). Por isso o histórico completo vem do **cache do banco** (`stockPriceHistory`), aquecido de fora; brapi 3mo é fallback ao vivo + merge do recente.
- **US** → **Twelve Data** (`time_series`, free 800/dia, 8/min). Cobre US **ao vivo, 5 anos**, na Railway. (Free **não** cobre B3 → 404; Finnhub free **não** dá candles → descartado.)
- **Fallback gracioso**: se a fonte falhar, serve o cache; merge não deixa o cache encolher → **nunca 502**.
- **Cache diário = 24h**; **intraday = sempre ao vivo**.
- Envs na Railway: `BRAPI_TOKEN`, `TWELVEDATA_API_KEY`.

**Aquecimento do cache (`scripts/warm-charts.ts`):** roda do PC (Yahoo funciona em IP residencial) contra o banco de produção (via `DATABASE_PUBLIC_URL`). Popula `stockPriceHistory` (5 anos). Só **B3/BDR** precisam (US é ao vivo via Twelve Data). ⚠️ Yahoo throttla o IP após algumas centenas de fetches → rodar com delay e/ou em várias passadas. Em 2026-06-11 ficaram ~143/181 B3/BDR completos; ~38 (VALE3, SUZB3, etc.) ainda em brapi 3mo — **o sync agendado do worker completa**.

**Fundamentos (seed/runSync):** mesma lógica — rodar do PC/worker (Yahoo) contra a prod. Seed de 2026-06-11: 671/716 (US 490, B3 146, BDR 35).

**Dedup:** `technical-insights` era chamado 2x (stock-chart + mm200-insights) → agora os dois usam o service com cache de 30s = 1 req.

**Próximos passos combinados:** (1) **cron de alertas** (`/api/cron/check-alerts` + `CRON_SECRET` via cron-job.org/GitHub Actions); (2) **worker WhatsApp** (Baileys fora da Railway → prod); (3) **sync agendado** no worker (seed + warm-charts) p/ manter fundamentos + B3 frescos automaticamente.

## Pendências / cuidados

- Validar o **build de produção** antes (`pnpm build:web`) — primeiro deploy costuma falhar por env faltando.
- **Resend**: sem domínio verificado, e-mails só vão pro dono da conta (o log de dev não roda em prod).
- Mover o worker pra um serviço Railway depois (QR do Baileys via logs uma vez) quando não quiser depender do PC ligado.
