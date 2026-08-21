# 2026-08-18 — Justificativa por IA no Ranking e uma arquitetura para o cache

← [[00 - Home]] · sessão anterior: [[Diario/2026-08-17 - Validacao de ticker pela B3, limpeza aplicada e lucide removido]] · base: [[Levantamento Tecnico e de Produto (2026-08-15)]]

> Uma entrega de produto (justificativa por IA no pódio dos Rankings) e a **correção de raiz do problema de cache** — que já tinha aparecido 4 vezes em 4 sessões e virou reclamação do Marcos: *"estamos tendo mt problema com cache, credo, toda hora isso aparece"*.
>
> Ele estava certo. Não era descuido: era o formato do código. Esta sessão troca o formato.

---

## 1. Justificativa por IA no pódio dos Rankings (desktop, Pro)

**Só no desktop** — o PWA não tem tela de Rankings, que é o gap de paridade aberto.

| Camada | Arquivo |
|---|---|
| Service | `services/rankings/ai-insights.ts` — `explainPodium(criterion, topStocks)` |
| Rota | campo `aiInsights` em `GET /api/rankings?insights=true` |
| Cache | model `RankingInsight`, unique `[criterion, tickersKey]`, TTL **24h** |
| Front | seção "Por que estão no pódio" em `rankings-podium.tsx` |

Saída real, critério Maior Dividend Yield:

> **#1 SYNE3 (68%)** — "O critério principal é o dividend yield de 68%, posicionando a ação em primeiro lugar, mas a sustentabilidade desse rendimento é questionável considerando a dívida líquida sobre EBITDA em 1.99x e o crescimento de lucro modesto de 9,75%. (…) o payout não foi divulgado, o que impede confirmar se a distribuição está alinhada com o lucro gerado."

Três frases: número do critério, indicador de sustentabilidade, ponto de atenção.

**Decisões de implementação**

- **TTL de 24h, não 7 dias** — ranking muda mais rápido que análise de ação individual.
- **Geração só com `?insights=true`** — a resposta padrão de `/api/rankings` continua idêntica. O front carrega o ranking primeiro e busca a justificativa depois, para não atrasar o primeiro paint.
- **Só grava se cobrir todas as 3 ações** (`coversEveryStock`) — aplicando a lição do cache antes de ela virar arquitetura.
- **Renderizado abaixo do pódio, não dentro de cada card.** A spec pedia "abaixo de cada card", mas os cards têm 240/300px e formam uma escada com `items-end`; três frases dentro de cada um destruiriam o pódio. Virou uma linha por ação, com o badge de posição na mesma cor do card.

**Verificado:** Free com `insights=true` não gera · Pro sem o param não gera · 1ª chamada 5,1s → 2ª 0,18s · critério diferente gera entrada própria · 3/3 ações cobertas · zero termos prescritivos.

> ⚠️ **Não conferido visualmente.** A página de Rankings é `"use client"` inteira — o HTML servido é só skeleton, e o pódio (o que já existia, inclusive) só aparece após o fetch no navegador. `curl` não alcança.

### Detalhe que custou uma rodada

A página já estava **exatamente no limite** de `complexity` do eslint (15). Somar a lógica de carregamento estourou para 18–20. Extraí para um hook `usePodiumInsights` no mesmo arquivo, que absorveu também o `data?.plan ?? "free"` que a página repetia. Voltou a 0 erros.

## 2. A quarta ocorrência do cache envenenado

Antes de propor arquitetura, varri o código procurando o padrão. Achei a quarta:

**`technical-insights`** — com menos de 200 fechamentos diários, `computeMM(closes, 200)` devolve `null` em toda a série, nenhum toque na MM200 é detectado, e o resultado sai com `touchCount: 0`. Isso era gravado **por 24h** como se fosse "o preço nunca encostou na média", quando na verdade era "não há dado suficiente para calcular a média".

Corrigido em `155ab910`: devolve `unavailable` sem gravar, como já fazia no `catch`.

`quotes` foi auditado e estava OK (`if (!quote) continue`).

## 3. A arquitetura de cache — o item central da sessão

### Por que quatro bugs iguais em quatro sessões

O formato era:

```ts
resultado = await buscar()
gravar(resultado)
```

No ponto onde se grava, um fallback — `?? 0`, lista vazia, resposta parcial montada com `continue` dentro do loop — é **indistinguível de um valor real**. O código não tinha onde expressar "isso aqui é degradado". Então cada função de cache nova recomeçava o erro do zero.

Já existia meia-solução (o `shouldCache` de 14/08), mas era **opcional** — só protegia quem lembrasse de usar.

### A troca: obrigar a classificar

`apps/web/src/lib/cache.ts`, novo. O fetcher passa a **retornar** um de três estados, em vez de `T` cru:

```ts
type FetchOutcome<T> =
  | { status: "fresh";    value: T; coverage? }   // grava
  | { status: "degraded"; value: T; reason }      // devolve, NÃO grava
  | { status: "failed";   reason }                // cai no último valor bom
```

Não é convenção, é o tipo. **Gravar sem classificar não compila.**

