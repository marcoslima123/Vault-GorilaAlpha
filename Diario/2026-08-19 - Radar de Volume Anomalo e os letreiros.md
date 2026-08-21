# 2026-08-19 — Radar de Volume Anômalo, backfill e os dois letreiros

← [[00 - Home]] · sessão anterior: [[Diario/2026-08-18 - IA no Ranking e arquitetura de cache]] · base: [[Levantamento Tecnico e de Produto (2026-08-15)]]

> Primeira das cinco ideias do sócio entregue: o **Radar de Volume Anômalo**. Junto vieram o backfill que o viabiliza, a correção de uma amplificação de escrita de 2500x, e dois letreiros — o público na home e o de sinais na área logada.
>
> A conversa com o Edmar neste dia **destravou o Monitor de Derivativos**, que estava marcado como média prioridade por um bloqueio que não existe. Registrado em [[Backlog - Inteligencia de Mercado (ideias do socio)]].

---

## 1. O backfill, que era o pré-requisito escondido

O Radar precisa da média de volume de 30 dias. Mas ao medir a cobertura veio a surpresa:

```
PROD — acoes com serie diaria: 10 · de 680 na base
```

**`StockPriceHistory` não é dataset de mercado, é cache de gráfico.** Ele só é preenchido pelo `getDailySeries` quando alguém abre o gráfico de uma ação. As 10 eram as que alguém visitou.

Qualquer feature de série temporal varreria 10 ações. O Radar não era a tarefa — o backfill era.

### 90 dias, não 5 anos

Medi antes de rodar, porque o vault registra um incidente de disco cheio em junho:

| Cenário | Tamanho | vs. banco (20 MB) |
|---|---|---|
| 5 anos × 680 ações | **338 MB** | 17x |
| 90 dias × 680 ações | **24 MB** | dobra |

O Radar precisa de 31 pregões. 90 dão margem e custam 24 MB. Gráfico continua puxando 5 anos sob demanda.

> ⚠️ Depois descobri que os 338 MB **não seriam problema de capacidade** — o volume da Railway foi redimensionado para 20 GB em junho, então seria 1,7%. Eu tinha citado a restrição antiga. Os 90 dias continuam certos, mas por economia, não por medo.

### Inserção aditiva, não delete-e-recria

`createMany({ skipDuplicates: true })` em vez do `deleteMany` do `getDailySeries`. Se eu reusasse, sobrescreveria com 90 dias o histórico de 5 anos das ações que já têm gráfico carregado.

### Lista de skip, senão a fila nunca drena

8 tickers falham permanentemente (`NEOE3`, `ODPV3`, `PTNT4` dão 404 na brapi; `OIBR3` e `PCAR3` não têm 40 pregões com volume). Como a fila ordena por "menos linhas primeiro", eles voltariam em **toda** execução e consumiriam o lote inteiro para sempre. Marcados por 24h em `cache_entries`.

**Medido:** 126 de 134 ações B3 em 25s · 39 BDR em 9s.

## 2. Amplificação de escrita de 2500x no gráfico

Investigando o disco, os números de produção contaram a história:

```
16.242 linhas vivas · 35.166 inseridas · 18.913 deletadas
```

Rotacionou 2,2x o próprio tamanho **com 10 ações**. A causa é o `getDailySeries`: `DAILY_CACHE_MS` de 24h e `deleteMany + createMany` de tudo sempre que chega barra nova.

Para adicionar **1 pregão**, apagava e reescrevia **~1250 linhas**. Com 680 ações visitadas diariamente seriam **1,7 milhão de escritas por dia para adicionar 680**.

**Correção:** candle diário fechado é imutável, então o caminho normal virou `skipDuplicates`. Medido: **1292 ids preservados contra 3 linhas novas**, onde antes seriam 1295 apagadas e 1295 recriadas.

O `delete` continua existindo, mas **uma vez por semana por ação** — porque desdobramento reajusta o histórico inteiro retroativamente e o `skipDuplicates` deixaria preços antigos errados para sempre. A idade do último refresh completo sai do `MIN(updatedAt)` das próprias linhas.

> Esse churn já estava sinalizado no vault desde junho e nunca tinha sido corrigido.

## 3. Radar de Volume Anômalo

| Camada | Arquivo |
|---|---|
| Engine | `services/signals/volume-anomaly.ts` |
| Rota | `GET /api/signals/volume` |
| Letreiro | `components/app/signals-tape.tsx`, no topo do grupo `(app)` |
| Gate | `signalsMax` no `PLAN_LIMITS` — free 3, pro ilimitado |

**Compara contra 30, 60 e 90 dias**, como o Edmar sugeriu. Ele estava certo — uma janela só mente:

| | 30d | 90d | Leitura |
|---|---|---|---|
| `BRSR6` | 2,03x | **1,26x** | a média de 30 dias é que estava baixa |
| `ORVR3` | 3,19x | **3,88x** | mais anômalo do que o 30d mostra |
| `ONCO3` | 4,49x | 4,22x | consistente, anomalia real |

