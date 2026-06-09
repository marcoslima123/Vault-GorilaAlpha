# 2026-05-29 — Feed de Relatórios de Corretoras

← [[00 - Home]] · sessão anterior: [[Diario/2026-05-27 - Inteligencia Historica MM200]] · sprint: [[Sprints/Sprint 08 - Feed de Relatorios de Corretoras]]

## Objetivo

Feature nova pedida pelo Marcos: um **feed estilo Twitter** que exibe análises de corretoras extraídas automaticamente de **PDFs recebidos via WhatsApp**. Tem menu próprio no sidebar (**Feed**). Stack: Next.js 16 + Prisma + Postgres + Claude API + Baileys.

Implementado na ordem do brief, confirmando cada parte.

## O que entregou (7 partes)

### Parte 3 — Schema Prisma
- Models `Report` e `ReportConsensus` em `apps/web/prisma/schema.prisma`, adaptados às convenções do repo (`@map`/`@@map` snake_case, `cuid()`).
- `ticker` ficou como **string indexada sem FK** (igual `TechnicalInsight`) — permite relatórios de tickers fora da base e SECTOR/MACRO sem ticker.
- Migration `20260529182011_add_reports` aplicada no Postgres do Docker (`localhost:5433`). Tabelas `reports` (índices ticker/broker/report_type/created_at) e `report_consensus` (unique ticker).

### Parte 4 — API Routes
- `GET /api/reports` — filtros `ticker`/`broker`/`recommendation`/`reportType` + paginação (`page`/`limit`, default 20, máx 50). Resposta `{ reports, total, page, totalPages }`.
- `GET /api/reports/consensus/[ticker]` — lê `ReportConsensus` + último STOCK; **404 se não houver relatórios** (pro widget se esconder).
- `POST /api/reports/process` — wrapper protegido por `Bearer REPORTS_PROCESS_SECRET`, `runtime nodejs`, **trava o `pdfPath` dentro de `REPORTS_TEMP_PATH`** (anti path-traversal).
- Decisão: as duas rotas GET usam `guardSession()` (área logada, igual watchlist).

### Parte 2 + 6 — Processador de PDF + consenso
- `apps/web/src/services/reports/report-processor.ts` — `processReport(pdfPath)`: lê PDF → base64 → Claude (`claude-haiku-4-5`, mesmo cliente do `analysis-ai.ts`, com o prompt exato do brief no `system` + cache ephemeral) → strip de fences → parse/valida → salva → recalcula consenso se STOCK → deleta PDF → loga com timestamp.
- `apps/web/src/lib/report-consensus.ts` — `recalculateConsensus(ticker)`: janela **90 dias** por `reportDate`, conta COMPRA/NEUTRO/VENDA, média de `targetPrice`, `upsert` em `ReportConsensus`.

### Parte 5 — Componentes
- `report-card.tsx` — STOCK (broker · ticker · timeAgo, badge de recomendação com cores exatas da spec, alvo + potencial, summary, strengths com `CheckIcon`, risks com `AlertTriangleIcon`, botões Screener/Detalhe) e SECTOR/MACRO (badge azul, empresas como tags).
- `report-feed.tsx` — tabs Todos/Compra/Neutro/Venda, busca por ticker (debounce 400ms), multi-select de corretora, scroll infinito (`IntersectionObserver`), **polling 60s com banner "X novos relatórios"**.
- `consensus-widget.tsx` — barras buy/neutral/sell, alvo médio, potencial vs preço atual, último. Se esconde no 404.
- `timeAgo()` adicionado em `lib/format.ts`. Tipos compartilhados em `src/types/reports.ts`.

### Parte 5 — Página + sidebar
- `src/app/(app)/reports/page.tsx` — header + `<ReportFeed />` centralizado (timeline `max-w-[680px]`).
- Item **Feed** (`/reports`, `DocumentIcon`) adicionado no `app-sidebar.tsx` logo após o Screener.

### Parte 5 (item 15) — ConsensusWidget no Stock Detail
- Inserido em `/stock/[ticker]` **abaixo dos score modules**, recebendo `currentPrice={indicator.price}`. Stock Detail **já era real** (sem mock PETR4 — resolvido em sprint anterior).

### Parte 1 — WhatsApp (Baileys)
- `src/services/whatsapp/whatsapp.service.ts` — QR no terminal (`qrcode-terminal`), sessão persistida em `whatsapp-session/`, reconecta sozinho (exceto logout), monitora `WHATSAPP_GROUP_IDS`, detecta PDFs, baixa pra `REPORTS_TEMP_PATH`, chama `POST /api/reports/process` via HTTP.
- `src/scripts/start-whatsapp.ts` + script `pnpm --filter @gorila/web whatsapp`.
- `.gitignore` ignora `whatsapp-session/` (credenciais!) e `tmp/reports/`.

## Ajustes vs. brief (e por quê)

- **pnpm + tsx** no lugar de `npm install`/`ts-node` (monorepo não tem ts-node).
- **Baileys 7** removeu `printQRInTerminal` → QR tratado manualmente no `connection.update`.
- `useMultiFileAuthState` importado como **`loadAuthState`** — o eslint-plugin-react-hooks confundia o prefixo "use" com Hook. Aliasar evita o erro **sem** `eslint-disable` (comentário é proibido pela regra do projeto).
- `@types/qrcode-terminal` adicionado.
- Processador chamado **via HTTP** (`/api/reports/process`) — o script standalone não precisa carregar Prisma/Next nem resolver alias `@/`.

## Validação

- **type-check (`tsc --noEmit`) e lint (`eslint`) passam limpos** em todos os arquivos novos.
- ⚠️ **Não testado em runtime** ainda: feed no browser, processamento de PDF real com Claude, conexão WhatsApp. Próximo passo é teste end-to-end.

## Pendências pra usar de verdade

1. Setar no `.env` real: `REPORTS_PROCESS_SECRET` (`openssl rand -base64 32`), `WHATSAPP_GROUP_IDS` (formato `...@g.us`).
2. **Rota nova `/api/reports/*` exige REBUILD da imagem Docker** (padrão já documentado) — em dev `pnpm dev:web` já funciona.
3. **Diretório de PDF compartilhado**: script roda no host e salva em `REPORTS_TEMP_PATH`; o app lê do mesmo path. No Docker, o container não enxerga o `/tmp/reports` do host → precisa de volume compartilhado em prod.
4. `pnpm approve-builds` se o Baileys reclamar de build script.

## Próximo

- Teste end-to-end (inserir um Report manual no banco → ver o feed e o ConsensusWidget; depois testar um PDF real pelo WhatsApp).
- Avaliar paywall do Feed (free vs pro) e filtro de corretora server-side (hoje é client-side sobre o que carregou).
