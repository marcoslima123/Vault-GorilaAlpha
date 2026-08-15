# 2026-08-13 a 15 — Quality gate destravado, cron de sync e Renda Fixa no PWA

← [[00 - Home]] · sessão anterior: [[Diario/2026-08-07 - Ambiente local destravado, brapi fora e cotacao separada dos fundamentos]] · ver [[03 - Roadmap]]

> Continuação direta da sessão de 07/08. O plano acordado foi **0.1 → 0.2 → 0.3 → Tier 1**. O 0.2 (fitness functions / quality gate) já tinha sido feito por outra sessão, mas **falhava por três motivos de ambiente**, todos consertados aqui. Depois: cron de sync agendado (0.3), abertura de relatório no Feed do PWA e a **Renda Fixa completa no mobile** (5 abas). Achado do caminho: um **cache envenenado zerava o IPCA por 1 hora**, subestimando todo produto IPCA+.
>
> **O 0.1 continua pendente e continua travando o próximo deploy.** Nada foi commitado.

---

## 1. Quality gate — falhava por ambiente, não por código

O `pnpm gate` (`.claude/scripts/quality-gate.sh`) roda prisma generate, type-check dos 4 workspaces, eslint, dependency-cruiser (3 configs), auditoria do `globalEnv` do Turbo e checagem de scoring duplicado. Três falhas, nenhuma relacionada a código de feature:

**1.1 `docker/Dockerfile` nunca copiava `packages/core/package.json`.** Sem isso o pnpm não montava o workspace `@gorila/core`, e `react` e `@gorila/config/typescript` ficavam irresolúveis → type-check quebrado. Adicionados os três (`config`, `ui`, `core`) nos estágios `deps` **e** `dev`.

**1.2 `quality-gate-summary.mjs` com path só de Windows.** Usava `new URL(import.meta.url).pathname.slice(1)` — o `slice(1)` remove a barra inicial que só existe no Windows. No Linux o `ROOT` virava `/app/app` e o resumo dizia "status.tsv ausente" **sempre**. Como o CI roda em `ubuntu-latest`, **o gate nunca podia passar lá**. Trocado por `fileURLToPath`.

**1.3 `tailwindcss` não resolvível da raiz.** O `eslint.config.mjs` raiz usa `eslint-plugin-tailwindcss`, mas o `tailwindcss` só existia em `apps/web` e `apps/pwa`. Na máquina do Marcos funcionava por uma cópia hoisted velha; em instalação limpa (container, CI) não. Adicionado nas devDependencies da raiz.

> Padrão que se repete: **as três só apareciam em ambiente limpo.** O host mascarava todas.

## 2. 0.3 — sync agendado (o pendente desde junho)

Fecha o buraco descrito em [[Diario/2026-08-07 - Ambiente local destravado, brapi fora e cotacao separada dos fundamentos]]: sem cron, o preço só atualizava quando alguém abria a página de uma ação.

- Nova rota `POST /api/cron/sync-stocks`, guardada por `Authorization: Bearer $CRON_SECRET`, `maxDuration = 300`.
- Estratégia **rotativa**: seleciona os tickers com indicador mais velho que 12h (`LEFT JOIN` no `MAX(fetched_at)` de `stock_indicators`), lote padrão de 60, teto de 200. Filtro opcional por mercado (`B3 | BDR | US`).
- Resposta traz `requested, synced, failed, durationMs, market, oldestBefore, staleRemaining` — dá pra acompanhar o backlog encolhendo.
- Novo `.github/workflows/sync-stocks.yml`: cron `*/30 * * * *` + `workflow_dispatch` com inputs `limit`/`market`.
- `SYNC_BATCH_SIZE` adicionado ao `globalEnv` do `turbo.json`.

**Medido no local:** 200 ações sincronizadas em 29,5s.

## 3. Feed do PWA — dava pra ver, não dava pra abrir

O Feed **já era a home do PWA** (`(app)/page.tsx`) — eu tinha afirmado que não existia e o Marcos corrigiu. O que faltava era abrir o relatório.

