# 2026-08-07 — Ambiente local destravado, brapi fora do ar e cotação separada dos fundamentos

← [[00 - Home]] · sessão anterior: [[Diario/2026-06-19 - Watchdog WhatsApp, incidente full_text e skeletons]] · ver [[Deploy - Railway (Producao)]]

> Retomada depois de ~7 semanas parado. A sessão começou como "sobe a API no Docker" e virou uma cascata: disco cheio → daemon travado → `node_modules` congelado em maio → migration pendente → descoberta de que a **brapi estava fora do ar** e que **produção servia preços de 15 dias atrás em silêncio**. Terminou com as 3 mudanças de arquitetura de cotação implementadas. **Nada foi commitado nem deployado.**

> ⚠️ **Lacuna de registro:** o diário pulou de 2026-06-19 para cá. No meio houve só 1 commit (`9519646b`, 05/08) e a captura do [[Backlog - Inteligencia de Mercado (ideias do socio)]] em 07/07.

---

## 1. Subir o ambiente local — 3 bloqueios em série

**1.1 Disco C: em 0 bytes.** O primeiro `docker compose build` passou por todos os passos e morreu no `exporting layers` com `read-only file system`. Marcos liberou ~34 GB.

**1.2 Daemon do Docker travado.** Depois do disco encher, nem `docker ps` respondia. `Docker Desktop.exe -Shutdown` não funcionou; foi preciso matar os processos à força + `wsl --shutdown` e subir de novo.

**1.3 Build context de ~2,5 GB.** O `.dockerignore` não excluía `.turbo/` (2,1 GB de cache do Turborepo) nem `apps/web/tmp/` (350 MB). Adicionados: `.turbo`, `**/.turbo`, `.pnpm-store`, `**/tmp`, `**/whatsapp-session`, `**/*.tsbuildinfo`.

**Resultado:** `exporting layers` caiu de **150,1s → 2,7s**.

## 2. Renda Fixa "sumida" no local — volume anônimo stale — RESOLVIDO

**Sintoma:** `/fixed-income` funcionava em produção mas no local renderizava só a casca (título + 5 abas) sem nenhum dado. Mesmo código.

**Descartado no caminho:** banco (27 taxas idênticas nos dois), migrations das tabelas, paywall (3 usuários locais, todos `pro/active`), rota (200 sempre), e acesso às fontes (BCB, FRED, AwesomeAPI todos 200 de dentro do container).

**Causa:** o `docker-compose.override.yml` monta `.:/app` (código do host, atual) mas usa **volumes anônimos** para `node_modules`. Esses volumes sobrevivem a `up -d` e a `restart`, e **têm precedência sobre o conteúdo da imagem no mount point** — então rebuildar não resolvia. Dentro dele:

- **Prisma Client gerado em 2026-05-27**, anterior ao módulo de Renda Fixa. Os models `fixedIncomeRate`, `fixedIncomeCache`, `report`, `reportConsensus` tinham **zero ocorrências** no client (`user` e `stockIndicator` estavam lá, por isso login e Screener funcionavam).
- **`unpdf` faltando** — dependência do processador de PDF do Feed.

A cadeia até a tela: `db.fixedIncomeCache.findUnique` → `Cannot read properties of undefined` → rota 500 → o `fetch` do componente cai em `r.ok ? r.json() : null` → `data` null → renderiza `"Não foi possível carregar as taxas."` (`page.tsx:79`).

**Fix:** `docker compose up -d --force-recreate --renew-anon-volumes app`. Um `prisma generate` avulso conserta só o Prisma — foi a primeira tentativa, e aí o `unpdf` faltando derrubou o `/api/ping` para 404.

> Isso também estava quebrando **Feed e Watchlist** no local, pelo mesmo motivo.

## 3. Erro 500 em `/api/reports/consensus/*` no local — RESOLVIDO

Mesmo incidente do `full_text` de junho, agora do lado local. Com o Prisma Client já correto, o model `Report` passou a incluir `full_text`; o `findFirst` seleciona todas as colunas; o banco local não tinha a coluna → `P2022`. Antes de regenerar o client isso nem aparecia (o client de maio não conhecia o model).

