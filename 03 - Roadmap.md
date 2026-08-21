# Roadmap

Visão consolidada dos sprints. Detalhes em cada nota.

Voltar para [[00 - Home]] · ver [[02 - Proximos Passos]].

## Linha do tempo (estimativa)

```
Semana 1     [Sprint 01 — Hardening] [Sprint 02 — Screener]
Semana 2     [Sprint 03 — Stock Detail] [Sprint 04 — Watchlist]
Semana 3     [Sprint 04 cont.] [Sprint 05 — Rankings]
Semana 4     [Sprint 06 — Compare] [Sprint 07 — Alerts início]
Semana 5+    [Sprint 07 cont. — Alerts + Paywall]
```

## Sprints

| # | Sprint | Esforço | Bloqueia |
|---|---|---|---|
| 01 | [[Sprints/Sprint 01 - Hardening de Auth]] | 1–2d | nada |
| 02 | [[Sprints/Sprint 02 - Screener Funcional]] | 2–3d | 04, 05 |
| 03 | [[Sprints/Sprint 03 - Stock Detail Real]] | 2–3d | 04, 06 |
| 04 | [[Sprints/Sprint 04 - Watchlist]] | 3–4d | 07 |
| 05 | [[Sprints/Sprint 05 - Rankings]] | 2d | — |
| 06 | [[Sprints/Sprint 06 - Compare]] | 2d | — |
| 07 | [[Sprints/Sprint 07 - Alerts e Paywall]] | 5–7d | — |
| 08 | [[Sprints/Sprint 08 - Feed de Relatorios de Corretoras]] | ~1d | — |
| 09 | [[Sprints/Sprint 09 - Renda Fixa]] | ~1d | — |

**Total estimado:** ~4–5 semanas para chegar em MVP usável + monetizável.

## Status

- [x] Sprint 01 — concluído em 2026-05-20
- [x] Sprint 02 — concluído em 2026-05-20 (após reset + repopular DB)
- [x] Sprint 03 — concluído em 2026-05-20
- [x] Sprint 04 — concluído em 2026-05-21
- [x] Sprint 05 — concluído em 2026-05-21
- [x] Sprint 06 — concluído em 2026-05-21
- [x] Sprint 3.5 — Gráfico histórico (concluído em 2026-05-21)
- 🟢 Sprint 07 — Alerts + Paywall: 7.A/7.B/7.C OK · **7.D (Asaas) + 7.E (garantia 7d) implementados em 2026-06-09** (checkout testado no sandbox, webhook, reembolso, paywall do Feed). Falta: webhook em URL pública + reiniciar dev p/ `.env` corrigida. Ver [[Diario/2026-06-09 - Paywall Feed e Asaas]]
- [x] Sprint 08 — Feed de Relatórios: **CONCLUÍDO end-to-end em 2026-06-08** (WhatsApp → PDF → Claude → feed funcionando; 3 PDFs reais processados). App + ConsensusWidget testados.
- 🟢 Sprint 09 — Renda Fixa (BR + Internacional): **código completo em 2026-06-09** (BCB+FRED reais, comparador c/ IR+câmbio, calculadora, curva, advisor IA, 4 tabs). 🟡 falta validação visual + câmbio/US no app real. Tesouro Direto bloqueado (Cloudflare) → produtos via BCB. **(2026-08-15) Portado para o PWA** com 5 abas (Visão, Calculadora, Comparar, Histórico, Análise IA), abas em rolagem horizontal e máscara de real. Ver [[Diario/2026-08-13 - Quality gate destravado, cron de sync e Renda Fixa no PWA]].

## ✅ Bloqueios de deploy — todos resolvidos (2026-08-21)

1. ~~Migration `full_text` travando o boot~~ → resolvida em 17/08.
2. ~~Start Command chamando `start` em vez de `start:railway`~~ → corrigido em 18/08.
3. ~~Watch Paths descartando todo push como *skipped*~~ → **campo apagado em 20/08**. Era a causa real de nenhum deploy sair.
4. ~~Limpeza de tickers órfãos em produção~~ → aplicada em 18/08.

