# Backlog — Adversarial Verification + Tournament

> Status: **ideia / a implementar.** Registrado em 2026-08-15.
> Contexto: sistema **multi-agente** para a análise de ações. Em vez de um único agente produzir a análise fundamentalista, **Bull, Bear e Quant analisam em paralelo, se atacam, e um agente juiz sintetiza o veredito** com score de confiança.
> Ligações: [[00 - Home]] · [[03 - Roadmap]] · [[Decisoes/2026-05-20 - Mercados e IA]] (tom descritivo) · complementa [[Backlog - Inteligencia de Mercado (ideias do socio)]].

---

## A ideia em uma frase

> **Uma tese só vale o quanto ela resiste a ser atacada.** Em vez de entregar uma análise que parece boa, entregar uma análise que **sobreviveu a um ataque cruzado** — e dizer ao usuário o quanto ela sobreviveu.

## Os dois componentes

### 1. Adversarial Verification

Um **agente atacante** cuja única função é tentar derrubar a análise fundamentalista produzida. Não é revisor, é adversário: recebe a tese e é instruído a refutá-la.

O que ele ataca:
- números que não sustentam a conclusão
- indicador citado fora de contexto (P/L baixo que é armadilha de valor)
- omissão relevante (dívida, diluição, concentração de receita)
- extrapolação de tendência sem base

O que sobrevive ao ataque entra na análise final. O que não sobrevive é descartado **ou** aparece como ressalva explícita.

### 2. Tournament — Bull vs Bear vs Quant

Três agentes analisam **a mesma ação em paralelo**, cada um com uma lente diferente e sem ver o output dos outros:

| Agente | Lente |
|---|---|
| **Bull** | constrói a tese de alta mais forte possível com os dados reais |
| **Bear** | constrói a tese de baixa mais forte possível com os mesmos dados |
| **Quant** | ignora narrativa; olha só números, múltiplos, séries históricas e desvios |

Depois, **cada um ataca as teses dos outros dois**. Por fim um **agente juiz** lê tudo — as três teses e os ataques cruzados — e sintetiza o veredito final.

```
             ┌─ Bull  ─┐
ação ────────┼─ Bear  ─┼──→ ataque cruzado ──→ juiz ──→ veredito + confiança 0-100
             └─ Quant ─┘
```

### 3. Score de confiança 0–100

O número que sai do processo: **quanto da tese resistiu ao ataque cruzado**.

Não é "quão boa é a ação" — é **quão confiável é a análise**. Uma ação pode ter veredito positivo com confiança 40 (tese frágil, muita divergência entre os agentes) ou veredito neutro com confiança 90 (os três concordam que não há caso claro).

> Isso é honesto de um jeito que análise de mercado normalmente não é: expõe a incerteza em vez de escondê-la atrás de um texto bem escrito.

## Por que isso é diferenciador

O mercado de apps de análise entrega **uma opinião**. Aqui a entrega é **uma opinião + o quanto ela aguentou apanhar**. É difícil de copiar porque o valor não está no prompt, está na arquitetura do pipeline.

Casa com a decisão de tom já tomada em [[Decisoes/2026-05-20 - Mercados e IA]]: **descritivo, não prescritivo**. O score de confiança reforça isso — não manda comprar, mostra o grau de solidez do que foi apurado.

## Modelo de negócio

**Feature Pro.** Multiplica o custo de tokens (5+ chamadas onde hoje há 1), mas eleva a qualidade da análise exponencialmente. É exatamente o tipo de coisa que justifica assinatura.

O gate já existe: `PLAN_LIMITS.free.aiAnalysis = false` / `pro.aiAnalysis = true` em `apps/web/src/lib/plan.ts`. Provavelmente vale um limite próprio (ex.: `tournamentPerMonth`) em vez de reusar o flag da análise simples, já que o custo por execução é muito maior.

## Notas de implementação (quando for fazer)

- **Modelo:** hoje `AI_MODEL = "claude-haiku-4-5-20251001"` em `apps/web/src/lib/anthropic.ts`. Para o juiz vale avaliar um modelo mais forte — os três analistas podem seguir em Haiku (paralelizáveis e baratos), o juiz é onde a qualidade decide o resultado.
- **Custo:** 3 análises + 6 ataques cruzados + 1 síntese = **10 chamadas por ação**. Precisa de cache agressivo por ticker (a análise não muda de hora em hora) e provavelmente execução **assíncrona** com o resultado persistido, não síncrona no request.
- **Reuso do que já existe:** o system prompt estruturado do processador de relatórios (`services/reports/report-processor.ts`) já é um precedente de "Claude devolve JSON estruturado e a gente persiste". Mesmo padrão serve aqui, e permite cachear cada etapa.
- **Prompt caching:** os system prompts dos 3 analistas são fixos — usar `cache_control: { type: "ephemeral" }`, como já é feito em `ai-advisor.service.ts`.
- **Persistir o processo, não só o resultado.** Guardar as teses e os ataques permite mostrar ao usuário *por que* a confiança é 62. Sem isso o número vira caixa-preta.
- **Risco de teatro:** se os prompts não forem bem construídos, os agentes concordam entre si e o "ataque" vira formalidade — confiança sempre alta, sinal zero. Vale validar cedo com casos onde a resposta certa é conhecida (uma ação obviamente problemática deve produzir confiança baixa).
