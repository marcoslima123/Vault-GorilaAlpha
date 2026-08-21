# Backlog — Inteligência de Mercado (ideias do sócio)

> Status: **ideia / a implementar.** Capturado em 2026-07-07 a partir de uma conversa com o sócio do Marcos.
> Contexto: bloco de features novas focadas em **sinais de mercado a partir de dados públicos** (B3, Tesouro Nacional, CVM). A tese central: sinalizar **fatos incomuns** (volume, taxa, participação) **sem dar recomendação** — só apontar o dado para o usuário monitorar.
> Ligações: [[00 - Home]] · [[03 - Roadmap]] · complementa o [[Diario/2026-05-27 - Inteligencia Historica MM200]] (mesma filosofia de "sinal, não conselho").

---

## Princípio comum a todas as ideias

> **Sinalizar o fato, nunca recomendar.** Toda apresentação mostra o dado bruto + o desvio vs. a média, e uma frase neutra ("movimentação incomum, monitorar"). Nunca "compre" / "venda".

Todas se apoiam em **dados públicos e gratuitos**, o que as torna viáveis sem contrato de dados caro.

---

## 1. Radar de Volume Anômalo — "Tem boi na linha"

**O que é:** detector automático de volume muito acima da média histórica em ações (especialmente small caps).

**Como funciona:**
```
Calcula média exponencial de volume dos últimos 30 dias
        ↓
Monitora volume de hoje em tempo real
        ↓
Se volume > 2x a média → dispara alerta
        ↓
"Ativo negociou hoje R$60M vs média de R$30M
 Volume 2x acima do normal — monitorar"
```

**Aplica para:** Ações (foco small caps) e Derivativos/Opções da B3.

**Fonte de dados:** B3 disponibiliza book de negociação aberto — dá pra ver quantidade e corretora que comprou.

**Como apresentar (sem recomendação):**
```
⚠️ Volume anômalo detectado

3R Petroleum (RRRP3)
Volume hoje:    R$ 62,4M
Média 30d:      R$ 28,1M
Desvio:         +122% acima da média

"Movimentação incomum detectada.
 Monitorar para possível evento."
```

**Prioridade:** 🟢 Alta — dados já públicos.

> 💡 **É a mais barata das cinco, e mais do que o registro original sugeria.** Não precisa de fonte de dados nova: `StockPriceHistory.volume` (série diária) e `StockQuote.volume` (dia corrente, criado em 2026-08-07) **já estão no banco**. A média 30d e o desvio saem de query. Boa candidata a primeira implementação — serve de prova da engine de detecção de anomalia que as ideias 3 e 4 vão reusar.

---

## 2. Monitor de Derivativos — Opções com volume anômalo

**O que é:** mesmo conceito do Radar de Volume, aplicado ao mercado de **opções** da B3.

**Por que é poderoso:** no Brasil, quem tem informação privilegiada costuma comprar ação **e** derivativo. O derivativo alavanca mais → grande movimentação em opções de uma empresa pequena é um sinal ainda mais forte.

**Como funciona:**
```
Monitora volume de contratos de opções por ativo
        ↓
Compara com média histórica dos últimos 30 dias
        ↓
Se volume de opções > 3x a média → alerta
        ↓
"RRRP3 opções: R$15M hoje vs média R$5M"
```

**Prioridade:** 🟡 Média — precisa de acesso ao book de opções.

---

## 3. Monitor de Leilões do Tesouro Direto

**O que é:** análise automática dos resultados dos leilões do Tesouro Nacional. Quando a **taxa ofertada sobe mas a demanda cai**, o mercado está dando um recado sobre expectativas de inflação.

**Como funciona:**
```
Tesouro Nacional faz leilão de NTN-B 2050
        ↓
Últimos leilões: IPCA + 7,00%
Leilão hoje:     IPCA + 8,20%
Demanda:         menor que anteriores
        ↓
"Mercado sinalizando que expectativa de
 inflação está acima do projetado pelo governo"
```

**Fonte de dados:** Tesouro Nacional publica todos os resultados em `tesouronacional.fazenda.gov.br`.

**Como apresentar:**
```
📊 Leilão Tesouro — Sinal do Mercado

NTN-B 2050 (IPCA+)
Taxa ofertada: IPCA + 8,20%
Taxa anterior: IPCA + 7,00%
Demanda:       ▼ Abaixo da média

"Taxa subiu mas demanda caiu — mercado
 pode estar precificando inflação maior
 do que o governo projeta."
```

**Prioridade:** 🟢 Alta — Tesouro Nacional tem API pública. Sinergia direta com o [[Diario/2026-06-09 - Modulo Renda Fixa]].

---

## 4. Radar de Fundos — Quem está comprando o quê

**O que é:** monitor das **cartas de participação relevante**. Quando um fundo adquire mais de **5%** do capital de uma empresa, é obrigado por lei a comunicar à CVM — dado público.

**Por que é poderoso:** quando fundos com histórico excelente (Constellation, Verde, SPX, Kapitalo) estão comprando uma ação específica, é um sinal institucional fortíssimo.

**Como funciona:**
```
CVM publica comunicados de participação relevante
        ↓
Sistema captura automaticamente
        ↓
Cruza com histórico de rentabilidade do fundo
        ↓
"Constellation (retorno +150% em 3 anos)
 adquiriu 6,2% de RECV3 em 15/05/2026"
```

**Fonte de dados:** CVM disponibiliza todos os comunicados de participação relevante em `cvm.gov.br` via dados abertos.

**Prioridade:** 🟢 Alta — CVM tem dados abertos gratuitos.

---

