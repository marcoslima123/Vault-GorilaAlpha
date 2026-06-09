# Sprint 08 — Feed de Relatórios de Corretoras

**Esforço:** ~1 dia · **Status:** 🟢 CONCLUÍDO — end-to-end funcionando (2026-06-08): WhatsApp → PDF → Claude → feed

← [[Sprints/Sprint 07 - Alerts e Paywall]] · diários: [[Diario/2026-05-29 - Feed de Relatorios de Corretoras]] · [[Diario/2026-06-08 - Feed teste visual e wiring WhatsApp]]

## Objetivo

Feed estilo Twitter de análises de corretoras, extraídas automaticamente de PDFs recebidos por WhatsApp. Menu próprio no sidebar (**Feed**). Cada relatório vira um card; ações ganham um **Consenso de Analistas** no Stock Detail.

## Entregáveis

### Banco
- [x] Model `Report` (reportType, ticker?, company?, broker, recommendation?, targetPrice?, currentPrice?, upside?, analystName?, reportDate, sector?, summary, strengths[], risks[], companiesMentioned[], sourceType, processedAt, createdAt). Índices: ticker, broker, reportType, createdAt.
- [x] Model `ReportConsensus` (ticker unique, buyCount, neutralCount, sellCount, avgTargetPrice?, lastUpdated).
- [x] Migration `add_reports`.

### Processamento (Claude API)
- [x] `services/reports/report-processor.ts` — PDF → base64 → Claude (`claude-haiku-4-5`, suporta document/PDF) → JSON → salva → deleta. Tipos STOCK / SECTOR / MACRO.
- [x] `lib/report-consensus.ts` — recálculo de consenso (janela 90d) ao salvar STOCK.

### API
- [x] `GET /api/reports` — filtros + paginação.
- [x] `GET /api/reports/consensus/[ticker]` — 404 se vazio.
- [x] `POST /api/reports/process` — interno, `Bearer REPORTS_PROCESS_SECRET`, pdfPath travado em `REPORTS_TEMP_PATH`.

### UI
- [x] `ReportCard` (STOCK + SECTOR/MACRO).
- [x] `ReportFeed` (tabs, busca ticker, multi-select corretora, scroll infinito).
- [x] **Realtime via SSE** (2026-06-08): `GET /api/reports/stream` + `EventSource`. Card aparece sozinho no topo quando o PDF é processado; indicador "● ao vivo". Substituiu o polling de 60s. Event bus em memória (`lib/report-events.ts`); prod multi-instância pediria Redis pub/sub.
- [x] **Log de custo por PDF** no processador (`tokens in/out · US$ · R$`) — Haiku 4.5 ≈ centavos/PDF.
- [x] `ConsensusWidget` (barras + alvo médio + potencial).
- [x] Página `/reports` + item **Feed** no sidebar.
- [x] `ConsensusWidget` integrado no Stock Detail abaixo dos score modules.

### WhatsApp (Baileys)
- [x] `services/whatsapp/whatsapp.service.ts` — QR no terminal, sessão persistida, monitora grupos, baixa PDF, chama o processador via HTTP.
- [x] `scripts/start-whatsapp.ts` + `pnpm --filter @gorila/web whatsapp`.
- [x] **Env via dotenv** (`import "dotenv/config"`) — tsx não carrega `.env` sozinho.
- [x] **`REPORTS_PROCESS_SECRET`** gerado e gravado (raiz + `apps/web/.env`).
- [x] **Listagem de grupos ao conectar** (`groupFetchAllParticipating`) pra descobrir o `WHATSAPP_GROUP_IDS`.
- [x] **Extração robusta de PDF** (documentMessage / documentWithCaption / ephemeral).
- [ ] **`WHATSAPP_GROUP_IDS`** preenchido com o grupo alvo (depende do scan).
- [x] **Teste real (2026-06-08):** QR escaneado, 3 PDFs do grupo "VT Relatórios 5" processados (JHSF3/STOCK, ArenaXP, tendencias/MACRO) e exibidos no feed.