> 🎯 O deploy de 20/08 foi o **primeiro com migration pendente desde as correções**, e a `drop_fixed_income_cache` aplicou sozinha no boot. Ciclo fechado de ponta a ponta. Detalhe e checklist de diagnóstico em [[Deploy - Railway (Producao)]].

### Ainda em aberto

- 🟡 **Credenciais expostas** — rotação adiada por decisão. Gatilho: antes de dar acesso a outra pessoa.
- 🔧 **VHDX do Docker com 56 GB reservados** — compactar exige PowerShell como administrador.
- 🔧 `Custom Build Command` da Railway está preenchido e sendo ignorado; quebra o build se um dia for honrado.

## ✅ Padrão recorrente RESOLVIDO: cache que gravava falha como dado bom

**Quatro ocorrências da mesma classe de bug**, em quatro sessões seguidas:

| Data | Onde | O que era cacheado |
|---|---|---|
| 14/08 | `fixed-income/brazil.service.ts` | `ipca12m ?? 0` por 1h → subestimava todo produto IPCA+ |
| 16/08 | `usa`, `international`, `fx`, `history` services | listas vazias **e parciais** por 1h–6h |
| 17/08 | `providers/tickerList.ts` | `[]` por 24h → rebaixaria todo relatório entrante |
| 18/08 | `technical-insights` | `touchCount: 0` sem histórico suficiente, por 24h |

✅ **Resolvido na raiz em 2026-08-18** com `lib/cache.ts`. O fetcher passa a retornar `FetchOutcome<T>` (`fresh` · `degraded` · `failed`) em vez de `T` cru — **gravar sem classificar não compila**. O helper `byCoverage` trata resposta parcial, e a tabela `cache_entries` guarda `status`, `source`, `coverage` e `retryAfter`. Detalhe em [[Diario/2026-08-18 - IA no Ranking e arquitetura de cache]].

**Ainda pendente:** levar `status`/`stale` até a tela da renda fixa (a cotação já faz isso com `priceStale`), e dar `DROP` na tabela `fixed_income_cache`, que ficou órfã.

## Paridade PWA ↔ Web (atualizado 2026-08-18)

O PWA tem Feed (home), Screener, Stock, Watchlist, Compare, Alertas, Perfil e **Renda Fixa**. Falta **Rankings** — única tela que existe no web e não no mobile, e que agora também tem a justificativa por IA no pódio.

🟡 **Decisão de 2026-08-18:** entra quando o Marcos ajustar os menus do app. Não é pendência técnica — a API `/api/rankings` já está pronta.

## Deploy

- 🟢 **Railway (produção) — NO AR** ✅ App publicado e funcional: auth, screener, rankings, stock+gráficos, billing/Asaas+webhook, cron de alertas (GitHub Actions). **(2026-06-17)** B3/BDR 100% via brapi PRO (preço/intraday/5y/fundamentos), Yahoo só pra US. **(2026-06-18) Worker WhatsApp NO AR** → Feed recebendo relatórios das corretoras ao vivo (4º serviço `gorila-whatsapp`). **Falta (opcional)**: sync agendado (cron na Railway p/ B3/BDR), Resend domain, magic-link. Handoff completo: [[Deploy - Railway (Producao)]]

## Backlog (depois do MVP)