Sem a comparação, `BRSR6` entraria como sinal forte e é ruído de base baixa.

**Dois filtros que evitam falso positivo:**

1. **Pregão recente** (4 dias de folga para fim de semana). Sem isso o radar misturava datas — cada ação tem seu próprio último candle, e apareciam sinais de 14/08 ao lado de 18/08 como se fossem do mesmo dia.
2. **Volume financeiro mínimo de R$ 1M.** Small cap ilíquida que negocia R$ 50 mil dispara todo dia por ruído; 2x de quase nada ainda é quase nada.

**Medido:** 11 sinais em 178 ações cobertas, 2 acima de 3x.

## 4. Os dois letreiros e o bug do Tailwind

**Home (público):** o `TickerTape` já rolava, mas os valores eram array fixo no código — com bugs (`RENT3` e `RADL3` tinham porcentagem no campo de preço) sob um `aria-label` que prometia "cotações em tempo real".

Nova rota pública `GET /api/market/ticker-tape`, **deliberadamente estreita**: lista montada no servidor pelos maiores volumes, devolve só ticker, preço, variação e nome. Sem parâmetro de ticker e sem fundamentos, então não reabre a extração da base que o `guardSession` de 16/08 fechou.

> **Só B3 e BDR.** Descobri que o Twelve Data é free com **8 créditos por minuto** — a chamada com tickers US voltou `429`. Essa cota é compartilhada com gráficos e página da ação, que são features de verdade. Não vale gastar na vitrine.

### 🐛 O Tailwind v4 comia o seletor de pausa

Os dois letreiros renderizavam certo, com dados certos, e ficavam **parados**. A animação estava dentro de `@layer utilities`, e o Tailwind removeu o seletor de atributo na compilação:

```css
escrito:    .animate-ticker-tape[data-paused="true"] { animation-play-state: paused }
compilado:  .animate-ticker-tape                     { animation-play-state: paused }
```

Sem o atributo, a regra valia sempre. **Movido para fora da layer**, onde o CSS sai literal.

### 🐛 E o Windows do Marcos

Mesmo corrigido, continuou parado. Análise do CSS servido mostrou que existia **uma única** regra capaz de desligar a animação: a minha, sob `prefers-reduced-motion: reduce`.

O Windows dele está com **efeitos de animação desligados**. Isso também explica por que o letreiro original nunca rolou — tinha a mesma guarda via `matchMedia`. Foi por isso que ele pediu "fazer do tipo letreiro": não era pedido de estilo, era um bug que ele via e eu não.

**Decisão de produto do Marcos:** animar por padrão. A guarda saiu; ficam o botão de pausa e a lista `sr-only`.

## 5. Onde o letreiro de sinais mora

Marcos pediu **acima do sidebar** e **já pronto ao carregar**. O container do grupo `(app)` virou `flex-col` e o layout passou a calcular os sinais no servidor.

Como isso rodaria em toda página autenticada, a detecção foi envolvida em **cache de 30 minutos** — a consulta leva de 54 a 127ms com 178 ações e escala com a tabela. Sinal muda uma vez por dia, quando entra o candle novo.

**Tooltip no padrão do ScoreRing.** A primeira versão usou o atributo `title`, que o sistema operacional renderiza sem estilo. Refeito com `@base-ui/react/tooltip`, e o conteúdo deixou de ser texto corrido: ticker, data, quanto girou em destaque, os três múltiplos numa lista, e **uma frase que interpreta a relação entre as janelas** — exatamente a explicação que o Marcos precisou pedir para entender como se lê o sinal.

## Commits

```
f472d617  style: tooltip do letreiro no mesmo padrao do ScoreRing
5c9800ba  style: letreiro de sinais segue a linguagem visual do app
19991697  feat: tooltip com nome da empresa e contexto
a5de522a  feat: letreiro acima do sidebar e pronto no primeiro paint
7df0cf7b  fix: botao pode ligar a animacao sob reduced-motion
83eaaa16  fix: letreiro parado porque o Tailwind comia o seletor
9acc9d6f  feat: radar de volume anomalo com letreiro
95998ce1  perf: grafico diario insere so o pregao novo
1045dc0b  feat: backfill do historico diario
589a2c3b  feat: letreiro da home com cotacoes reais
```

## Lições

- **Verificar a cobertura do dado antes de projetar a feature.** O Radar parecia trivial (30 linhas de SQL) e o trabalho real era o backfill.
- **Uma janela de comparação mente.** O `BRSR6` a 2,03x sobre 30 dias e 1,26x sobre 90 é o caso didático.
- **CSS customizado não vai dentro de `@layer utilities` no Tailwind v4** se tiver seletor de atributo.
- Quando algo "não funciona" e o código parece certo, **conferir o CSS servido, não o escrito**.