## Critério de pronto

- [x] type-check + lint limpos.
- [x] Report inserido manualmente aparece no feed; ConsensusWidget aparece no Stock Detail (testado 2026-06-08, GGPS3 + PETR4).
- [x] PDF real recebido pelo WhatsApp é processado e aparece no feed (2026-06-08).

## Notas técnicas / decisões

- `ticker` sem FK pro `Stock` (igual `TechnicalInsight`) — aceita ticker fora da base e SECTOR/MACRO.
- Processador via **HTTP** (não import direto) pra desacoplar o script standalone do Prisma/Next/alias.
- **Baileys fixado em `6.7.23`** (não usar 7.x rc — quebra com `ERR_PACKAGE_PATH_NOT_EXPORTED` pela dep nativa `whatsapp-rust-bridge`). QR manual; `useMultiFileAuthState` aliasado pra `loadAuthState` (regra de lint).
- **`fetchLatestBaileysVersion()` é obrigatório** — sem passar `version`, o WhatsApp fecha a conexão com **405** antes de emitir o QR (protocolo desatualizado).
- **Parse do Claude**: fatiar do primeiro `{` ao último `}` (em vez de regex de fence) tolera ` ```json `, ``` e texto em volta. `max_tokens: 4096` evita truncar o JSON em relatórios grandes.
- Extração que falha **não deleta o PDF** → dá pra reprocessar via `POST /api/reports/process`.
- **Extração de texto (`unpdf`)**: manda texto (não imagem) pro Claude — 3-5x menos tokens, sem 429. Fallback p/ erro "modo imagem não habilitado" se PDF for escaneado (modo imagem fica pra quando subir o tier). `maxRetries: 4`.
- **Filtro `IGNORE`**: jornais/notícias não viram relatório (prompt retorna `IGNORE`, processador descarta). Sem isso o grupo polui o feed com jornais.
- Rate limit Haiku tier 1 = 50k tokens input/min (subir tier = comprar créditos, pendente do Marcos).
- **Idioma**: prompt traduz tudo pra PT-BR (inclusive nomes de setor). Backfill `scripts/translate-reports.ts` p/ os antigos em inglês.
- **Densidade do resumo**: STOCK = ~5 frases (tese + números, com o badge de call em destaque); SECTOR/MACRO = resumo denso (5-8 frases). Decisão do Marcos.
- **Robustez do ingest** (`lib/report-queue.ts`): `/process` enfileira e responde **202** (não-bloqueante → fim do "fetch failed"); fila serial (1 por vez, evita 429 de burst); retry de 429 com backoff 30s (até 3x); quarentena de PDFs só-imagem → `tmp/reports/pending-image/` e falhas → `failed/`.
- **Scan no boot** (`src/instrumentation.ts` → `scanPendingReports`): ao subir, varre `tmp/reports` e re-enfileira PDFs órfãos (fecha o edge da fila em memória).
- **Dedup `sourceMessageId`** (migration `add_report_source_message_id`, campo único): processor checa antes de processar e ignora duplicados. Mensagens ao vivo gravam `message.key.id`. Protege contra reprocessamento em reconexão/restart.
- **Histórico opt-in** (`WHATSAPP_HISTORY_LIMIT`, default 0): com `syncFullHistory`, no `messaging-history.set` junta os PDFs do grupo, ordena por data desc e importa os últimos N (deduplicados). Reports de antes do feature (sourceMessageId null) podem duplicar uma vez se reimportados.
- Filtro de corretora é **client-side** (sobre o que carregou) — API só aceita 1 broker por vez. Evoluir pra server-side OR se necessário.

## Riscos / pendências de infra

- **Rota nova exige rebuild da imagem Docker** (padrão conhecido).
- **PDF dir compartilhado**: host (script) vs container (app) não compartilham `/tmp/reports` sem volume — resolver em prod.
- Custo Claude por PDF: monitorar (Haiku barato, mas relatórios podem ter muitas páginas).
- Sem paywall ainda — decidir se Feed é free ou pro.