## 5. Indicadores de Volatilidade Implícita

**O que é:** monitorar quando a volatilidade de uma ação sobe acima do normal — geralmente antecede eventos (resultados, notícias, movimentações).

**Como funciona:**
```
Calcula volatilidade histórica (desvio padrão dos retornos)
        ↓
Compara com volatilidade implícita das opções
        ↓
Gap grande entre as duas → mercado precificando evento
        ↓
"Volatilidade implícita de PETR4 subiu 40%
 acima da histórica — mercado antecipando evento"
```

**Prioridade:** 🟡 Média — precisa calcular a partir das opções.

---

## 🆕 2026-08-19 — conversa com o Edmar e descoberta técnica

### O que o Edmar acrescentou

> *"Tem muito player de mercado que se entrar no mercado à vista, ele faz muito preço. Então normalmente tem player que entra via derivativo porque é mais 'escondido'."*

Esse é o **porquê** do Monitor de Derivativos, que faltava no registro original. Não é só "alavanca mais": quem tem tamanho **não consegue** comprar à vista sem mover o preço contra si. O derivativo é onde dá pra montar posição sem denunciar.

Outros três pontos:

1. **Janelas de 30/60/90 dias**, não só 30. Barato de fazer e dá contexto (2x sobre 30d pode ser normal se 90d já vinha subindo).
2. **Não tentar inferir a estratégia.** Ele diz que a parte difícil do derivativo é trazer contexto pra leigo — saber se é trava de alta, compra seca de call etc. **Isso reforça o princípio da nota:** sinalizar o fato (delta de volume), nunca interpretar a operação. Tentar adivinhar a estratégia seria dar conselho disfarçado.
3. **Volume anômalo de derivativo + volatilidade implícita baixa = o sinal mais forte.** Se a VI está muito baixa e o volume de derivativos dispara, a chance da VI subir é alta. **Isso funde as ideias #2 e #5 num sinal só**, em vez de duas features separadas.

### 🔓 Descoberta: a brapi PRO tem chain de opções

A ideia #2 estava marcada 🟡 média **porque dependia de "acesso ao book de opções"**. Esse bloqueio não existe — testado em 2026-08-19 com o token PRO que já pagamos:

```
GET /api/v2/options/expirations?underlying=PETR4
  -> 20+ vencimentos, com tradedOnly

GET /api/v2/options/chain?underlying=PETR4&expirationDate=2026-08-21
  -> 253 contratos
```

Campos por contrato:

```
symbol, underlyingSymbol, side (call|put), market, optionStyle,
strike, expirationDate, firstTradeDate, lastTradeDate,
open, high, low, average, close, bid, ask,
trades, volume, financialVolume
```

**Custo de armazenamento é baixo** se guardarmos o agregado: o sinal precisa do volume financeiro **somado por ativo por dia**, não de cada contrato. Isso é 1 linha por ação por dia. O custo real é de chamadas (1 por vencimento), mas o volume concentra nos 2–3 vencimentos mais próximos.

### ⚠️ O que a chain NÃO traz: volatilidade implícita

Não há campo de VI. Teríamos que calcular (Black-Scholes) a partir de strike, vencimento, preço da opção e preço do ativo — todos presentes — mais a **taxa livre de risco**.

> 💡 E a taxa livre de risco **já temos**: a Selic vem do BCB pelo módulo de [[Diario/2026-06-09 - Modulo Renda Fixa]]. A ideia #5 deixa de depender de fonte externa nova.

### Impacto na priorização

| Ideia | Antes | Agora |
|---|---|---|
| #2 Monitor de Derivativos | 🟡 média (dependia do book) | 🟢 **alta** — dado disponível na brapi PRO |
| #5 Volatilidade Implícita | 🟡 média (calculado das opções) | 🟡 média, mas **sem bloqueio de fonte** — falta só o cálculo |

## Prioridade recomendada

| # | Ideia | Fonte de dados | Prioridade |
|---|---|---|---|
| 1 | Radar de Volume Anômalo (ações) | B3 (dados abertos) | 🟢 Alta |
| 3 | Monitor de Leilões do Tesouro | Tesouro Nacional (API) | 🟢 Alta |
| 4 | Radar de Fundos (cartas CVM) | CVM (dados abertos) | 🟢 Alta |
| 2 | Monitor de Derivativos (opções) | B3 (book de opções) | 🟡 Média |
| 5 | Volatilidade Implícita | Calculado das opções | 🟡 Média |

**Sugestão de sequência:** começar pelas 3 de dados abertos já acessíveis (1 → 3 → 4), que compartilham o mesmo padrão "detectar desvio vs. média 30d → sinalizar fato". As de opções (2, 5) vêm depois, pois dependem do book de derivativos.

---

## Notas de implementação (quando for fazer)

- As três de "alta prioridade" compartilham a **mesma engine**: baixar série pública → calcular média/desvio de 30d → detectar outlier → renderizar card neutro. Vale criar um **service genérico de "detecção de anomalia"** reaproveitável.
- Encaixe natural como **novo tipo de alerta** (estende o sistema de `Alert` do [[Sprints/Sprint 07 - Alerts e Paywall]]) e/ou um **feed de sinais** (padrão visual do Feed de Relatórios).
- Candidatas fortes a **feature Pro** (inteligência de mercado como diferencial pago).
- Os exemplos usam `RRRP3` (hoje `BRAV3` após fusão 3R+Enauta) — apenas ilustrativos.
- Fontes a validar: formato/limite da API do Tesouro, dataset de participação relevante da CVM (dados abertos), e disponibilidade do book de opções da B3.