- `GET /api/reports/[id]` no web, com `guardSession()`. O `fullText` só vai para plano **pro**; free recebe `fullText: null` + `fullTextLocked: true`.
- `fetchReport` + `ReportDetailDTO` em `packages/core/src/reports.ts`.
- `apps/pwa/src/components/app/report-detail.tsx` — skeleton, 404 tratado, `ProGate` linkando pra `/perfil`.
- CTA "Ver relatório completo" no `report-card.tsx`, que era o que faltava para chegar na tela.
- Ícones `FileTextIcon` e `ZapIcon` criados em `packages/ui/src/icons/` (regra do projeto: criar na pasta de ícones antes de usar).

## 4. Renda Fixa no PWA — 5 abas

O módulo existia só no web desde 09/06. Agora no mobile.

**Camada de dados:** `packages/core/src/fixed-income.ts` — tipos espelhados de `apps/web/src/types/fixed-income.ts` (o PWA não pode importar do web, e a fronteira é vigiada pelo dependency-cruiser) + fetchers para `rates`, `compare`, `calculator`, `history` e `ai-analysis`.

> Detalhe que custou uma correção: as rotas usam o parâmetro **`period`**, não `years`/`months`. Só descobri lendo as rotas.

**Telas** (`apps/pwa/src/components/app/`):

| Aba | Componente | Conteúdo |
|---|---|---|
| Visão | `fixed-income-overview.tsx` | Selic/CDI/IPCA + curva de juros (SVG, BR/US/DE) + lista de produtos |
| Calculadora | `fixed-income-calculator.tsx` | valor/taxa/prazo/tipo → líquido, bruto, IR, veredito vs CDI |
| Comparar | `fixed-income-compare.tsx` | valor + prazo → ranking de 27 produtos por valor final |
| Histórico | `fixed-income-history.tsx` | 9 séries × 4 períodos, gráfico de linha, modo "comparar países" (8 séries) |
| Análise IA | `fixed-income-ai.tsx` | valor + prazo + perfil → `POST /api/fixed-income/ai-analysis` |

**Bottom nav:** novo item `/renda-fixa` com `CoinIcon`. Com 7 itens, `w-[48px]` daria 361,6px e estouraria tela de 360px. A 44px: 7×44 + 6×2 + 2×6,8 = **333,6px**. Cabe com 26px de folga.

**Correções pedidas depois da primeira validação visual:**

1. **`<select>` nativo vazava para fora da tela.** Em PWA o dropdown é renderizado pelo sistema, fora do fluxo da página — não há CSS que o contenha. Trocado por chips com rolagem horizontal. **Não sobrou nenhum `<select>` na Renda Fixa do PWA.**
2. **Comparar não tinha input de valor** — estava fixo em R$ 100.000 hard-coded. Agora editável, com debounce de 450ms.
3. **Faltavam Histórico e Análise IA** — implementadas (tabela acima).
4. **Abas em rolagem horizontal** — o segmented control `flex-1` espremia tudo. Virou faixa rolável de chips; cabem as 5 e sobra espaço.
5. **Máscara de real** nos três campos de dinheiro — dígitos entram pela direita, como app de banco (`10000` → `R$ 100,00`). Teto de 13 dígitos para não perder precisão de `Number`. Campo vazio vira `R$ 0,00`, que cai nos guards existentes.

`SparkleIcon` criado em `packages/ui/src/icons/` para a aba de IA.

## 5. 🐛 Cache envenenado zerava o IPCA — CORRIGIDO

O achado técnico da sessão. Apareceu na validação: o card de IPCA mostrava **0,00%**.

Não era a tela. `fetchBrazilIndicators` fazia `ipca12m ?? 0` quando a API do BCB falhava, e o `cached()` gravava **esse zero no banco como se fosse valor bom, por 1 hora**:

```
fi:br:indicators | exp 2026-08-13T18:09:17Z | {"cdi":13.9,"selic":14,"ipca12m":0}
```

