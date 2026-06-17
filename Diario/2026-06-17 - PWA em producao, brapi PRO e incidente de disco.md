# 2026-06-17 — PWA mobile em produção, brapi PRO e incidente de disco

← [[00 - Home]] · sessão anterior: [[Diario/2026-06-09 - Fundacao PWA Mobile]] · deploy: [[Deploy - Railway (Producao)]]

> Arco longo (11→17/06): das telas do PWA até o app mobile **no ar em produção** (2º serviço Railway com roteamento mobile→PWA), troca do brapi free pelo **PRO** (fundamentos B3/BDR + 5 anos), uma série de bugs de produção e um **incidente de disco** no Postgres.

## 1. Telas do PWA (skill `implement-figma-mobile`)
Construídas todas as telas mobile, cada uma ancorada nos dados REAIS do backend (omitindo o que o Figma mostrava sem lastro):
- **Onboarding/Welcome**, **Login + Cadastro** (fluxo 4 passos), **Home + Feed de relatórios**, **Screener**, **Stock Detail** (gráfico lightweight-charts com mm50/200/400 + seção "comportamento histórico na MM200" + Balanço & DRE), **Watchlist**, **Alertas**, **Perfil** (espelhado do web), **Compare** (no bottom menu).
- **`@gorila/core`** estendido: auth + **refresh token**, stocks, scoring (portado), watchlist, alerts, account. Ícones novos no `@gorila/ui`.
- **Ícones PNG** gerados (via `sharp`) p/ instalação no iOS; manifest com PNG + maskable; ícone = gorila do `login.svg`.
- **CPF/CNPJ**: máscara + validação (dígitos verificadores) no PWA **e** no web.
- **Checkout 500** corrigido: callback do Asaas só é enviado se a URL for pública (localhost quebrava em dev) → dá pra assinar local (pagamento na página do Asaas + sync).

## 2. Roteamento mobile → PWA (o pulo do gato)
Decisão: **mobile inteiro vai pra PWA**, desktop fica no web. Mecanismo = detecção de User-Agent, **não** instalação (PWA é site; instalar é bônus).
- Vive em `apps/web/src/proxy.ts` (Next 16 renomeou `middleware.ts`→**`proxy.ts`**; ter os dois CRASHA o web). Mobile UA não-bot → `307` pra PWA; bots ficam no web (SEO); escape `?view=web`; auth gate preservado.
- **Gotcha**: o proxy só lê env de `apps/web/.env` (não do `.env` raiz nem do `environment:` do compose). Em prod, o `Dockerfile` do web recebe `ARG/ENV PWA_URL` no build.

## 3. PWA em produção (2º serviço Railway)
- `Dockerfile.pwa` + `railway.pwa.json` (raiz). 2º serviço `gorila-mobile`: Root Directory vazio, **Config-as-code = `railway.pwa.json`**, `WEB_INTERNAL_URL` = URL pública do web, sem volume.
- Web service: `PWA_URL` = URL pública do PWA.
- **Proxy `/api` em runtime**: os `rewrites()` do `next.config.ts` são avaliados em **build-time** → o destino ficava "assado" como `localhost:3000` (ECONNREFUSED em prod). Trocado por um **route handler catch-all** `apps/pwa/src/app/api/[...path]/route.ts` que lê `WEB_INTERNAL_URL` no request e repassa método/headers/body/set-cookie. Validado: login mobile em prod ✅.

## 4. Robustez de gráficos/MM200 — nunca mais 502
- `getDailySeries` não relança mais erro (retorna cache vazio); rotas `price-chart` e `technical-insights` caem pra `200` vazio (`unavailable:true`) em vez de `502`. Retry no Yahoo.

