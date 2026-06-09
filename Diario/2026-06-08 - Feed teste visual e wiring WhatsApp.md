# 2026-06-08 — Feed: teste visual OK + wiring do WhatsApp

← [[00 - Home]] · sessão anterior: [[Diario/2026-05-29 - Feed de Relatorios de Corretoras]] · sprint: [[Sprints/Sprint 08 - Feed de Relatorios de Corretoras]]

## Contexto

Continuação do [[Diario/2026-05-29 - Feed de Relatorios de Corretoras]]. Naquela sessão o app ficou pronto mas **sem teste em runtime**. Hoje: subimos o dev, testamos visualmente e deixamos o WhatsApp pronto pra conectar.

## Infra de dev resolvida

- **Porta 3000 estava presa pelo container Docker `gorila-alpha-app`** (não era node solto). `docker stop gorila-alpha-app` liberou. Banco (`gorila-alpha-db`, 5433) seguiu de pé.
- **`next dev` não achava `JWT_ACCESS_SECRET`**: o `.env` mora na **raiz** do monorepo, mas o Next carrega `.env` da pasta do app (`apps/web`). Solução: **copiar `.env` da raiz → `apps/web/.env`** (gitignored). Causa: no Docker o compose injeta env; em dev não.
  - ⚠️ Virou cópia → se editar a raiz, re-sincronizar `apps/web/.env`.

## Teste visual (passou)

- Seed de exemplo: `apps/web/scripts/seed-reports.ts` insere 4 reports `sourceType: "seed"` (GGPS3 COMPRA, PETR4 COMPRA + NEUTRO, setorial Itaú BBA) e recalcula consenso de GGPS3 e PETR4.
- `/reports`: cards renderizam certo (badge de recomendação, alvo/potencial, strengths/risks com ícones, card SETORIAL azul com tags). Filtros e busca funcionando.
- `/stock/PETR4`: **ConsensusWidget aparece** abaixo dos score modules (1 Compra, 1 Neutro, alvo médio R$ 41,00, potencial vs preço real). ✅ Marcos aprovou ("ficou mt bom").

## Wiring do WhatsApp (pra conectar de verdade)

- **`dotenv`** adicionado; `start-whatsapp.ts` faz `import "dotenv/config"` (tsx não carrega `.env` sozinho). Roda com cwd `apps/web` → lê `apps/web/.env`.
- **`REPORTS_PROCESS_SECRET`** gerado (`crypto.randomBytes(32).base64`) e gravado na raiz + `apps/web/.env`. Placeholders `WHATSAPP_GROUP_IDS`, `REPORTS_TEMP_PATH`, `WHATSAPP_SESSION_PATH` adicionados.
- **Listagem de grupos ao conectar**: resolve o ovo-e-galinha do group ID. Ao abrir conexão, o serviço chama `groupFetchAllParticipating()` e loga `subject -> id` de cada grupo, pra copiar pro `WHATSAPP_GROUP_IDS`.
- **Extração de PDF mais robusta**: `extractDocument()` cobre `documentMessage`, `documentWithCaptionMessage` e wrappers `ephemeralMessage`. Download usa mensagem sintética `{ key, message: { documentMessage } }`.
- `REPORTS_TEMP_PATH=./tmp/reports` resolve igual no script e no dev (ambos cwd `apps/web`) → `apps/web/tmp/reports`, então o `POST /api/reports/process` valida o path certo.
- **Baileys 7.0.0-rc13 quebrou** com `ERR_PACKAGE_PATH_NOT_EXPORTED` (dep nativa `whatsapp-rust-bridge` com `package.json` sem `exports`, incompatível com Node 20.11 + tsx). **Downgrade pra `@whiskeysockets/baileys@6.7.23`** (JS puro, estável) resolveu — toda a API usada existe na v6. Smoke test: script cria `tmp/reports` + `whatsapp-session` e chega no QR sem erro de resolução.

## Fluxo pra ligar (pendente de execução pelo Marcos)

1. Reiniciar `pnpm dev:web` (pra dev pegar o `REPORTS_PROCESS_SECRET`).
2. 2º terminal: `pnpm --filter @gorila/web whatsapp` → escanear QR → ver lista de grupos.
3. Copiar o ID do grupo alvo pra `WHATSAPP_GROUP_IDS` (raiz + `apps/web/.env`) → reiniciar o script.
4. Enviar um PDF de relatório no grupo → processa via Claude → aparece em `/reports`.

## Conexão real — funcionou end-to-end ✅

