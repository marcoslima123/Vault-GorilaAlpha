# 2026-08-17 — Validação de ticker contra a B3, limpeza aplicada e lucide-react removido

← [[00 - Home]] · sessão anterior: [[Diario/2026-08-16 - Hardening de API, contrato de erro unificado e cache envenenado]] · base: [[Levantamento Tecnico e de Produto (2026-08-15)]]

> Sessão de execução da lista "Próximo" de 16/08. Fechados: **gate em Node 22 (PASS)**, **limpeza de tickers órfãos aplicada**, **decisão do `lucide-react`** (removido, 7 órfãos apagados).
>
> 🐛 **O achado da sessão:** a validação de ticker introduzida em 16/08 estava **destruindo dado legítimo**. Validava contra a tabela `stocks` (o que sincronizamos) em vez da lista da B3 (o que existe). Foi pego **antes** de rodar o `--apply`.
>
> ✅ **E o 0.1 foi executado.** Marcos passou a URL do banco de prod no fim da sessão e o `migrate resolve` do `full_text` rodou. **O deploy está destravado** — ver seção 5.

---

## 1. 🐛 A validação de ticker rebaixava relatórios de empresas reais

O `cleanup-orphan-tickers.ts` em dry-run acusou **7 órfãos**:

```
GGPS3 · CPRI · PICS · AUGO · EMBJ3 · AXIA3 · GGBR3
```

`GGBR3` é **Gerdau ON** e `GGPS3` é **GPS Participações**. Não são lixo. Conferindo contra a lista de tickers do projeto:

| Ticker | Na lista da B3 (1401) | Veredito |
|---|---|---|
| GGPS3 | ✅ existe | legítimo, só não sincronizado |
| GGBR3 | ✅ existe | legítimo, só não sincronizado |
| EMBJ3 | ✅ existe | legítimo, só não sincronizado |
| AXIA3 | ✅ existe | legítimo, só não sincronizado |
| CPRI | ❌ | lixo (sem sufixo numérico) |
| PICS | ❌ | lixo |
| AUGO | ❌ | lixo |

**A causa:** tanto o script quanto o `resolveKnownTicker` do `report-processor.ts` validavam contra a tabela `stocks`. Mas `stocks` tem o que **nós sincronizamos** — 140 B3 no local, de **1401** que existem na B3. Cerca de 90% dos tickers válidos seriam tratados como inexistentes.

**O dano no fluxo de entrada era pior que no script.** Um relatório da Gerdau chegando pelo WhatsApp era salvo como `SECTOR` com `ticker: null`, **perdendo `recommendation`, `targetPrice`, `currentPrice`, `upside`, `strengths` e `risks`** — os campos que só existem no tipo `STOCK`. Silenciosamente, com um `console.warn`.

### A correção

Nova regra de validação, em três níveis:

1. está em `stocks` → válido (caminho rápido)
2. está na lista da B3 (`getAllBrazilianTickers`, 1401 tickers via brapi) → válido
3. a lista voltou vazia → **fail open**, mantém como `STOCK`

O nível 3 é o que impede a correção de virar um problema pior: sem ele, uma indisponibilidade da brapi rebaixaria todo relatório que chegasse.

### 🐛 E o mesmo cache envenenado, de novo

`getAllBrazilianTickers` cacheava `[]` por **24h** quando a brapi falhava:

```ts
if (cachedTickers && now - cacheTimestamp < CACHE_TTL) return cachedTickers
```

Array vazio é *truthy*. Com a brapi fora na primeira chamada, a lista vazia ficava gravada por um dia e **todo** relatório seria rebaixado. É exatamente o padrão corrigido no `brazil.service.ts` em 14/08 e nos outros quatro services em 16/08 — a **terceira** aparição da mesma classe de bug.

Corrigido: `getAllBrazilianTickers` não cacheia resultado vazio.

> 📌 **Padrão que vale virar regra do projeto:** *toda função de cache precisa decidir se o valor merece ser cacheado.* Já apareceu em `fixed-income/cache.ts` (5 services) e agora em `tickerList.ts`. Vale varrer o resto do código procurando `cache = await fetch...` sem predicado.

### Guard no script destrutivo

`cleanup-orphan-tickers.ts` agora usa a mesma regra (união de `stocks` + lista da B3) e **aborta com exit 1** se a lista da B3 voltar vazia — não apaga com conhecimento incompleto.

## 2. ✅ Limpeza aplicada no banco local

Com a regra corrigida, o dry-run passou de 7 para **3 órfãos** — só o lixo real.

```
Tickers válidos: 1896 (lista da B3: 1401)
report_consensus órfãos: 3 — CPRI, PICS, AUGO
reports órfãos: 3
```

Aplicado com o default (converter, não apagar):

```
report_consensus removidos: 3
reports convertidos em SECTOR com ticker null: 3
Restantes — reports: 69 · consensus: 11
```

**Nenhum relatório perdido** — os 69 seguem lá, e os 4 legítimos (Gerdau, GPS, EMBJ3, AXIA3) mantiveram tipo `STOCK` e ticker.