- **Suitability / Perfil de Investidor no cadastro** (enriquecer o Passo 4) — questionário de 13 perguntas → Conservador/Moderado/Arrojado. Ver [[Backlog - Suitability (Passo 4 do Cadastro)]]
- ~~**Sync agendado** dos fundamentos~~ → ✅ implementado em 2026-08-14 (rota rotativa + GitHub Actions a cada 30min). Só entra em vigor depois do deploy.
- **Limpeza das linhas antigas de `stock_indicators`** — o churn foi estancado (update-or-create), mas prod tem ~1300 linhas históricas paradas
- ~~**Auditar cache envenenado nos services de renda fixa**~~ → ✅ os 4 restantes (`usa`, `international`, `fx`, `history`) auditados em 2026-08-16. A varredura do resto do código foi feita em 18/08 (achou `technical-insights`) e o padrão foi resolvido na raiz. Ver acima.
- ~~**Padronizar formato de erro das rotas**~~ → ✅ feito em 2026-08-16: `ApiErrorCode` de 7 → 27 códigos, 31 rotas e 97 chamadas de `jsonError`. Zero `NextResponse.json({ error: … })` solto no `app/api`.
- ~~**Apertar a validação de ticker no extrator do Feed**~~ → ✅ feito em 16/08 e **corrigido em 17/08** — validava contra a tabela `stocks` e rebaixava relatórios de empresas reais não sincronizadas. Agora valida contra a lista da B3, com fail open.
- **Fila e SSE do Feed são in-process** (`globalThis` + `EventEmitter`) — só funciona com uma instância do web. Trava escala horizontal.
- 🔴 **Rotacionar credenciais expostas** — `RESEND_API_KEY` commitada em `.env.example:21` e a URL do Postgres de prod, que já circulou em texto plano várias vezes.
- Importar carteira do investidor (CSV B3, integração com corretora)
- Backtest de estratégias
- Notificações in-app (não só email)
- Dashboard mobile-first (PWA: **fundação criada em 2026-06-09** — shell, navegação, sessão via proxy `/api`, `@gorila/core`; telas via Figma a seguir. Ver [[Diario/2026-06-09 - Fundacao PWA Mobile]])
- Análise IA mais profunda (já tem `@anthropic-ai/sdk`)
- Internacionalização (UserPreferences.locale já existe)
- Compartilhamento público de watchlist/análise

### Inteligência de Mercado (ideias do sócio, capturadas 2026-07-07)

- ✅ **Radar de Volume Anômalo** — entregue em 2026-08-19. Letreiro no topo da área logada, compara 30/60/90 dias, gate por plano.
- 🟢 **Monitor de Derivativos** — fases 1 e 2 entregues em 20–21/08 (coletor diário + painel na página da ação). **Fase 3 (o sinal de anomalia) depende só de tempo:** a chain da brapi não tem histórico, então a série é acumulada um pregão por vez. Coleta começou em 20/08; média de 30 dias por volta de **outubro**.
- 🟢 **Monitor de Leilões do Tesouro** — não iniciado, fonte pública disponível.
- 🟢 **Radar de Fundos (CVM)** — não iniciado, dados abertos disponíveis.
- 🟡 **Volatilidade Implícita** — sem bloqueio de fonte: a chain traz strike, vencimento e preço, e a Selic vem do módulo de Renda Fixa. Falta o cálculo (Black-Scholes).

Detalhe completo em [[Backlog - Inteligencia de Mercado (ideias do socio)]].

### Análise avançada (detalhado 2026-08-15)

- ✅ **Análise comparativa por IA no Comparador** — entregue em 2026-08-17 (web + PWA, feature Pro). Cruza fundamentos + consenso de corretoras de 2–4 ativos e descreve perfil de tese, se são concorrentes ou complementares, e os trade-offs. Cache de 7 dias com chave normalizada por ordem. É um **primeiro passo na direção do Tournament** abaixo: mesma família de "IA que compara teses", só sem o pipeline adversarial.

- **Adversarial Verification + Tournament** — Bull vs Bear vs Quant analisam em paralelo, se atacam, e um juiz sintetiza o veredito com **score de confiança 0–100** baseado em quanto a tese resistiu ao ataque cruzado. Feature Pro: multiplica tokens (~10 chamadas por ação) mas eleva a qualidade exponencialmente. Detalhe completo em [[Backlog - Adversarial Verification e Tournament]].
- **Correlação Macro + Fundamentos** (ideia do sócio) — cruzar Selic, câmbio e commodities com o desempenho histórico por setor: *"nas últimas X vezes que o açúcar ficou abaixo de $Y, sucroenergéticas caíram Z%"*. Fontes BCB + FRED + histórico de ações, **todas já em produção** por conta da Renda Fixa. Detalhe completo em [[Backlog - Correlacao Macro e Fundamentos]].

### Plataforma / Negócio (mapeado)

- Agregador Multi-Banco (Pluggy)
- Academia GorilaAlpha (vídeos educativos)
- Feed de Preços-Alvo (scraping público)
- Chat com Relatórios (cruzamento com o score)
- Portfólio pessoal do usuário
- Push notifications
- Rename da plataforma (BRX ou Verum)
- Lançamento Beta fechado
