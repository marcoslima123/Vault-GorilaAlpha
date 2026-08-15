# Backlog — Correlação Macro + Fundamentos

> Status: **ideia / a implementar.** Ideia do sócio do Marcos, registrada em 2026-08-15.
> Contexto: cruzar **dados macroeconômicos** (juros, câmbio, commodities) com o **desempenho histórico dos setores** da bolsa, para responder perguntas do tipo *"nas últimas X vezes que o açúcar ficou abaixo de $Y, empresas sucroenergéticas caíram Z%"*.
> Ligações: [[00 - Home]] · [[03 - Roadmap]] · [[Backlog - Inteligencia de Mercado (ideias do socio)]] (mesma filosofia de sinal, não conselho) · [[Diario/2026-06-09 - Modulo Renda Fixa]].

---

## A ideia em uma frase

> **O usuário sabe o que ele tem. Não sabe o que o cenário atual já fez com isso antes.** A feature liga as duas pontas: pega o macro de hoje e mostra o que aconteceu historicamente com os setores dele em cenários parecidos.

## O que cruzar

| Eixo macro | Fonte | Setores mais sensíveis |
|---|---|---|
| **Selic / IPCA** | BCB (SGS) | bancos, construção civil, varejo, consumo discricionário |
| **Câmbio USD/BRL** | já temos (`fx.service.ts`) | exportadoras, celulose, proteína, importadoras |
| **Juros do Fed / treasuries** | FRED | fluxo estrangeiro, BDRs, small caps |
| **Commodities** (açúcar, petróleo, minério, grãos) | FRED | sucroenergético, petróleo, mineração, agro |

Cruzados com o **desempenho histórico por setor** — o campo `sector` já existe em `Stock` (`schema.prisma`), junto com `industry` e `market`.

## Como se apresenta

Seguindo o princípio do [[Backlog - Inteligencia de Mercado (ideias do socio)]] — **sinalizar o fato, nunca recomendar**:

```
📉 Contexto macro atual — seus setores

Açúcar:  US$ 17,80/lb  (abaixo de US$ 18 desde 12/07)

Nas 6 vezes desde 2015 em que o açúcar ficou
abaixo de US$ 18 por mais de 30 dias, o setor
sucroenergético da B3 recuou em média 11,4%
nos 90 dias seguintes (pior: -23%, melhor: +2%).

Você tem 2 ações desse setor na watchlist.

"Padrão histórico, não previsão."
```

O gancho de valor é a última linha antes do disclaimer: **cruzar com a watchlist do usuário** transforma um dado genérico em algo pessoal.

## Por que é viável aqui

Boa parte da fundação **já está construída**, e por acaso — veio do módulo de Renda Fixa:

- **BCB e FRED já são consumidos em produção** (`services/fixed-income/brazil.service.ts` e `history.service.ts`). O `history.service.ts` já tem um catálogo de séries FRED com cache de 6h — adicionar séries de commodities é estender um mapa, não construir integração.
- **Câmbio já existe** (`fx.service.ts`).
- **Histórico de ações já está no banco:** `StockPriceHistory` (`stockId`, `interval`, `date`, `close`, `volume`), com unique em `[stockId, interval, date]`.
- **Setor já está no banco:** `Stock.sector` e `Stock.industry`.
- **Watchlist já existe** para o cruzamento pessoal.

> Ou seja: o trabalho novo é sobretudo **o motor de correlação e a apresentação**, não coleta de dados.

## Notas de implementação (quando for fazer)

- **O motor é o mesmo padrão das outras**: definir uma condição macro (limiar + duração) → achar as janelas históricas em que ela valeu → medir o retorno médio do setor nos N dias seguintes → renderizar card neutro. Mesma família da engine de detecção de anomalia sugerida em [[Backlog - Inteligencia de Mercado (ideias do socio)]] — vale checar se dá pra compartilhar.
- **Índice setorial sintético:** não existe índice por setor pronto no banco. Precisa construir a série do setor agregando as ações (média ponderada por valor de mercado, ou simples para começar) a partir de `StockPriceHistory`.
- **Rigor estatístico é o risco principal.** "6 ocorrências desde 2015" é uma amostra minúscula. Sem cuidado, a feature vira gerador de coincidências convincentes. Mitigações: exigir **N mínimo de ocorrências** para exibir, mostrar **dispersão** (pior/melhor caso, não só a média) e nunca apresentar como previsão.
- **Sobrevivência e viés de seleção:** o banco tem as ações de hoje, não as que quebraram no caminho. Retornos setoriais históricos vão sair otimistas por construção. Vale declarar isso.
- **Cobertura do histórico:** conferir até onde `StockPriceHistory` vai de fato para B3 (o `5y` da brapi PRO). Se não alcançar 2015, o recorte precisa ser mais curto ou o histórico precisa ser aprofundado antes.
- **Candidata a feature Pro** e/ou a novo tipo de card no Feed.