Helper `byCoverage(value, got, expected, reason)`: `fresh` se `got === expected`, `degraded` se parcial, `failed` se zero. É o que mata o caso da resposta parcial.

### Nova tabela `cache_entries`

| Coluna | Para quê |
|---|---|
| `value` | o dado (igual a antes) |
| `status` | fresh · degraded · failed |
| `source` | bcb · fred · brapi · twelvedata · awesomeapi |
| `coverageGot` / `coverageExpected` | 3/3, 2/3, 7/7 — torna a falha parcial visível |
| `fetchedAt` | quando veio da fonte de verdade |
| `expiresAt` | quando o valor deixa de ser fresco |
| `retryAfter` | backoff de 60s numa fonte que falha |

O `retryAfter` resolve o trade-off que apareceu no desenho: não cachear a falha significaria bater na fonte a cada request. Agora tenta de novo em 60s, sem gravar a falha e sem perder o último valor bom.

### O que foi migrado

| Cache | Fonte | TTL | Cobertura esperada |
|---|---|---|---|
| `fi:br:indicators` | BCB | 1h | 3/3 séries |
| `fi:us:treasuries` | FRED | 1h | 7/7 |
| `fi:intl:bonds` | FRED | 1h | 12/12 |
| `fi:fx` | AwesomeAPI | 5min | 5/5 moedas |
| `fi:hist:{série}:{período}` | FRED / BCB | 6h | tudo-ou-nada |
| Lista de tickers B3 | brapi | 24h | memória, mín. 100 |

`quotes` e `technical_insights` **não** mudaram de storage — têm colunas de domínio e índices que a query do sync rotativo usa. Ganharam só log de cobertura.

`ai_analyses`, `compare_analyses` e `ranking_insights` ficaram como estão: falha do Claude é tudo-ou-nada e já tratada.

### Verificação

Simulei os três cenários contra a implementação real:

| Situação | Antes | Agora |
|---|---|---|
| BCB devolve 2 de 3 séries | gravava `ipca=0` por 1h | log `degraded [2/3]`, **mantém `ipca=4.44`**, marca `stale` |
| Fonte fora, cache existe | gravava vazio | respeita backoff, serve último valor bom |
| Fonte fora, cache vazio | gravava vazio | devolve `null`, chamador aplica fallback |

> 🎯 **E aconteceu de verdade durante o teste:** o BCB deu **timeout de 12s** na série histórica da Selic. O log registrou `failed (The operation was aborted due to timeout)`, a série **não ficou cacheada vazia por 6 horas**, e na tentativa seguinte voltou com 62 pontos. Sob o código antigo aquele `{points: []}` teria ficado gravado até de tarde. Era exatamente o bug de 16/08, reproduzido por acidente e agora inofensivo.

Estado real da tabela depois dos testes:

```
fi:br:indicators    fresh  bcb         3/3
fi:fx               fresh  awesomeapi  5/5
fi:intl:bonds       fresh  fred        12/12
fi:us:treasuries    fresh  fred        7/7
fi:hist:SELIC:5A    fresh  bcb
fi:hist:US10Y:5A    fresh  fred
fi:hist:DE10Y:5A    fresh  fred
```

### Observabilidade

`cacheHealth()` exposto no `GET /api/sync` (admin) — conta entradas por status. Antes, descobrir que algo estava degradado dependia de sorte.

## 4. Aviso de dado desatualizado na tela da renda fixa

O contrato de cache já distinguia valor fresco de valor servido após falha da fonte, mas isso morria no servidor. Agora `GET /api/fixed-income/rates` devolve um campo `freshness`:

```
fi:intl:bonds     fresh  fred        stale=false  12/12
fi:br:indicators  fresh  bcb         stale=true   3/3
fi:fx             fresh  awesomeapi  stale=false  5/5
fi:us:treasuries  fresh  fred        stale=false  7/7
```

Web e PWA mostram uma faixa âmbar acima dos indicadores dizendo **qual fonte** está desatualizada e **de quando** é o dado. Mesmo padrão que o `StockHeaderCard` já usava para a cotação.

`freshnessFor` lê direto de `cache_entries`, então **nenhum service precisou passar metadado adiante** — evitou costurar `CachedValue` por toda a cadeia.

`AlertTriangleIcon` criado em `packages/ui/src/icons/` — só existia na pasta do web, e a regra #2 pede criar antes de usar.

**Verificado:** simulei o BCB fora (entrada vencida + backoff) e o **IPCA continuou 4,44** com só aquela entrada marcada `stale`. Removendo a entrada, volta tudo a `fresh` e o aviso some.

## 5. 🚨 O deploy subiu sem aplicar as migrations

Marcos deployou. O app respondeu 200 e o **código novo estava no ar** (rotas de 13, 17 e 18/08 todas respondendo 401). Mas as **4 tabelas novas não existiam** em produção, e a última migration registrada ainda era a de 17/08.

O `CMD` do Dockerfile roda `prisma migrate deploy && next start`. Se a migration tivesse falhado, o `&&` derrubaria o app — mas ele respondia. **A explicação que sobra é um Start Command customizado na Railway sobrescrevendo o `CMD`.**