> 🔴 **Produção não foi tocada.** Rodar o dry-run lá antes de qualquer `--apply` — a base de prod é maior, então o conjunto de órfãos é diferente.

## 3. ✅ Quality gate em Node 22 — PASS

A pendência de 16/08 (a máquina de lá rodava Node v20.20.2 e o `dependency-cruiser` exige `^22||^24||>=26`). Rodado no container, que tem **Node v22.23.2**:

```
prisma generate    OK
type-check         OK
eslint             OK   0 erro(s), 313 warning(s)
dependency-cruiser OK   0 nova(s), 2 congelada(s) no baseline
turbo globalEnv    OK
scoring unico      OK
RESULTADO: PASS
```

**Confirma que o trabalho de 16/08 não tinha regressão** — o FAIL de lá era só de ambiente.

## 4. ✅ `lucide-react` removido — decisão tomada

Os 7 arquivos foram confirmados órfãos (nenhum import em nenhum arquivo do repo):

`components/filter-sidebar.tsx` · `stock-hero.tsx` · `stock-table.tsx` · `indicator-card.tsx` · `ai-analysis.tsx` · `ui/checkbox.tsx` · `ui/select.tsx`

Detalhes que confirmaram: `ui/checkbox` só era importado pelo `filter-sidebar`, que também é órfão; o componente de IA vivo é `components/app/ai-analysis-loader.tsx`, não `components/ai-analysis.tsx`.

**Marcos decidiu apagar os 7 e remover a dependência.** Com isso a **violação da regra #2 do `CLAUDE.md` está fechada** — não há mais nenhum ícone de biblioteca externa no projeto.

### Achado extra: `components.json` reintroduziria o lucide

Aprendendo com o caso do `shadcn` em 16/08 (o grep de imports não cobria `.css`), varri **todos** os tipos de arquivo. Achei `apps/web/components.json:13`:

```json
"iconLibrary": "lucide",
```

É a config do CLI do shadcn. Não quebra build, mas **qualquer `shadcn add` futuro voltaria a puxar ícones do lucide**, reabrindo a violação. Zerado para `""`.

**Efeito colateral bom:** os warnings do eslint caíram de **355 → 313**. Os 7 órfãos carregavam 42 warnings.

### ⚠️ Incidente: o `pnpm install` derrubou o container

Depois de remover a dependência, `pnpm install --frozen-lockfile` dentro do container:

1. `ERR_PNPM_ABORTED_REMOVE_MODULES_DIR_NO_TTY` → precisa de `-e CI=true` no `docker compose exec`
2. Com `CI=true`, o pnpm imprimiu `Recreating /app/node_modules` e **apagou o node_modules**, matando o `pnpm dev`. Com `restart: unless-stopped`, o container entrou em **loop de restart** — e `docker compose exec` não funciona em container reiniciando, então não dava para terminar o install

**Saída:** `docker compose stop app` + `docker compose up -d --force-recreate --renew-anon-volumes app` — o mesmo remédio do volume anônimo stale. Restaura o `node_modules` da imagem e o dev server sobe.

> 📌 **Nota operacional:** nunca rodar `pnpm install` que mexa em `node_modules` dentro do container de dev com `restart: unless-stopped`. A recriação do diretório mata o processo e o container entra em loop antes de terminar. Fazer no host (`--lockfile-only` é seguro) e recriar o container.

## 5. ✅ 0.1 executado — deploy destravado

Marcos passou a URL do banco de produção. Diagnóstico read-only antes de qualquer escrita:

| Checagem | Resultado | Ação pelo runbook |
|---|---|---|
| `reports.full_text` | **EXISTE** (`text`) | `migrate resolve --applied` |
| Registro em `_prisma_migrations` | **ausente** | drift confirmado |
| `stock_quotes` | **NÃO EXISTE** | nada — `migrate deploy` cria |
| `finished_at NULL` | **nenhuma** | fila limpa, só faltavam registros |
| Volume | 150 reports · 680 stocks | — |

O `MIGRATIONS.md` acertou os dois cenários. Executado:

```
Migration 20260618120144_add_report_full_text marked as applied.
```

`migrate status` depois:

```
Following migration have not yet been applied:
20260807195143_add_stock_quote
```

**Não é preciso rodar `migrate deploy` à mão** — o `start:railway` é `prisma migrate deploy && next start`, então o boot do próximo deploy aplica a `add_stock_quote` e sobe. O que travava era só o drift.

> O `migrate deploy` manual foi **bloqueado pelo classificador de permissões** do Claude Code. Não fez falta, pelo motivo acima.

## 6. ✨ Nova feature: análise comparativa por IA no Comparador (Pro)

Primeira feature nova depois da faxina. Card no fim da tela de Compare, nos dois apps.

**Backend**

