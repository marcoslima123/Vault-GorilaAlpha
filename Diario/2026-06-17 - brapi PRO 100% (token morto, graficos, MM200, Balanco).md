# 2026-06-17 — brapi PRO como fonte única de B3/BDR (token morto, gráficos, MM200, Balanço)

← [[00 - Home]] · sessão anterior: [[Diario/2026-06-17 - PWA em producao, brapi PRO e incidente de disco]] · ver [[Deploy - Railway (Producao)]]

> Sessão de caça a bugs de dados em produção. Vários sintomas (gráfico curto, intraday quebrado, Balanço vazio, MM200 sumida) tinham **uma causa-raiz comum: o token brapi PRO em prod estava MORTO**. Resolvido + endurecimento do código pra brapi PRO ser a única fonte de B3/BDR, removendo o Yahoo desse caminho.

## Sintomas relatados (todos em produção)
1. **PETR4/ITUB4**: 3M/6M/1A/5A mostravam o **mesmo histórico** (só ~3 meses).
2. **ABEV3**: gráfico **1D/5D não renderizava**.
3. **Balanço & DRE vazio**.
4. **5A mostrando só 3M** (mesmo depois do 1º fix) — cache do banco mascarando.
5. **"Comportamento Histórico na MM200" não aparecia**.
6. **ITUB4 (banco)** sem Caixa/Lucro/Ativo Total no Balanço.

## Causa-raiz (uma só pra 1, 2 e 3)
O serviço **web em prod estava com um token brapi INVÁLIDO/INATIVO** (`tMihVTBkEq63pUdgNkcuaf`). A Railway "insistia" em mostrar esse token antigo (não estava no código — era só a variável da dashboard; resolvido removendo + redeploy + adicionando o novo + redeploy). Token PRO válido atual: **`rpeGNpgbvm9Ugax8bDPedT`**.

Sem token efetivo:
- **Daily** caía no fallback `brapi 3mo` → 3M/6M/1A/5A iguais (a brapi só gating: PETR4/ITUB4 ela libera free, ABEV3 não → por isso ABEV3 era a que mais quebrava).
- **Intraday** (1D/5D) dava erro "Token não fornecido" pros tickers gated (ABEV3).
- **Fundamentos**: `fetchProQuotes` retornava vazio sem token → `getStock` caía no caminho FREE (`normalizeResult`) que **zerava todos os fundamentos** → Balanço vazio + refresh ao vivo sobrescrevia os dados bons com nulos.

## O que foi feito (código)
- **`brapi.ts` reescrito PRO-only** (`da9c29b`): removido todo o caminho FREE (`/api/quote/list`, `normalizeResult`, `fetchStockList`, cache da lista, `refreshCache`, `getCacheStats`). `getStock`/`getStocks` usam só `fetchProQuotes`. Sem token retorna vazio (não fabrica indicador all-null).
- **`chart-data.ts`** (`da9c29b` + `b149f40`): B3/BDR diário vem só de brapi `range = BRAPI_HISTORY_RANGE || "5y"`. E `getDailySeries` deixou de servir o cache do banco por 24h — agora **sempre busca o histórico real da brapi PRO**, persiste só quando o conjunto muda, e cai pro banco apenas se a brapi falhar. (Resolve o "5A mostrando 3M".)
- **Módulos extras de fundamentos** (`a9413c9`): `incomeStatementHistory`, `balanceSheetHistory`, `cashflowHistory` → mapeiam **EBIT**, **Ativo Total**, **CAPEX** (derivado = OCF − FCF, só > 0; bancos viram null) e **Margem EBIT** (alimenta o score de Qualidade).
- **Campos de banco** (`9406e8c`): `caixa` cai pro `balanceSheetHistory.cash` quando `financialData.totalCash` é null (ITUB4 37B, BBDC4 193B); `lucroLiquido` cai pro `netIncomeFromContinuingOps`. EBITDA/EBIT/CAPEX/dívida seguem null em banco (não reportam no padrão — **correto**).
- **MM200**: a tabela `technical_insights` (cache 24h) guardou `touchCount:0` calculado com histórico curto → mascarava. Fix = limpar esse cache (script).
- **Yahoo removido do B3/BDR** (`b76ab8a`): `providers/index.ts` não chama mais Yahoo pra B3/BDR (brapi PRO é a única fonte; o fallback só fazia barulho e é bloqueado na Railway). Yahoo fica **exclusivo do US**. `yahooFinance.ts`: `validation.logErrors=false` corta o spam de `FailedYahooValidationError`.
- **Scripts**: `clear-chart-cache.ts [TICKER]` (`b1a9066`/`9406e8c`) limpa `stock_price_history` **e** `technical_insights`. `seed.ts` agora aceita mercado: `tsx scripts/seed.ts B3|BDR|US`.