`prisma migrate status` confirmou a migration `20260618120144_add_report_full_text` pendente. Aplicada com `migrate deploy`. Endpoint voltou a 401 sem sessão (era 500), idêntico a produção.

## 4. ⚠️ PRODUÇÃO — a pendência do `full_text` está CONFIRMADA e vai travar o próximo deploy

A dúvida aberta desde 2026-06-19 foi resolvida com acesso ao banco de prod:

```sql
SELECT migration_name, finished_at FROM "_prisma_migrations" WHERE migration_name LIKE '%full_text%';
-- 0 linhas
```

**A migration NÃO está registrada.** E o SQL dela não tem guarda:

```sql
ALTER TABLE "reports" ADD COLUMN     "full_text" TEXT;
```

Como a coluna já existe (criada na mão em junho), o próximo `prisma migrate deploy` no boot vai falhar com `column already exists`, e o `&&` do `start:railway` (`prisma migrate deploy && next start`) **impede o `next start`**. O serviço web não sobe.

**Ainda não corrigido** — escrita em prod ficou pendente de aprovação. O conserto é uma linha, sem tocar em dado:

```
prisma migrate resolve --applied 20260618120144_add_report_full_text
```

> Agora há **uma migration nova na fila atrás dela** (`add_stock_quote`, item 6), o que torna o boot travado ainda mais provável.

## 5. 🔴 brapi fora do ar e produção com preço de 15 dias

Investigando a frescura do preço, a brapi não respondia:

| Teste | Resultado |
|---|---|
| `brapi.dev/` (raiz) | HTTP 200 em 0,56s |
| `/api/quote/PETR4` sem token | timeout 25s |
| `/api/quote/PETR4` token falso | timeout 25s |
| `/api/quote/PETR4` token real | timeout 25s |
| UA de browser / header Bearer / HTTP1.1 | timeout 30s |
| Twelve Data (controle) | responde em 0,49s |

Site no ar, API não. Não era token (confere com o PRO do vault), nem Docker, nem rede. **A brapi voltou ao normal ~1h depois** — foi indisponibilidade transitória do provedor, mas longa.

**O impacto revelado em produção:**

```
indicador mais recente:  2026-07-23 22:53  (352h atrás)
B3/BDR mais recente:     2026-07-23 22:53
US mais recente:         2026-06-17 20:12
último sync log:         2026-06-17 20:15
```

Produção estava servindo **preços de 15 dias** (B3/BDR) e de ~7 semanas (US), **em silêncio**: quando o provedor falha, a rota cai no ramo `cache-stale` e devolve o preço velho com um campo `warning` que a UI nunca mostrou. Somado a isso, **não existe sync agendado** (pendente desde junho) — o único caminho de atualização é alguém abrir a página de uma ação com o cache vencido.

## 6. As 3 mudanças de arquitetura de cotação — IMPLEMENTADAS (não commitadas)

### 6.1 Preço separado dos fundamentos

Antes, atualizar a cotação refazia a chamada pesada da brapi (`fundamental=true&dividends=true` + 6 módulos) e o preço herdava o cache de 15min dos fundamentos.

- Novo caminho leve: `fetchBrapiQuotes` (B3/BDR) e `fetchTwelveDataQuotes` (US) em `/api/quote/{tickers}` **sem `fundamental` nem `modules`**, timeout 8s.
- Novo serviço `apps/web/src/services/quotes/` com TTL de **60s** (`QUOTE_TTL_SECONDS`), gravando em novo model **`StockQuote`** (uma linha por ação, upsert).
- TTL de fundamentos: 15min → **12h** (`FUNDAMENTALS_TTL_HOURS`) — mudam por trimestre.
- Fallback: provedor fora → devolve a última cotação conhecida com `stale: true` e o `fetchedAt` **real** (não finge frescor).

