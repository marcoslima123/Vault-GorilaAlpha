# 2026-05-27 — Gráfico interativo (Lightweight Charts / TradingView)

← [[00 - Home]] · sessão anterior: [[Diario/2026-05-26 - Analise IA real (estrutura + fallback)]]

## Objetivo

Substituir o gráfico SVG manual (Sprint 3.5) por **Lightweight Charts v5** (open source do TradingView): pan/zoom, crosshair com tooltip, MMs sobrepostas, marcadores de toque na MM200, persistência no Postgres com cache 24h.

## Expectativa ajustada (dito ao Marcos)

- **Entregue**: pan/zoom nativo, crosshair + tooltip (preço/data-hora ao passar mouse/dedo), MM50/MM200/MM400 toggleáveis, marcadores nos toques da MM200.
- **NÃO entregue (fora do escopo da lib free)**: ferramenta de desenho livre (desenhar uma reta e arrastá-la). Isso é da versão paga do TradingView / exige plugin custom. Fica como etapa 2 se o Marcos quiser.

## O que entregou

### Lib
- `lightweight-charts@5.2.0` instalado (v5 — API nova: `addSeries(LineSeries, opts)`, `createSeriesMarkers`).

### Banco
- Model `StockPriceHistory` (stockId, interval, date, OHLCV, updatedAt). `@@unique([stockId, interval, date])`. Migration `add_price_history`.

### Backend
- `lib/chart-data.ts`:
  - **Daily (1M/3M/6M/1A/5A)**: persiste 5 anos no Postgres. Cache 24h (via max updatedAt). MMs calculadas sobre a série COMPLETA e fatiadas pro período → MM200 aparece correta mesmo no 1M.
  - **Intraday (1D/5D)**: fetch direto Yahoo (5m/15m), sem persistir, sem MM.
  - Detecta toques na MM200 (cruzamento de sinal close-mm200).
- `GET /api/stocks/[ticker]/price-chart?period=` — guardSession, retorna `{intraday, currency, price[], mm50[], mm200[], mm400[], touchesMM200[]}`.

### Frontend
- `components/app/stock-chart.tsx` (client): cria chart no useEffect, ResizeObserver pra responsivo, séries de preço + MMs, `createSeriesMarkers` nos toques, crosshair via `subscribeCrosshairMove` → tooltip overlay HTML (data/hora + preço + MMs ativas). Cores dark obrigatórias aplicadas. Toggles MM50/200/400. Botões 1D/5D/1M/3M/6M/1A/5A.
- Integrado em `/stock/[ticker]` no lugar do `StockPriceChart` antigo.

## Validações

| Cenário | Resultado |
|---|---|
| 1A 1ª vez (fetch+persist 5a) | 3.1s, 250 pts, MM50/200/400 × 250, 9 toques ✅ |
| 1A 2ª vez (cache banco) | 0.21s ✅ |
| 1M (MM200 via série completa) | 20 pts visíveis, MM200 presente ✅ |
| 1D intraday | 167 pts, timestamp unix, sem MM ✅ |
| Página /stock/AAPL34 | 200, "Gráfico de Preço" presente, sem erros ✅ |

## 🔴 Achado CRÍTICO de infra (importante pro futuro)

**Rotas de API novas exigem REBUILD da imagem Docker — não basta restart/recreate.**

Gastei bastante tempo nisso: a rota `/api/stocks/[ticker]/chart` (e depois `price-chart`, e até `/api/ping` trivial) davam **404** mesmo com:
- arquivo presente no container (bind mount ativo, confirmado via `ls`)
- type-check passando
- `docker restart`, `docker compose down/up`, `--force-recreate`, `--renew-anon-volumes`, `rm -fsv` + up

Rotas **antigas** (history, refresh, ai-analysis) funcionavam; só as **criadas após o último `docker compose build`** davam 404.

**Causa provável**: o Next 16 + Turbopack neste setup (Docker Desktop Windows + `WATCHPACK_POLLING`) faz o scan de rotas a partir do snapshot que estava presente no build da imagem; arquivos novos via bind mount não entram no manifesto em runtime.

**Solução**: `docker compose build app` + `up --force-recreate`. Depois disso, `/api/ping` e `/api/stocks/[ticker]/price-chart` passaram a responder 200.

**Implicação pro fluxo de trabalho**: toda vez que eu criar uma ROTA nova (arquivo route.ts novo), preciso rebuildar a imagem, não só restart. Edições em rotas existentes pegam por hot reload normal. Documentar isso evita perder tempo de novo.

> Curiosidade: por isso nos sprints anteriores os `--force-recreate` "funcionavam" pra descobrir rotas — na verdade o que resolvia era o rebuild que eu fazia junto, ou as rotas já estavam na imagem. Agora ficou claro o mecanismo.

## Resíduo limpo

- Removida a pasta `/api/stocks/[ticker]/chart` (tentativa abandonada pelo nome) — rota final é `price-chart`.
- `/api/ping` foi só teste de diagnóstico — **deixei no código**; pode remover depois (é inofensivo). TODO: remover ping.
- `StockPriceChart` (SVG antigo, Sprint 3.5) ainda existe exportado mas não é mais usado na página. Pode remover.

## Backlog

- **Remover `/api/ping`** (resíduo de debug).
- **Remover `StockPriceChart` SVG** antigo (substituído).
- **Ferramenta de desenho de reta arrastável** (etapa 2, se o Marcos quiser) — precisa plugin custom de série primitive do lightweight-charts.
- **MM400 em períodos curtos**: como guardo 5 anos daily (~1250 pts), MM400 funciona em 1A/5A. Em 1M/3M a MM400 aparece (calculada sobre série completa). OK.
- O endpoint `/api/stocks/[ticker]/history` (Sprint 3.5) ficou órfão — pode remover.

## Próximo

Marcos vai testar a interatividade no browser (/stock/AAPL34): pan, zoom, crosshair, trocar períodos, ligar/desligar MMs, ver marcadores de toque.