## Descobertas sobre as fontes
- **brapi PRO cobre B3/BDR**: preço, intraday, histórico 5y (1245 pts) e fundamentos completos. Token destrava tudo.
- **brapi PRO NÃO dá fundamentos de US**: testado (AAPL/MSFT/NVDA) → retorna preço + gráfico (1255 pts) mas P/L e financialData **vazios**. Por isso o **Yahoo continua sendo a fonte de fundamentos US** (e só roda em IP residencial — bloqueado na Railway).
- **`cleanEbitda` da brapi é lixo** (= cleanEbit no PETR4) — não usar.
- **Bancos** não têm EBITDA/EBIT/CAPEX/dívida no padrão Yahoo (passivo = depósitos). Preenchem Receita/Lucro/Ativo Total/Caixa/PL; o resto fica "—" (correto).

## Operacional (feito em prod)
- Env do serviço **web**: `BRAPI_TOKEN=rpeGNpgbvm9Ugax8bDPedT` + `BRAPI_HISTORY_RANGE=5y`.
- Re-sync rodado do PC contra o banco de prod (`DATABASE_PUBLIC_URL`): **B3 155/176**, **BDR 35/41** (as que falham são tickers delistados/renomeados na lista default: ARZZ3→AZZA3, VIIA3→BHIA3, RRRP3→BRAV3; CIEL3/ENBR3/SQIA3 fecharam capital).
- `clear-chart-cache.ts` rodado: 3.793 candles + 8 insights MM200 limpos.
- Validado no app: ITUB4 mostra 5A (5 anos), MM200 aparece, Balanço com Caixa/Lucro/Ativo Total. ✅

## Tempos de sync medidos
Chunk de 15 tickers ~2-6.6s (payload ~2MB com os 6 módulos). **B3 (176) ≈ 50s · BDR (41) ≈ 12s · syncAll (1429 brapi) ≈ 6-7min.** US via Yahoo é separado e lento (só do PC).

## Commits (branch `dev`)
`da9c29b` → `a9413c9` → `b1a9066` → `b149f40` → `9406e8c` → `b76ab8a`.

## Pendências / TODO
- 🔐 **Rotacionar a senha do Postgres** na Railway (a `DATABASE_PUBLIC_URL` com senha foi exposta no chat da sessão).
- **Fundamentos US** só atualizam rodando `seed.ts US` do PC (Yahoo bloqueado na Railway).
- Limpar tickers delistados de `DEFAULT_TICKERS` (yahooFinance.ts) ou seedar do universo brapi (`getAllBrazilianTickers`: 764 B3 + 665 BDR).
- (Opcional) **Religar o cache temporal de 24h do gráfico** depois que o histórico estiver 100% estável — hoje ele busca sempre da brapi (1 chamada por troca de período). É mudança de código.
- 6 BDRs não vieram da brapi (SPOT34, MAST34, MCDZ34, INTC34, AMDR34, CRMC34) — investigar código/cobertura depois.