**Medido:** cache quente 55ms · refetch 227ms · ticker novo 159ms. Cotação com ~100s de idade (`priceQuotedAt` vs `priceUpdatedAt`), contra os 15min anteriores.

### 6.2 Fim do churn de `stock_indicators`

`saveStockData` fazia `create` a cada refresh. Prod tinha **1978 linhas para 680 ações** (AAPL34 com 11). Agora é update-or-create: 2 syncs seguidos de PETR4 mantiveram 1 linha.

> Escolha deliberada: **não** usar índice único em `stockId`, porque a migration teria que apagar ~1300 linhas históricas em prod. Assim nada é destruído e o crescimento para. Limpeza das linhas antigas fica como script opcional.

### 6.3 Idade do dado visível na tela

Novo `PriceFreshness` no `StockHeaderCard`, 3 estados:
- fresco → `Cotação agora · atraso de 15min da bolsa`
- velho → `⚠ Cotação de 15d atrás` (âmbar, `AlertTriangleIcon` da pasta de ícones do projeto)
- sem dado → `Cotação indisponível`

A API passou a expor `priceUpdatedAt`, `priceQuotedAt`, `priceSource`, `priceStale`, `fundamentalsUpdatedAt`.

**Teto de realidade:** B3 via brapi tem 15min de atraso por regra da bolsa (o código já carimbava `exchangeDataDelayedBy: 15`). Tempo real de verdade exigiria feed licenciado da B3. O alvo alcançável é "15min da bolsa e mais nada nosso".

## 7. Commit `9519646b` desfeito (mensagem não batia com o conteúdo)

A mensagem dizia `feat(challenges): explica empates de Constância no card de Premiação`, mas não existe "challenges", "Constância" nem "Premiação" em lugar nenhum do código. O conteúdo real: extração do `ScoreRing` para componente próprio com tooltip dos 5 pilares, e atualização de `DEFAULT_TICKERS.B3` (RRRP3→BRAV3, ARZZ3/SOMA3→AZZA3, remove ENBR3/SQIA3/SEQL3/VIIA3).

Não tinha sido pushado (`dev` estava 1 à frente de `origin/dev`), então `git reset --soft HEAD~1` — commit desfeito, alterações preservadas e staged. **Falta recommitar com mensagem correta.**

## Achados menores

- **Tickers lixo no Feed:** prod tem `AZZAS2154` em `report_consensus`; local tem `AUGO`, `PICS`, `CPRI`. O extrator cria entradas a partir de menções soltas. Também há linhas de consenso com `buy+neutral+sell = 0` (prod: ADSK, PBGO) que nunca renderizam.
- **`HANDOFF.md` da raiz do repo está tóxico** — congelado em 2026-04-10, descreve estrutura pré-monorepo (`src/` na raiz, Playwright, Node 20, Yahoo primário). Contradiz a realidade. Vale apagar ou trocar por ponteiro pro vault.
- **ESLint quebrado no projeto:** `eslint.config.mjs` na raiz importa `eslint`, que não está no `node_modules` raiz (só `turbo` está no `package.json` da raiz). Falha em qualquer arquivo. Pré-existente.
- **Credencial de prod exposta:** a `DATABASE_PUBLIC_URL` do Postgres circulou em texto plano no chat. Considerar rotacionar na Railway.

## Estado no fim da sessão

- Local no ar: web :3000, PWA :3001, Postgres :5433. `tsc --noEmit` limpo.
- Migration `20260807195143_add_stock_quote` criada e aplicada **só no local** (aditiva, cria `stock_quotes`).
- **Working tree sujo, nada commitado:** as 3 mudanças de cotação + o ScoreRing staged do reset + o `.dockerignore`.

## Próximo

1. Recommitar o ScoreRing com mensagem correta (ou separar em 2 commits).
2. Commitar as mudanças de cotação.
3. **Antes de qualquer deploy:** rodar o `migrate resolve` do `full_text` em prod, senão o web não sobe.
4. Considerar o **sync agendado** (pendente desde junho) — sem ele o preço só atualiza por visita.