- **405 antes do QR**: faltava `fetchLatestBaileysVersion()` → sem `version`, o WhatsApp rejeitava com 405 e fechava antes de emitir o QR. Passar a versão resolveu; QR apareceu, sessão salva em `whatsapp-session`.
- Grupo monitorado: **VT Relatórios 5** (`120363317359033134@g.us`). `WHATSAPP_GROUP_IDS` gravado na raiz + `apps/web/.env` (cuidado: o `@g` quebrou o `perl` por interpolação de array — usei `$ENV{GID}`).
- **3 PDFs reais processados**: `cardapio do giba` → JHSF3/STOCK, `analises ArenaXP` → ok, `tendencias` → MACRO. Todos via XP.
- **Bug de parse**: 1 PDF veio com JSON em ` ```json ` e meu regex de fence falhou (JSON truncado em 1500 tokens, sem ``` de fechamento). Fix: fatiar `{`..`}` + `max_tokens` 4096. Reprocessado com sucesso pelo endpoint.

## Realtime (SSE) + custo

- **Custo Claude por PDF**: Haiku 4.5 ($1/MTok in, $5/MTok out). PDF é processado página a página (~1.5–3k tokens/página). Estimativa: ~R$ 0,07 (curto) a ~R$ 0,30 (15-20 pág), típico ~R$ 0,13. Adicionei **log real de tokens+custo** no processador (`logCost(message.usage)`) → aparece no terminal do `dev:web`.
- **Feed em tempo real via SSE** (não WebSocket — Next não hospeda WS nativo; SSE é server→client nativo, zero infra). `lib/report-events.ts` (EventEmitter singleton via globalThis) → `processReport` emite `report:created` → `GET /api/reports/stream` faz stream `text/event-stream` (guardSession, heartbeat 25s, cleanup no abort) → `ReportFeed` usa `EventSource`, insere o card no topo na hora (filtros respeitados via ref, sem reconectar) e mostra "● ao vivo".
- Gargalo real é a extração do Claude (~5–17s/PDF), não o transporte. SSE elimina o atraso de até 60s do polling.
- Verificado: rota responde 401 sem auth (guard ok); tsc + lint limpos.
- ⚠️ Event bus em memória só serve 1 instância (dev / 1 container). Prod multi-instância → Redis pub/sub.

## Rate limit, extração de texto e filtro de não-relatórios

- **429 rate_limit_error**: tier 1 = 50k tokens input/min no Haiku. PDF como imagem gasta 30-50k/PDF → bursts estouravam. Também causava "fetch failed" (SDK re-tentava 429 por ~3,5min e o fetch do script expirava).
- Decisão do Marcos: **só extração de texto agora**, modo imagem (que precisa subir o tier) fica pra depois.
- **`unpdf`** extrai o texto do PDF; mando **texto** pro Claude (3-5x menos tokens, mais barato/rápido, sem 429). Se o PDF não tiver texto (escaneado/imagem), retorna erro claro "modo imagem não habilitado" em vez de cair no modo caro. `maxRetries: 4` na chamada.
- **Filtro de não-relatórios (`IGNORE`)**: o grupo manda **jornais** (Estadão, O Globo, Valor, WSJ, FT) que o Claude forçava em MACRO/Desconhecida (lixo no feed). Prompt novo manda retornar `{"reportType":"IGNORE"}` pra jornal/notícia/propaganda; o processador descarta (não salva, deleta o PDF). Confirmado: Estadão/Globo/WSJ → `skipped`.
- Limpei 3 jornais que entraram antes do fix (`cleanup-reports.ts`).
- **Resultado**: dezenas de relatórios reais entraram ao vivo (Safra, BB/RAIZ4, BB/PicPay, Plural/CPRI, XP, Eleven, Hub do Investidor…). Feed funcionando bem.
- Pendência: PDFs só-imagem (ex: FT escaneado) ficam parados em `tmp/reports` até habilitar modo imagem.

## Tradução e densidade do resumo

- **Tradução**: relatórios em inglês processados antes da regra de idioma ficaram em inglês. Backfill (`scripts/translate-reports.ts`) traduz os campos de texto (summary/sector/strengths/risks/company) via Claude, mantendo tickers/nomes. Setor era teimoso ("Capital Goods" voltava igual) → reforcei o prompt com exemplos ("Capital Goods" → "Bens de Capital"). Scan final: 0 em inglês. Regra de tradução de setor adicionada também no processador (próximos).
- **Resumo denso (pedido do Marcos)**: STOCK = ~5 frases (tese do call + números: alvo, múltiplos, resultados) — destaque ainda é o badge COMPRA/VENDA/NEUTRO. **SECTOR e MACRO** = resumo DENSO (5-8 frases, 1-2 parágrafos, com contexto/números/teses/conclusões). Validado: MACRO Eleven saiu com resumo rico (Selic 14,25-14,50%, Goldman, Brent, minério, ouro…). Só vale pros novos — os antigos já estão comprimidos (PDF deletado, não dá pra re-densificar sem reenviar).
- PDFs só-imagem (FT, Morning Call DeltaX) seguem pendentes (modo imagem futuro).

## Feed como entrada principal

