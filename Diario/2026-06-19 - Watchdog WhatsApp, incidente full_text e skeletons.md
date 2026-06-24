# 2026-06-19 — Watchdog do WhatsApp, incidente `full_text` e skeletons de loading

← [[00 - Home]] · sessão anterior: [[Diario/2026-06-18 - Worker WhatsApp em producao (Feed ao vivo)]] · ver [[Deploy - Railway (Producao)]]

> Sessão de estabilização: o worker do WhatsApp ficava **mudo do nada** (zombie socket) e o **Feed caiu inteiro em produção** por uma migration que não aplicou. Resolvidos os dois, mais um polimento de UX (skeletons + spinner nos gráficos).

## 1. Worker WhatsApp ficava surdo (zombie socket) — RESOLVIDO

**Sintoma:** o worker conectava, capturava PDFs por horas, e de repente **parava sem nenhum log** — sem `conexão caiu`, sem reconnect, sem a Railway reiniciar. Ficava vivo mas surdo, e o Feed parava em silêncio.

**Causa:** o Baileys às vezes mata o WebSocket no transporte **sem disparar** o evento `connection.update` com `connection === "close"`. Como o reconnect só vivia dentro do handler de `close`, o `connect()` nunca era chamado de novo. Gatilho provável: pressão de memória de **jornais gigantes** (The Times 49MB, O Globo 58MB, etc., ~240MB de buffers juntos).

**Fix (commit `cba38c5`):**
- **Watchdog** (`setInterval` 60s) checa `sock.ws.isOpen`/readyState e atividade (`lastSeen`). Fora de OPEN, ou ping `sendPresenceUpdate` sem resposta em 10s → `forceReconnect`.
- Reconnects unificados em `scheduleReconnect` com guard `reconnectScheduled` (sem corrida watchdog × close).
- **Guarda de tamanho** `WHATSAPP_MAX_PDF_MB` (default 20MB) pula PDFs gigantes via `document.fileLength` **antes de baixar** → mata o risco de OOM e o lixo dos jornais (maior relatório real ~3.8MB).
- `listGroups` lista 1×/processo e só sugere "copie os IDs" quando `WHATSAPP_GROUP_IDS` vazio (matou o ruído enganoso que aparecia em todo reconnect).

**Validação:** deploy novo conectou limpo (sem reparear) e rodou o dia inteiro; o watchdog pingou a cada ~5min e **nunca precisou forçar reconexão**. Segurou.

## 2. Feed caiu em produção — coluna `reports.full_text` não existia — RESOLVIDO

**Sintoma:** relatórios chegavam no worker (`enfileirado: ok`) mas **não apareciam no Feed**. Nos logs do **web**: `P2022 — The column reports.full_text does not exist in the current database`, em `findUnique` (dedup) E `findMany` (listagem do Feed). Ou seja, o Feed inteiro estava 500 pra qualquer usuário.

**Causa:** a feature do relatório completo (commit `89e837f`, migration `20260618120144_add_report_full_text`) subiu com o **código novo**, mas a migration **não aplicou no banco de produção** — provavelmente marcada como aplicada no `_prisma_migrations` sem a coluna ter sido criada (apply parcial, possivelmente no incidente de disco cheio). O `migrate deploy` do boot achava que não tinha nada pendente e pulava.

**Fix imediato (Marcos, no Postgres de prod):**
```sql
ALTER TABLE "reports" ADD COLUMN IF NOT EXISTS "full_text" TEXT;
```
Feito → Feed voltou na hora e relatórios passaram a salvar (visto nos logs: `salvo SECTOR ConfianceTec`, `salvo MACRO C6 Bank`, etc.). O lote das 13:19 (C6 Pares/Flow/Morning Call/Descontos, Tullett) foi **perdido** (PDFs no `tmp` efêmero, já descartados da fila no erro).

**Hardening (commit `4dbe398`):** o `findUnique` de dedup estava **fora do try/catch** → o `P2022` virava `unhandledRejection` repetido, que pode **matar o processo do web**. Agora está em try/catch: erro de banco vira `{ ok:false }` → quarentena `failed/`, sem derrubar o web.

> ⚠️ **PENDÊNCIA OPERACIONAL (não confirmada):** reconciliar o `_prisma_migrations` em prod. A migration não tem `IF NOT EXISTS` — se ela **não** estiver registrada como aplicada, o próximo `migrate deploy` tenta criar a coluna de novo, falha, e o `&&` no Dockerfile **bloqueia o boot do web**. Conferir:
> ```sql
> SELECT migration_name, finished_at FROM "_prisma_migrations" WHERE migration_name LIKE '%full_text%';
> ```
> 0 linhas → rodar `prisma migrate resolve --applied 20260618120144_add_report_full_text` contra prod. Detalhe em [[Deploy - Railway (Producao)]].

## 3. Skeletons + spinner central nos gráficos (commit `4d74949`)

Padronização de carregamento, **web + PWA**:
- Novo componente `Spinner` (usa `SyncIcon` girando) + `ChartLoadingOverlay` (spinner central sobre o gráfico) em cada app. `SyncIcon` adicionado ao `@gorila/ui/icons` (o PWA não tinha).
- **Gráficos** (web `stock-chart` + mini SVG `stock-price-chart`, PWA `stock-chart`): overlay com spinner central enquanto carrega/recarrega.
- **Skeleton** onde faltava: Feed web (cards na carga inicial + spinner no carregar-mais), Renda Fixa (Visão Geral) e Rankings.
- Já tinham skeleton (não mexi): Screener (web+PWA), Watchlist (web+PWA), Detalhe da ação (loading.tsx de rota web), Feed PWA.

**Pendente (opcional):** sub-abas da Renda Fixa (Histórico, Comparativo, Calculadora, Análise IA) ainda usam loading próprio; broker "Desconhecida" em alguns relatórios (extrator não acha o nome da corretora).

## Commits da sessão
- `cba38c5` — watchdog + guarda de tamanho de PDF + limpeza de log
- `89e837f` (sessão anterior) — feature relatório completo (origem do incidente)
- `4d74949` — skeletons + spinner central nos gráficos
- `4dbe398` — try/catch no dedup + watchdog silencioso
