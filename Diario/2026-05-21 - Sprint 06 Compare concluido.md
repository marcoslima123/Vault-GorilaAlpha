# 2026-05-21 — Sprint 06 (Compare) concluído

← [[00 - Home]] · sessão anterior: [[Diario/2026-05-21 - Sprint 05 Rankings concluido]]

## O que entregou

### Service + types
- `apps/web/src/services/compare/index.ts` — `parseTickersParam`, `serializeTickers`, `colorFor`, palette `COMPARE_COLORS` (4 cores: azul, laranja, verde, lilás) e tipo `ComparedStock` (stock + indicator + scores + color).

### Página `/compare` — virou server component
- Lê `?tickers=` da URL, valida no banco, calcula `computeScores()` por ativo.
- Aplica `PLAN_LIMITS.compareMax` (free=2, pro=4) — se URL pede mais que o limite, `redirect()` trim pro limite.
- Tickers inválidos são filtrados via redirect (mantém só os que existem na base).
- Estados: vazio → empty state; 1 ticker → empty state pedindo segundo; 2+ → comparação completa.
- Disclaimer obrigatório no rodapé.

### Componentes refatorados

| Componente | Mudança |
|---|---|
| `CompareTickerSelector` | Autocomplete real consumindo `fetchStocks({ search })`. Adiciona/remove via `router.replace`. Mostra erro inline ao tentar passar do limite. Tickers já adicionados ficam desabilitados na lista. |
| `CompareHealthMatrix` | Recebe `ComparedStock[]` + `plan`. 3 modos: Bars, Heatmap, Radar. **Radar é PRO-only** (free vê com blur + overlay "Disponível no Pro"). Usa scores 0-10 do `lib/scoring`. Cores da palette. 6 eixos: Valuation, Qualidade, Saúde Fin., Dividendos, Crescimento, Geral. |
| `CompareBenchmarking` | Tabela com 13 indicadores reais (Preço, Market Cap, P/L, P/VP, EV/EBITDA, ROE, ROIC, Margem Líq., Dív./EBITDA, Liq. Corrente, DY, Cresc. Receita, Cresc. Lucro). Verde = melhor do grupo, vermelho = pior (configurável por `betterIs: "higher" \| "lower" \| "none"`). |
| `CompareSidebarPanels` | Lista de ativos comparados linkáveis. **Aviso de setores diversos** quando há mistura (com explicação do impacto). Aviso de moedas diferentes se mercados diferentes. Painel de plano (free/pro) com CTA upgrade. |
| `CompareBottomPanels` | Tabela com 8 itens de balanço/DRE (Receita, EBITDA, Lucro Líq., CAPEX, Ativo Total, Patrim. Líq., Dívida, Caixa) + Margem Líquida. Valores em moeda local de cada ativo (não convertidos — explicado no header). |

## Validações

| Cenário | Resultado |
|---|---|
| `/compare` (vazio) | 200, empty state ✅ |
| `/compare?tickers=PETR4,VALE3` (pro) | 200, matriz + benchmarking + bottom ✅ |
| `/compare?tickers=PETR4,VALE3,ITUB4,BBAS3` (pro, 4) | 200 ✅ |
| Free + 3 tickers | 307 redirect para 2 ✅ |
| Ticker inexistente | 307 filtra silenciosamente ✅ |
| Free + 2 tickers | 200, painel "PLANO FREE" visível ✅ |
| Free + radar | overlay "DISPONÍVEL NO PRO" com blur ✅ |

## Decisões aplicadas

- **Tom descritivo** ([[Decisoes/2026-05-20 - Mercados e IA]]): tabelas/headers neutros. Subtítulo do benchmarking diz "Verde sinaliza o melhor valor entre os ativos comparados" — **não** "comprar/vender".
- **Paywall** ([[Decisoes/2026-05-20 - Modelo de Monetizacao]]): free 2 ativos com bars/heatmap; pro 4 ativos + radar.
- **Setores diversos**: warning visível mas não bloqueante. Usuário decide se vale comparar empresas de setores diferentes.

## Achados / limitações

- **Sem AVG INDUSTRY** — não temos benchmark setorial agregado no banco; tabela compara só entre os ativos selecionados. Sprint futuro pode calcular média setorial via agregação.
- **Comparação cross-currency**: ativos em BRL vs USD aparecem na própria moeda (não convertidos). Mostra warning na sidebar. Conversão real precisaria de cotação USD/BRL no banco — fora do escopo.
- **`CompareBenchmarking` mostra "—" muito** quando ativos US tem campos nulos (Yahoo Finance não retorna alguns indicadores pra US). Tolerável.

## Estado do plano do user

Demovi o user pra `free` durante testes. Pra voltar pro `pro`:

```sql
UPDATE subscriptions SET plan='pro' WHERE user_id IN (
  SELECT id FROM users WHERE email='marcoslima_1212@hotmail.com'
);
```

## Próximo

[[Sprints/Sprint 07 - Alerts e Paywall]] (5-7 dias) — última peça do MVP. Model `Alert`, cron via Vercel/Asaas integration, modal de upgrade adequado.

Alternativa: **Sprint 3.5** (gráfico histórico) — opcional, polish.