## 5. Fonte de dados — brapi PRO (decisão de custo)
Pesquisa de provedores (2026): real-time de verdade na B3 = inviável (taxa de bolsa, milhares/mês); APIs baratas dão B3 com delay; US real-time é barato. Nenhuma fonte grátis dá 5 anos de B3 confiável (Yahoo bloqueado na Railway, brapi free=3mo, Twelve Data B3=pago).
- **Marcos assinou o brapi PRO (~R$117/mês).** Token PRO destrava **5y/max** + **fundamentos completos** da B3/BDR.
- `brapi.ts`: novo path PRO (`/api/quote` em lote, `fundamental=true&modules=...`) mapeia `defaultKeyStatistics`+`financialData` → P/L, ROE, ROA, P/VP, EV/EBITDA, margens, dívida, liquidez, DY, EBITDA, receita, lucro. `providers/index.ts`: **B3/BDR usam brapi PRO primeiro**, Yahoo fallback; US no Yahoo/Twelve Data. Ativa com env `BRAPI_TOKEN`(PRO) + `BRAPI_HISTORY_RANGE=5y`.
- Bug do `price` corrigido: `sync/index.ts` forçava `price=null` quando `pl` era null → empresas com prejuízo (MGLU3, BHIA3/Casas Bahia) sumiam. Agora `price: stock.price`.

## 6. 🔥 Incidente de disco (Postgres prod)
- Sync em massa começou a dar `PostgresError 53100 "No space left on device"` — **o volume do Postgres era de 500 MB e encheu**. Disco 100% cheio bloqueia ATÉ conectar (`could not write init file`). Isso (não o proxy, como pensei a princípio) era a causa de TODO sync dar 0/erro.
- **Culpado**: `stock_price_history` (cache de gráficos) com `BRAPI_HISTORY_RANGE=5y` → **972.669 linhas / 310 MB (96% do banco)**.
- **Resolução**: Railway → clicar no **VOLUME** (não nas Settings do serviço) → **Live resize** pra 20GB (cobra só por uso) → `TRUNCATE stock_price_history` (cache regenerável). **DB caiu de 321MB → 10MB.**
- Depois: sync dos fundamentos rodou — **647 ações com P/L preenchido** (B3/BDR via brapi PRO + US). ABEV3 P/L 16.5, PETR4 4.6, MGLU3 29.4, BHIA3 -0.33 (prejuízo, e aparece).

## 7. Gráfico não aparecia no 1º load
Sintoma: o gráfico só renderizava depois de trocar de período e voltar; 1D/5D não apareciam.
- Causa A: lightweight-charts criado com container **width 0** (layout não assentou) → desenhava vazio. Fix: refit no `ResizeObserver` + `requestAnimationFrame` após `setData`.
- Causa B (1D/5D): o token **free não fazia intraday** (5m/15m) — o PRO faz. Resolvido com o token PRO.

## 8. Feed
- Sem ingestão real em prod (bot WhatsApp não hospedado). **Populado manualmente** com seed de 10 relatórios realistas (`apps/web/src/scripts/seed-reports.ts`) só pra ficar visível. Plano free mostra 6, pro mostra todos.
- **Deploy do bot pronto** (`Dockerfile.whatsapp` + `railway.whatsapp.json`) — falta o setup do Marcos (serviço + número dedicado + QR + `ANTHROPIC_API_KEY`/`REPORTS_PROCESS_SECRET`).

## Caveats honestos
- `next.config` rewrites e o env do `proxy.ts` são **build-time** → exigiram contornos (route handler runtime; `apps/web/.env`).
- **Churn do cache** (`getDailySeries` faz delete+reinsert do histórico inteiro a cada refresh) — autovacuum segura, ok em 20GB, mas é ineficiente. Append-only seria melhor (não corrigido).
- Sync rodado do container contra a prod (token PRO + `DATABASE_PUBLIC_URL`). O sync agendado em prod usa o `BRAPI_TOKEN` do serviço web.

## Próximo
- **Feed via WhatsApp**: criar o serviço `gorila-whatsapp` (config `railway.whatsapp.json`, volume `/data`, número dedicado, QR) + `ANTHROPIC_API_KEY` + `REPORTS_PROCESS_SECRET` no web.
- (Opcional) otimizar o churn do `stock_price_history` pra append-only.
- (Opcional) reduzir retenção do histórico se quiser voltar pro plano básico.
