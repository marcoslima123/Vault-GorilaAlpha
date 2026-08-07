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
