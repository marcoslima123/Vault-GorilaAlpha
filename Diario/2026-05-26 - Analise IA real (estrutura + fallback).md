# 2026-05-26 — Análise IA real (estrutura + fallback)

← [[00 - Home]] · sessão anterior: [[Diario/2026-05-26 - Frente A telas mock ligadas]]

## Objetivo

Implementar a Análise IA (Claude) como feature Pro, fechando a decisão da [[Decisoes/2026-05-20 - Mercados e IA]]: free vê heurística, pro vê IA. Escolhido o caminho **"construir com fallback"** — toda a estrutura pronta; sem `ANTHROPIC_API_KEY` real cai na heurística automaticamente.

## O que entregou

### Banco
- Model `AiAnalysis` (cache): `stockId, model, content (Json), generatedAt`. `@@unique([stockId, model])`. Migration `20260526194103_add_ai_analysis`.

### Anthropic
- `lib/anthropic.ts` += `hasAnthropicKey()` (detecta key real vs placeholder vs vazia) + `AI_MODEL = "claude-haiku-4-5-20251001"`.

### Lib de análise
- `lib/analysis-ai.ts` — `getAiAnalysis(stock, indicator)`:
  - Sem key real OU sem indicator → retorna `null` (fallback).
  - Cache 7 dias por (stock, model); renova se `generatedAt < indicator.fetchedAt` (dados mudaram).
  - Chama Claude Haiku 4.5 com **prompt caching** (system block `cache_control: ephemeral`).
  - Prompt descritivo com regras anti-prescrição explícitas (proíbe "compre/venda").
  - Parse JSON robusto (lida com fences markdown). Retorna shape `AnalysisOutput` (igual heurística).
  - Em erro: usa cache antigo se houver, senão null.
- `lib/analysis-heuristic.ts` += `analysisToSections()` compartilhado (DRY — presenter e endpoint usam).

### Endpoint
- `POST /api/stocks/[ticker]/ai-analysis` — guardSession + checa `PLAN_LIMITS.aiAnalysis`:
  - Free → 403 `plan_locked`.
  - Pro → gera/cacheia. Sem key → `{available:false}`. Com análise → `{available:true, sections}`.

### UI
- `StockAiAnalysis` ganhou `variant: "ai" | "heuristic"` (muda header "ANÁLISE IA · CLAUDE" vs "ANÁLISE FUNDAMENTALISTA") + `loading`.
- `AiAnalysisLoader` (client) novo:
  - Free → renderiza heurística direto.
  - Pro → mostra heurística + "gerando…", fetcha o endpoint, substitui por IA quando chega. Botão "Regenerar". Se `available:false` (sem key) ou erro → mantém heurística.
- Página `/stock/[ticker]` troca `StockAiAnalysis` por `AiAnalysisLoader` (passa heurística como fallback + plan + ticker).

## Validações (sem key real — caminho de fallback)

| Cenário | Resultado |
|---|---|
| `POST ai-analysis` como **pro**, sem key | 200 `{available:false}` ✅ |
| `/stock/PETR4` renderiza | 200, "ANÁLISE FUNDAMENTALISTA" (heurística) ✅ |
| `POST ai-analysis` como **free** | 403 `plan_locked` ✅ |

## Pra ativar a IA de verdade (falta só a key)

1. Pegar `ANTHROPIC_API_KEY` em https://console.anthropic.com (com créditos).
2. Colar no `.env` substituindo `sk-ant-COLOQUE_SUA_KEY_AQUI`.
3. Recriar container.
4. Pronto: pro passa a ver análise gerada pelo Claude Haiku 4.5, cacheada 7 dias.

Nenhuma mudança de código necessária — `hasAnthropicKey()` detecta a key e o fluxo liga sozinho.

## Custo esperado (quando ligar)

Haiku 4.5 + cache 7d por ticker + prompt caching ≈ R$ 0,02 por análise nova. 1ª visita de cada ticker gera; próximas 7 dias servem do banco.

## Observações

- **Modelo**: `claude-haiku-4-5-20251001` (o `/api/analysis` legado usa sonnet-4 antigo — esse endpoint legado pode ser removido/migrado depois; não é usado pela UI nova).
- **Tom descritivo** reforçado no system prompt (CVM). Disclaimer global já existe na página.
- **`AiAnalysisLoader` faz fetch client** → heurística aparece instantânea, IA chega depois sem travar o page load.

## Estado do produto

Análise IA estruturalmente completa. É a última peça do diferencial Pro.

## ✅ ATUALIZAÇÃO 2026-05-27 — IA LIGADA E FUNCIONANDO

Marcos colou a `ANTHROPIC_API_KEY` real + adicionou créditos na conta Anthropic. Testado:
- `POST /api/stocks/PETR4/ai-analysis` (pro) → `available=true`, 7.9s na 1ª geração.
- Análise rica e **100% descritiva** (RESUMO, PONTOS FORTES, OBSERVAÇÕES, PONTOS DE ATENÇÃO) — sem recomendação de compra/venda, conforme decisão CVM.
- Cacheada no banco (`AiAnalysis`) por 7 dias → próximas visualizações instantâneas.

A Análise IA (Claude Haiku 4.5) está **em produção no dev**, ligada pro plano Pro. Heurística continua pro Free.

## Próximo

- Colar `ANTHROPIC_API_KEY` quando tiver → IA liga
- **7.D Asaas** (pagamento) quando conta validar
- **Frente B deploy** pra ativar cron + URL pública
