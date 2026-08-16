# 2026-08-16 — Hardening de API, contrato de erro unificado e o resto do cache envenenado

← [[00 - Home]] · sessão anterior: [[Diario/2026-08-13 - Quality gate destravado, cron de sync e Renda Fixa no PWA]] · base: [[Levantamento Tecnico e de Produto (2026-08-15)]]

> Sessão de **dívida técnica**, atacando direto os gaps mapeados no levantamento de 15/08: rotas de mercado públicas (6.2), os três services de renda fixa não auditados (6.3 #3), formato de erro incompatível com o `apiFetch` (6.5) e dependências mortas.
>
> Fechados: **segurança das APIs públicas**, **auditoria do `cached()`**, **contrato de erro em 31 rotas**, **validação de ticker no report-processor**. O item 0.1 (migration travada) virou **runbook escrito**, não executado — o banco não estava acessível nesta máquina.
>
> **Correção ao histórico do vault:** a working tree estava **limpa** no início desta sessão. O acúmulo de 07/08 e 13–15/08 **já tinha sido commitado** (`Ajustes no pwa e cache`, `Ajuste na diferença do scoring em ambas as plataformas`, `chore: hooks do quality gate chamam pnpm gate`). O que está pendente de commit é **só esta sessão**.

---

## 1. 🔴 APIs de mercado agora exigem sessão

Era o gap #3 e #4 de segurança do levantamento: `/api/stocks`, `/api/stocks/[ticker]` e `/api/tickers` sem guard nenhum — dava para extrair a base inteira em páginas de 500, sem login.

**Auditoria antes de fechar** (a preocupação era quebrar a landing):

| Rota | Quem consome | Público? |
|---|---|---|
| `/api/stocks` | `screener`, `settings-alerts`, `compare-ticker-selector`, `app-header`, PWA via `@gorila/core/stocks` | não |
| `/api/stocks/[ticker]` | `stock/[ticker]`, PWA via `fetchStock` | não |
| `/api/tickers` | **nenhum consumidor** — só citado no `HANDOFF.md` | não |

> O **TickerTape da landing usa array hardcoded** (`components/landing/ticker-tape.tsx:11-24`), não faz fetch. E `/api/quotes` (BTC/ouro/dólar) só é consumido por `viewmodels/useAssetPrices.ts`, que **não é importado em lugar nenhum** — código morto.
>
> Conclusão: **não foi preciso criar variante pública limitada.** As três ganharam `guardSession()` direto.

O PWA continua funcionando porque o proxy catch-all repassa os cookies e todo fetch dele é client-side.

## 2. Páginas `/reports` e `/fixed-income` protegidas

Gap #5 do levantamento. Ambas renderizavam a casca sem login (os dados 401avam, mas a tela aparecia).

- `PROTECTED_PREFIXES` do `proxy.ts` ganhou `/reports` e `/fixed-income`. `/stock` já estava lá e o `matchesPrefix` já cobria o path sem ticker.
- `requirePageSession()` em `(app)/reports/page.tsx` e `(app)/stock/page.tsx`.
- `(app)/fixed-income/layout.tsx` **novo** — a page é `"use client"` e não pode chamar guard de servidor, então o guard vive no layout do segmento.

### 🐛 Bug encontrado de passagem: sessão expirada dava 500

`requireSession()` **lança** `UnauthorizedError`. Numa page isso vira tela de erro, não redirect. Com cookie ausente o middleware pega antes — mas com **cookie presente e token expirado** a página estourava 500. Já valia para `/stock/[ticker]`, `/compare` e `/settings`.

**Correção:** novo `requirePageSession()` em `lib/auth-session.ts`, que usa `redirect("/login?next=…")`. O `requireSession()` cru ficou intacto porque o `api-guards.ts` depende do `throw` para traduzir em 401/403. As 8 pages/layouts do grupo `(app)` migraram.

Para o `next=` funcionar o server component precisa saber o path — o `proxy.ts` agora injeta um header `x-ga-pathname` via `NextResponse.next({ request: { headers } })` (constante isolada em `lib/pathname-header.ts`, para não arrastar dependência pesada pro middleware).

**Também:** `/reports` entrou em `SHARED_PREFIXES`, então mobile mantém o path ao ser redirecionado pro PWA. `/fixed-income` **não** entrou — no PWA a rota chama `/renda-fixa` e preservar o path daria 404.

## 3. ✅ Cache envenenado — os outros três services auditados

A pendência aberta em 13/08. Além dos três previstos, **`history.service.ts` tinha o mesmo bug** e foi corrigido junto.

| Service | O que era cacheado como dado bom | Predicado aplicado |
|---|---|---|
| `usa.service.ts` | `[]` sem `FRED_API_KEY`, **ou lista parcial** — 1h | `rates.length === FRED_SERIES.length` |
| `international.service.ts` | idem, 12 séries — 1h | `rates.length === SERIES.length` |
| `fx.service.ts` | `EMPTY` (todas as taxas = 0) no catch, e os `\|\| 0` por moeda — 5min | toda taxa `> 0` |
| `history.service.ts` | `{ points: [] }` sem key ou com `!res.ok` — **6h** | `series.points.length > 0` |

> **O bug não era só o fallback total.** As três primeiras montam a resposta com `continue` dentro do loop, então uma **resposta parcial** — ex.: 3 das 7 Treasuries porque o FRED deu timeout no meio — era gravada como completa. Os predicados exigem o conjunto inteiro, igual ao `isComplete` do `brazil.service.ts`.

Verificado que não existe nenhum `setCached`/`getCached` chamado direto fora do `cache.ts` — todo caching passa pelo `cached()`.

## 4. Contrato de erro unificado — 31 rotas

O item de 6.5: `apiFetch` do core espera `{ error: { code, message } }`, as rotas devolviam `{ error: "codigo", message }`. Por isso o PWA mostrava mensagem genérica em toda falha.

> **Descoberta que mudou o plano:** o helper já existia. `jsonError(code, message, status, extra?)` em `lib/http.ts`, e **as 13 rotas de `auth` já emitiam o formato certo**. Criar um `lib/api-error.ts` novo seria duplicar. Estendi o existente.

- `ApiErrorCode` passou de 7 para **27 códigos** (union fechada — código novo tem que ser declarado).
- `api-guards.ts` convertido: isso sozinho arrumou **65 call sites** de guard de uma vez.
- **31 arquivos de rota, 97 chamadas de `jsonError`.** Restam zero `NextResponse.json({ error: … }, { status: 4xx })` no `app/api`.
- `"validation"` normalizado para `"validation_error"` (18 sites), alinhando com o que o `jsonValidationError` já emitia. Os códigos de domínio (`invalid_period`, `already_pro`, `cooldown`…) foram mantidos — a tarefa era o envelope, não renomear semântica.
- O `extra` do `jsonError` espalha **dentro** do objeto `error`, então `plan`, `limit`, `feature` e `fetchedAt` foram preservados.

**Consumers do formato antigo, atualizados:**

| Arquivo | Antes | Agora |
|---|---|---|
| `services/alerts/index.ts` | `body.error` como string × 3 | helper `alertError()` lendo `body.error.code` / `.message` |
| `services/watchlist/index.ts` | idem × 2 | helper `watchlistError()` |
| `components/ai-analysis.tsx` | `data.error` como texto | `data?.error?.message` |

`create-alert-button.tsx` e `settings-alerts.tsx` **não precisaram mudar** — leem `err.code` do `AlertError`, que o helper agora popula a partir de `error.code`. O gate de `limit_reached`/`type_locked` que abre o upgrade-modal continua funcionando.

> Resolve também o achado de 13/08: a aba de IA da Renda Fixa no PWA agora consegue mostrar o erro real (ex.: "ANTHROPIC_API_KEY ausente") em vez de "Erro inesperado".

⚠️ **Ressalva:** `sync/route.ts` devolvia o objeto `SyncResult` cru com status 500. Virou `jsonError("internal_error", …, 500, { result })` — o detalhe está preservado em `error.result`, mas quem lesse `body.status === "error"` quebraria. Não achei consumidor (é rota admin).

## 5. Tickers-lixo — validação na origem + script de limpeza

Gap "validação de ticker frouxa" (6.5 e 2.11).

**Na origem** (`report-processor.ts`): o `persist` agora resolve o ticker contra a tabela `stocks` antes de gravar. Se não existir, o relatório é salvo como **`SECTOR` com `ticker: null`** (o `company` extraído vai para `companiesMentioned`), com `console.warn` para ficar observável. Novos `AZZAS2154`/`AUGO`/`PICS` param de entrar.

**No histórico**: `apps/web/scripts/cleanup-orphan-tickers.ts`, **dry-run por padrão**.

```bash
pnpm exec tsx scripts/cleanup-orphan-tickers.ts                    # só lista
pnpm exec tsx scripts/cleanup-orphan-tickers.ts --apply            # apaga consenso órfão + converte reports em SECTOR/null
pnpm exec tsx scripts/cleanup-orphan-tickers.ts --apply --delete-reports   # apaga os reports também
```

O default converte em vez de apagar, para espelhar a regra nova do processador e não destruir o conteúdo do relatório. O consenso órfão é sempre deletado — é dado derivado, recalculável.

> 🔴 **Não executado.** O Postgres não estava de pé nesta máquina. Rodar o dry-run antes de qualquer `--apply`.

## 6. Dependências mortas — e uma que não estava morta

| Pacote | Veredito | Ação |
|---|---|---|
| `next-auth` | 0 imports | ✅ removido |
| `@auth/prisma-adapter` | 0 imports | ✅ removido |
| `xlsx` | 0 imports | ✅ removido |
| `shadcn` | ❌ **NÃO estava morto** | removido e **restaurado** |
| `lucide-react` | ❌ **NÃO está morto** — 7 arquivos importam | mantido, pendente decisão |

### 🐛 `shadcn` não era peso morto

Removi e o eslint quebrou: `Can't resolve 'shadcn/tailwind.css'`. O `apps/web/src/app/globals.css:3` faz `@import "shadcn/tailwind.css"`. **Meu grep de imports não cobria `.css`.** Restaurado na versão original `^4.2.0`.

> Lição: varredura de dependência morta precisa incluir `.css` — `@import` de pacote é import.

### `lucide-react`: uma ilha de código morto

7 arquivos importam, todos **órfãos** (nada os importa):

`components/filter-sidebar.tsx` · `stock-hero.tsx` · `stock-table.tsx` · `indicator-card.tsx` · `ai-analysis.tsx` · `ui/checkbox.tsx` · `ui/select.tsx`

> Detalhe: `components/ai-analysis.tsx` é morto — o componente vivo é `components/app/ai-analysis-loader.tsx`. E `ui/checkbox.tsx` só é usado pelo `filter-sidebar`, que também é órfão.

**Decisão pendente:** apagar os 7 arquivos destrava a remoção do `lucide-react` e fecha a violação da regra #2 do `CLAUDE.md`. Não fiz por conta própria — apagar arquivo-fonte é maior do que "remover dependência morta".

## 7. Migration travada (0.1) — runbook, não execução

Continua sendo o bloqueador do deploy. O banco de produção não estava acessível nesta máquina, então virou **`apps/web/prisma/MIGRATIONS.md`**, com a sequência pronta.

**Análise da fila:**

1. `20260618120144_add_report_full_text` — `ALTER TABLE "reports" ADD COLUMN "full_text" TEXT`, sem guarda. A coluna já existe em prod → `42701 column already exists`. Resolução: `prisma migrate resolve --applied 20260618120144_add_report_full_text`.
2. `20260807195143_add_stock_quote` — **aplica limpo na sequência.** Prisma ordena lexicograficamente pelo nome do diretório, que aqui coincide com a ordem cronológica (`20260618120144` < `20260807195143`). Ela só cria objetos novos (tabela `stock_quotes`, índice, unique e FK para `stocks`), não toca em `reports` — não há dependência com a #1 além da ordem da fila.

> ⚠️ **Ressalva que o levantamento não tinha:** o SQL da #2 **não usa `IF NOT EXISTS`**. Se `stock_quotes` já tiver sido criada fora do fluxo (por um `db push`), o deploy falha com `42P07` — mesmo tipo de drift, mesma resolução. Verificar com `psql "$DATABASE_URL" -c "\d stock_quotes"` **antes** do `migrate deploy`.

## Estado no fim da sessão

- `pnpm type-check` limpo nos 4 workspaces · `pnpm lint` **0 erros** (355 warnings pré-existentes).
- **`pnpm gate` → FAIL, por ambiente.** O `dependency-cruiser` exige Node `^22||^24||>=26` e esta máquina roda **v20.20.2** (o próprio projeto pede `>=22`). type-check e eslint passaram; só as 3 etapas de dependency-cruiser não rodaram. **Não é regressão de código** — reexecutar em máquina com Node 22+.
- 53 arquivos modificados, 4 novos (`prisma/MIGRATIONS.md`, `scripts/cleanup-orphan-tickers.ts`, `(app)/fixed-income/layout.tsx`, `lib/pathname-header.ts`).
- **Pendente de commit**, mas só esta sessão — a árvore estava limpa ao começar.

## Correções ao levantamento de 15/08

- `shadcn` **não** é dependência morta — `globals.css` importa `shadcn/tailwind.css`.
- `/api/tickers` não tinha nenhum consumidor no app (o levantamento só dizia que era pública).
- `viewmodels/useAssetPrices.ts` e `/api/quotes` são código morto — não estavam mapeados.
- O `CLAUDE.md` foi corrigido: a afirmação *"não há framework de testes"* é falsa (vitest + 6 arquivos de teste), como o levantamento já apontava em 6.5. **Ainda não corrigi essa frase** — só o contrato de erro e a seção de guards.

## Próximo

1. **Rodar em máquina com banco:** `pnpm exec tsx scripts/cleanup-orphan-tickers.ts` (dry-run) e depois o `--apply`.
2. **0.1 em produção** — seguir `apps/web/prisma/MIGRATIONS.md`, incluindo a checagem do `stock_quotes`.
3. **Commitar.** Três sessões acumuladas.
4. **Decidir sobre os 7 órfãos do `lucide-react`** (seção 6).
5. `pnpm gate` completo em Node 22+.
6. Depois: **Rankings no PWA** (único gap de paridade) e o Tier 1 do [[Backlog - Inteligencia de Mercado (ideias do socio)]].