Consultando o BCB direto do container na mesma hora, a série 13522 respondia normal (4,44). Ou seja: **um timeout transitório virou dado persistido**.

**O impacto real não era cosmético.** O comparador calcula produtos `IPCA+` como `Math.pow((1 + ipca12m/100) * (1 + r), years)` — com IPCA zerado, **todo Tesouro IPCA+ aparecia com retorno subestimado**, e ninguém teria como notar.

**Correção — separar "não consegui buscar" de "o valor é zero":**

- `cache.ts`: `cached()` ganhou um predicado opcional `shouldCache`.
- `brazil.service.ts`: o fetcher passou a devolver os `null` crus; `isComplete()` decide se cacheia; `withFallbacks()` aplica os defaults **só na leitura**.

Linha envenenada apagada do cache local. Depois: `{"selic":14,"cdi":13.9,"ipca12m":4.44}`.

> ⚠️ **`usa.service.ts`, `international.service.ts` e `fx.service.ts` usam o mesmo `cached()`** e podem ter o mesmo padrão de fallback-cacheado. **Não auditados.**

## Achados menores

- **`.next/dev/types/routes.d.ts` corrompido** derrubou o type-check com erros de sintaxe (`TS1128`, `TS1160`). Era escrita parcial do dev server — a cauda do arquivo aparecia duplicada. Apagar o arquivo e reiniciar resolve. **Não é código nosso, mas confunde:** o erro aponta para um arquivo gerado.
- **Colisão de nome no `@gorila/core`:** `stocks.ts` já exportava `AiAnalysisResponse`. O `index.ts` faz `export *` de tudo, então o type-check acusou `TS2308`. Renomeado para `FiAnalysisResponse`.
- **Formato de erro incompatível:** `apiFetch` espera `{ error: { code, message } }`, mas as rotas de fixed-income devolvem `{ error: "ai_failed", message }`. Por isso a aba IA mostra mensagem genérica em vez da real (ex.: "ANTHROPIC_API_KEY ausente"). Consistente com o resto do PWA, mas vale padronizar.
- **Histórico com cache frio é lento:** "comparar países" dispara 8 chamadas ao FRED. A primeira tentativa voltou vazia; depois de aquecer, 0,33s e então 0,04s. A tela degrada para "Sem histórico disponível" em vez de quebrar, mas o primeiro acesso do dia dá impressão errada.
- **Turbopack não pega rotas novas** — arquivo de rota novo exige `docker compose restart app`. Já é padrão recorrente.
- **`cut -d= -f2` trunca o `CRON_SECRET`** (contém `=`). Usar `sed 's/^CRON_SECRET=//'`.

## Estado no fim da sessão

- Local no ar: web :3000, PWA :3001, Postgres :5433.
- **`pnpm gate` → PASS** (0 erros, 390 warnings pré-existentes).
- Validado pelo proxy do PWA com token real: rates (27 taxas, selic 14 / cdi 13,9 / ipca 4,44), compare (27 produtos), calculator (série de 12 pontos), history (62 pontos série única; 8 séries no modo países), ai-analysis (análise real do Claude citando CDB 120% CDI e LCI 95% CDI).
- **Working tree sujo, nada commitado** — agora acumula o de 07/08 **mais** tudo desta sessão.

## Próximo

1. **0.1 — `prisma migrate resolve --applied 20260618120144_add_report_full_text` em prod.** Continua travando o próximo deploy, e agora com `add_stock_quote` na fila atrás.
2. **Commitar.** A working tree carrega duas sessões de trabalho.
3. **Rankings no PWA** — única tela que existe no web e não no mobile. A API já está pronta, é wiring.
4. Auditar os outros três services de renda fixa pelo mesmo bug de cache envenenado.
5. Depois disso, **Tier 1**: o [[Backlog - Inteligencia de Mercado (ideias do socio)]]. O Radar de Volume Anômalo é o mais barato — `StockPriceHistory.volume` e `StockQuote.volume` já existem no banco, não precisa de fonte nova.