- **Feed virou o 1º item do sidebar** (antes do Screener) em `app-sidebar.tsx`.
- **Landing da área logada agora é `/reports`**: default pós-login `next ?? "/reports"` (`login-screen.tsx`) e sucesso do Google signup → `/reports` (`register-screen.tsx`). O CTA "Explorar no Screener" da welcome-screen ficou (é rótulo específico, não a landing).

## Card de upsell do sidebar

- O card "PORTFOLIO INSIGHT / Ative alerts… Elite" era hardcoded: aparecia sempre e o botão não fazia nada.
- Agora **só aparece pra `plan === "free"`** (plano pego via `useWatchlist()`), e o botão **abre o `useUpgradeModal()`** (`feature: "alerts-count"`) → fluxo de upgrade existente (→ `/settings/subscription`).
- Texto "Elite" → "Pro" (não existe tier Elite no código; pago = `pro`).
- Usuário do Marcos é `pro/active` → card escondido pra ele.

## Bug: preferências do onboarding não persistiam

- **Sintoma**: Settings → Preferências de investimento não refletia as escolhas (mercados/setores/volume).
- **Causa raiz**: o onboarding (`register-preferences-screen`) coletava as escolhas só num **rascunho em `sessionStorage`** (`signup-draft`) que **nunca era enviado ao backend**. O `POST /api/auth/register` criava `preferences: { create: {} }` **vazio**. Ou seja, perfil E preferências do cadastro eram silenciosamente descartados. (Settings em si está correto — lê/salva certo; o banco é que estava vazio.)
- **Correção (persistir na criação da conta)**:
  - `registerSchema` agora faz `.merge(profilePayloadSchema)` → aceita `profile` e `preferences` opcionais.
  - `RegisterPayload` estendido com `profile?`/`preferences?`.
  - `register` route mapeia e persiste em `preferences: { create/update: prefsData }` (incl. birthDate via `brazilianDateToDate`, "any"→null no volume).
  - `register-preferences-screen` envia `profile` (do draft) + `preferences` (seleção atual) no `register()`.
- **Contas já criadas** (ex: a do Marcos, vazia) não recuperam as escolhas originais (sessionStorage já foi). Mas o **save do Settings funciona** (schema + `/api/profile` OK) — basta setar lá.

## Robustez do ingest (fila)

- Motivação: no burst de relatórios, ~8 PDFs ficaram parados (chamadas concorrentes → 429 + "fetch failed" pelo tempo da chamada).
- `lib/report-queue.ts`: fila serial em memória (globalThis). `enqueueReport(path)` → `drain()` processa 1 por vez.
- `POST /api/reports/process` agora **enfileira e responde 202** (não espera o Claude) → script do WhatsApp não estoura timeout.
- Retry: 429 → re-enfileira após 30s (até 3 tentativas). Quarentena: "modo imagem" → `pending-image/`, falha definitiva → `failed/` (declutter do `tmp/reports`).
- Testado: 2 PDFs só-imagem (FT, Morning Call) → 202 na hora → movidos pra `pending-image/`, root limpo.
- Edge conhecido: fila em memória; se o servidor reiniciar com PDFs no root de `tmp/reports`, eles não re-enfileiram sozinhos (follow-up: scan no boot).

## Histórico + dedup + scan no boot

- **Dedup persistente**: novo campo `Report.sourceMessageId` (único, migration criada na mão + `migrate deploy` porque `migrate dev` é interativo no warning de unique; `prisma generate` travava pelo lock do dev server → Marcos parou o dev pra regenerar). Processor checa o id antes de processar e ignora duplicado. Live grava `message.key.id`.
- **Histórico** (`WHATSAPP_HISTORY_LIMIT`, default 0 = off): `syncFullHistory` + `messaging-history.set` → junta PDFs do grupo, ordena por data desc, importa os últimos N (deduplicados). Opt-in pra não pesar a operação normal; ligar, rodar 1x, voltar pra 0. Nuance: os mais recentes já foram capturados ao vivo (sourceMessageId null antes do feature) → backfill agora duplicaria os de hoje; útil mesmo é pra preencher gap de antes da 1ª conexão / dali pra frente.
- **Scan no boot**: `src/instrumentation.ts` chama `scanPendingReports()` ao subir → re-enfileira PDFs órfãos do `tmp/reports`.
- tsc + lint limpos; campo testado (create/find/delete por sourceMessageId).

## Pendências / notas

- **Custo Claude**: cada PDF roda no Haiku. Monitorar com relatórios grandes.
- **Docker prod**: script (host) e app (container) não compartilham `tmp/reports` sem volume; e rota nova exige rebuild.
- **Env duplicado** (raiz vs `apps/web/.env`) é dívida técnica de DX — avaliar `next.config` carregar da raiz, ou symlink.
- Sem paywall no Feed ainda.
