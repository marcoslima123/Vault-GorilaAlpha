# Decisão — Mercados de foco + Estratégia de IA

**Data:** 2026-05-20 · **Status:** ✅ decidido

Voltar para [[00 - Home]].

## 1. Mercados no MVP

**Decisão:** lançar com **B3 + BDR + US** desde o início (os 3 mercados que já estão no banco).

### Por quê
- Os 1410 stocks já estão sincronizados — não tem retrabalho pra incluir.
- Diferencial de marketing real: "único brasileiro com análise IA de B3 + BDR + US num só lugar".
- Esforço extra é só de UI (formatação de moeda BRL vs USD), não de arquitetura.

### Implicações técnicas
- Toda renderização de `price`, `marketCap`, `volume` precisa usar `stock.currency` pra formatar (BRL vs USD).
- Criar helper `apps/web/src/lib/format-currency.ts` (ou estender se existir).
- Filtros do Screener precisam funcionar com a moeda do mercado selecionado (não fazer conversão automática — comparar P/L entre US e B3 é OK, comparar valor absoluto não).
- Setores: nomes em PT pra B3/BDR, em EN pra US. Pode haver inconsistência (`Energia` vs `Energy`). Avaliar normalizar para EN-baseado-em-GICS ou manter como vem do Yahoo — backlog se virar problema.
- Análise IA precisa adaptar prompt à origem (texto em PT pra todas, mas referenciar contexto: "ações brasileiras", "ADRs americanos no Brasil", "ações listadas nos EUA").

## 2. Modelo de Análise IA

**Decisão:** modelo **híbrido** alinhado com o paywall:

- **Free vê análise heurística** (regras determinísticas em código)
- **Pro vê análise IA via Anthropic SDK** (Claude Haiku 4.5 + prompt caching)

### Por quê
- Free tem valor real (heurística não é placebo — 20-30 regras boas cobrem 80% dos casos).
- Pro tem feature visível: "Análise IA aprofundada" vs "Análise básica" lado a lado. Gatilho de upgrade tangível.
- Custo da IA fica controlado (só pro paga, cache 7 dias por ticker, ~R$ 0,02 efetivo por análise).

### Arquitetura proposta
- `apps/web/src/lib/analysis-heuristic.ts` — função pura que recebe `StockIndicator` e retorna `{ pontosFortes: string[], riscos: string[], tese: string }`.
- `apps/web/src/lib/analysis-ai.ts` — função async que chama Anthropic, cacheada em um novo model `AiAnalysis` (ticker, generatedAt, content, model).
- Componente `StockAiAnalysis` decide qual chamar baseado em `getEffectivePlan(session)`.
- Cache: 7 dias por (ticker, model). Renovar se `StockIndicator.fetchedAt` for mais recente que `AiAnalysis.generatedAt`.

## 3. Tom da análise

**Decisão:** **descritivo, nunca prescritivo**.

### Regras
- ❌ Nunca dizer "compre", "venda", "evite", "recomendamos", "tem potencial de alta".
- ✅ Descrever fatos: "ROE de 22% acima da média do setor (15%)", "Dívida líquida sobre EBITDA de 4x — historicamente sustentável para utilities mas alto para indústria geral".
- ✅ Apontar pontos de atenção como observações neutras: "DY de 12% sustentado por payout de 110% — observar pagamento via reservas vs lucro corrente".

### Por quê
- Risco regulatório CVM: análise prescritiva (recomendação de investimento) exige analista certificado (CNPI). Análise descritiva é material informativo, regulação mais leve.
- Modelo de Status Invest, Investidor10, Bússola — todos descritivos.
- Disclaimer obrigatório visível no rodapé do `StockAiAnalysis`:
  > "Material informativo. Não constitui recomendação de investimento. Decisões são de responsabilidade do investidor."

## Implicações nos sprints

- [[Sprints/Sprint 03 - Stock Detail Real]] — implementar `analysis-heuristic.ts` (free) já no Sprint 03 com 15-20 regras. IA fica pra Sprint 07.
- [[Sprints/Sprint 02 - Screener Funcional]] — adicionar `format-currency.ts` para lidar com BRL vs USD nas células de price/marketCap.
- [[Sprints/Sprint 07 - Alerts e Paywall]] — implementar `analysis-ai.ts` + model `AiAnalysis` no banco + gate por plano + disclaimer global.

## Decisões derivadas pendentes

- [ ] Definir o conjunto inicial de regras heurísticas (20-30) — fica para o Sprint 03.
- [ ] Definir o prompt-mestre da análise IA — fica para o Sprint 07.
- [ ] Decidir formato do disclaimer (componente único reusável vs hardcoded em cada lugar).