O risco era concreto: o código no ar chamava `stock.quote` e `cache_entries`, que não existiam.

Marcos rodou o `migrate deploy` na mão (o classificador me bloqueou). As 4 aplicaram limpo. Depois: 24 tabelas, nenhuma migration com `finished_at NULL`, 680 stocks e 150 reports preservados.

🔴 **Pendente:** conferir Settings do serviço web na Railway. Sem isso o problema volta no próximo deploy. Detalhe em [[Deploy - Railway (Producao)]].

## 6. Limpeza de órfãos em produção — dry-run

```
report_consensus órfãos: 4   PBGO · TOPY11 · AZZAS2154 · BCTE39
reports órfãos: 5 (TOPY11 tem 2)
```

`AZZAS2154` e `PBGO` são lixo claro. `TOPY11` e `BCTE39` têm formato plausível mas não estão na lista atual da B3 — provavelmente deslistados.

✅ **Aplicado em produção** (modo padrão, que converte em vez de apagar):

```
report_consensus removidos: 4
reports convertidos em SECTOR com ticker null: 5
Restantes — reports: 150 · consensus: 21
```

Dry-run seguinte: **0 órfãos**. **Nenhum relatório perdido** — os 150 seguem lá (36 STOCK, 33 SECTOR, 81 MACRO).

> 🟡 **Sobrou um caso de classe diferente:** `ADSK` tem consenso com contagem zero (0 compra / 0 neutro / 0 venda, sem preço-alvo). Não é órfão — Autodesk é ticker válido — mas é linha morta que nunca renderiza nada. Precisa de regra própria: "consenso sem nenhuma recomendação parseável".

### Cron ainda não confirmado

`check-alerts.yml` e `sync-stocks.yml` usam **os mesmos dois secrets**: `APP_URL` e `CRON_SECRET`. Como o de alertas roda desde maio, os secrets provavelmente já existem — mas não dá para confirmar daqui (sem `gh` CLI, e o `CRON_SECRET` local **difere** do de produção, como deve ser).

O endpoint está deployado e guardando corretamente (401 com secret errado, não 404).

**Como confirmar em 1 minuto:** GitHub → aba **Actions** → workflow **Sync Stocks** → **Run workflow** (ambos têm `workflow_dispatch`). Dá resposta imediata em vez de esperar os 30min do schedule.

> ⚠️ **Detalhe do GitHub Actions:** workflow agendado por `schedule` só roda na **branch padrão** do repositório. Aqui `dev` é a única branch remota, então deve ser a padrão — mas se algum dia criarem `main`, os crons param sem aviso.

## Decisões do Marcos nesta sessão

| Item | Decisão |
|---|---|
| **Rotacionar credenciais** | 🟡 **Adiado** — só quando outras pessoas forem mexer no projeto. Risco aceito conscientemente enquanto o acesso é só dele. |
| **Rankings no PWA** | 🟡 **Aguardando** — vai acontecer quando ele ajustar os menus do app. Não é pendência técnica. |

## Commits da sessão

```
9d7bf4fd  feat: mostra na tela quando a renda fixa serve dado desatualizado
bf120255  refactor: contrato de cache que obriga classificar o resultado
dc8068a9  feat: justificativa por IA no podio dos Rankings (desktop, plano Pro)
155ab910  fix: nao cacheia insight de MM200 quando falta historico
```

Migrations novas: `20260817232630_add_ranking_insight` e `20260818163022_add_cache_entry`. Ambas aditivas.

## Estado no fim da sessão

- `pnpm gate` → **PASS** (0 erros, 323 warnings).
- Working tree **limpa** — tudo commitado.
- Produção: drift resolvido em 17/08. Agora há **3 migrations** na fila, todas aditivas, aplicadas no próximo boot.

## Dívidas que esta sessão criou ou deixou

1. **`fixed_income_cache` ficou órfã** — 46 linhas, ninguém mais lê. Vale um `DROP` numa limpeza com calma; é migration destrutiva, não fiz de afogadilho.
2. **`status`/`stale` não chegam à tela na renda fixa.** O `StockHeaderCard` já faz isso com `priceStale` ("⚠ Cotação de 15d atrás"); a renda fixa poderia mostrar o mesmo ao servir valor stale. É mudança de UI nos dois apps — não misturei com o refactor.
3. **Rankings no PWA** continua o único gap de paridade, e agora também não tem a justificativa por IA.

## Próximo

1. Confirmar o cron pela aba Actions do GitHub (**Run workflow** no Sync Stocks).
2. **Radar de Volume Anômalo** — primeiro do Tier 1. Não precisa de fonte nova: `StockPriceHistory.volume` e `StockQuote.volume` já estão no banco.
3. Regra para consenso com contagem zero (`ADSK` em prod).
4. `DROP` na tabela `fixed_income_cache`, que ficou órfã.
5. Rankings no PWA — quando o Marcos ajustar os menus.
6. Rotacionar credenciais — antes de dar acesso a outra pessoa.
