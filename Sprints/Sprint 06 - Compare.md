# Sprint 06 — Compare

**Esforço:** 2 dias · **Status:** ✅ concluído em 2026-05-21

← [[Sprints/Sprint 05 - Rankings]] · próximo: [[Sprints/Sprint 07 - Alerts e Paywall]]

## Objetivo

Comparador lado a lado de 2 a 4 ativos. Sem persistência — tudo via URL.

## Entregáveis

### UI
- [ ] `/compare?tickers=PETR4,VALE3,ITUB4,BBDC4` — URL define o estado.
- [ ] `CompareTickerSelector`:
  - Adicionar/remover ticker (autocomplete buscando em `Stock`).
  - Mínimo 2. **Free = 2 ativos máx; Pro = 4 ativos** (decidido em [[Decisoes/2026-05-20 - Modelo de Monetizacao]]). Tentar adicionar o 3º no free abre `<UpgradeModal />`.
- [ ] `CompareHealthMatrix`:
  - Tabela cruzada: linhas = indicadores (P/L, P/VP, ROE, ROIC, DY, divLiqEbitda, margemLiquida, etc.)
  - Colunas = tickers
  - Célula colorida pelo melhor/pior do grupo (verde/vermelho gradient)
- [ ] `CompareBenchmarking`:
  - Radar/spider chart com 5 dimensões: Valuation, Qualidade, Saúde Financeira, Crescimento, Dividendos (reusa `lib/scoring.ts` do Sprint 03).
  - **Pro-only.** No free renderizar tarjado com CTA "Disponível no Pro".
- [ ] `CompareSidebarPanels` e `CompareBottomPanels`: detalhes complementares (margens, balanço, etc.).
- [ ] Botão "Salvar comparação" → entra como backlog (poderia virar feature paga: salvar comparações nomeadas).

### Sem novos endpoints

- Server component faz `Promise.all` em `prisma.stock.findMany({ where: { ticker: { in: tickers } }, include: { indicator: true } })`.

## Critério de pronto

- URL compartilhável reproduz a comparação no outro navegador.
- Trocar tickers atualiza sem reload de página.
- Tickers inválidos são removidos da URL silenciosamente (ou mostram aviso).

## Notas técnicas

- Validar tickers contra DB antes de renderizar (filtrar inválidos do array).
- Pode reaproveitar `lib/scoring.ts` do [[Sprints/Sprint 03 - Stock Detail Real]] — daí a ordem.

## Riscos

- Comparar empresas de setores diferentes pode gerar conclusões ruins. Adicionar aviso visual quando setores divergem ([meu palpite] não bloquear, mas sinalizar).