| Arquivo | Papel |
|---|---|
| `services/compare/ai-comparison.ts` | `compareWithAI(tickers)` — carrega `Stock` + `StockIndicator` mais recente + `ReportConsensus` de cada ticker, monta prompt comparativo, chama Claude |
| `app/api/compare/ai-analysis/route.ts` | `POST` · `guardSession` + gate `aiAnalysis` · zod 2–4 tickers · valida existência na tabela `stocks` |
| `prisma` model `CompareAnalysis` | cache com unique em `tickersKey`, TTL 7 dias, migration `20260817160554_add_compare_analysis` |
| `packages/core/src/compare.ts` | `fetchCompareAiAnalysis` para o PWA |

**A chave de cache normaliza ordem e caixa.** `tickersKeyFor` faz `uppercase → dedup → sort → join(",")`, então `["vale3","itub4","petr4"]` e `["PETR4","VALE3","ITUB4"]` batem na **mesma** entrada. Sem isso, 3 ativos gerariam até 6 caches idênticos e 6× o custo de token.

Além do TTL, o cache invalida por **mudança de modelo**: `cached.model === AI_MODEL` faz parte do teste de frescor. Se o `AI_MODEL` subir, a análise é regerada em vez de servir conteúdo do modelo antigo — a spec pedia unique só em `tickersKey`, e essa checagem preserva isso sem servir dado velho.

**Prompt** — system com `cache_control: { type: "ephemeral" }`, tom descritivo obrigatório, lista de termos proibidos ("compre", "venda", "prefira", "melhor escolha"…), sem markdown.

**Frontend**

- Web: `components/app/compare-ai-analysis.tsx`. Free → parágrafos de exemplo com `blur-sm` + overlay "DISPONÍVEL NO PRO" com CTA (mesmo padrão do `RadarPaywallOverlay`). Pro → botão "Gerar análise comparativa" → skeleton → análise.
- PWA: mesmo componente com tokens CSS (`var(--surface)`, `var(--primary)`), botões de 48px, CTA para `/perfil`.
- Disclaimer obrigatório abaixo da análise nos dois.

**Verificado de ponta a ponta:**

| Cenário | Resultado |
|---|---|
| Sem sessão | `401 unauthorized` |
| Free | `403 plan_locked` com `plan` e `feature: "compare-ai"` |
| 1 ticker / 5 tickers | `422 validation_error` com `issues` |
| Ticker inexistente | `404 not_found` com `missing` |
| Pro, 3 ativos | análise real, **10,5s** |
| 2ª chamada | **0,06s** (cache) |
| Ordem diferente + minúsculas | mesmo cache, mesma resposta |
| Estrutura | 4 parágrafos + bloco com 1 frase por empresa, sem markdown |
| Termos prescritivos | **nenhum** |
| Pelo proxy do PWA | funciona, e o Free recebe a **mensagem real** (não a genérica) |

> O último item é o contrato de erro unificado de 16/08 pagando dividendo: o PWA mostra "Análise comparativa por IA disponível apenas no plano Pro." em vez de "Erro inesperado".

## Estado no fim da sessão

- Local no ar: web :3000, PWA :3001, Postgres :5433. `/compare` 200 nos dois apps e nos dois planos.
- **`pnpm gate` → PASS** (0 erros, 321 warnings — 313 depois da faxina, +8 dos arquivos novos da feature).
- Banco local limpo de tickers órfãos.
- Faxina **commitada** em `e27ff353 fix: valida ticker contra a lista da B3 e remove lucide-react`.
- Feature de análise comparativa **pendente de commit** (7 arquivos novos/alterados + migration).
- Produção: drift resolvido, `add_stock_quote` aplica no próximo boot.

## Correções ao histórico

- A working tree estava **limpa** no início desta sessão: o trabalho de 16/08 foi commitado em `e84c4e81 feat: protege APIs de mercado, unifica contrato de erro e audita cache de renda fixa`. O item "Commitar — três sessões acumuladas" **já estava resolvido**.
- `lucide-react` **não** era código morto no sentido de "ninguém importa" — 7 arquivos importavam. Eram os *arquivos* que estavam mortos. A distinção importa: remover só a dependência quebraria o eslint, igual ao shadcn.

## Próximo

1. **Commitar** esta sessão (13 arquivos alterados, 7 deletados).
2. **Deployar.** O bloqueador acabou. No primeiro boot a `add_stock_quote` aplica sozinha. Junto: configurar os secrets `APP_URL` e `CRON_SECRET` no GitHub para o `sync-stocks.yml` — sem eles o preço em prod continua defasado.
3. **Rotacionar as credenciais expostas** — a `RESEND_API_KEY` commitada no `.env.example:21` e a URL do Postgres de prod, que já circulou em texto plano três vezes.
4. **Dry-run da limpeza de órfãos em produção** antes de qualquer `--apply` lá (base maior, conjunto de órfãos diferente).
5. **Varrer o resto do código** procurando cache sem predicado (3ª ocorrência da mesma classe de bug).
6. **Rankings no PWA** — único gap de paridade. API pronta, é wiring.
7. Depois: Tier 1 do [[Backlog - Inteligencia de Mercado (ideias do socio)]] — começando pelo Radar de Volume Anômalo, que não precisa de fonte de dados nova.
