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

## 🔴 Bloqueios abertos (atualizado 2026-08-15)

1. **O próximo deploy não sobe** — migration `full_text` não registrada no `_prisma_migrations` de prod (confirmado por consulta). Rodar `prisma migrate resolve --applied 20260618120144_add_report_full_text` **antes** de deployar. Ver [[Deploy - Railway (Producao)]]. 🔴 **ainda aberto**
2. **Preços desatualizados em produção** — 🟡 o sync agendado **foi implementado** (`POST /api/cron/sync-stocks` + `.github/workflows/sync-stocks.yml`, a cada 30min), mas **só resolve depois do deploy**, que depende do item 1.
3. **Trabalho não commitado** — 🔴 **cresceu.** Agora acumula duas sessões: as 3 mudanças de cotação + ScoreRing (07/08) **mais** quality gate, cron de sync, detalhe de relatório no PWA e Renda Fixa no PWA (13–15/08).

## Paridade PWA ↔ Web (2026-08-15)

O PWA tem Feed (home), Screener, Stock, Watchlist, Compare, Alertas, Perfil e **Renda Fixa**. Falta **Rankings** — única tela que existe no web e não no mobile. A API `/api/rankings` já está pronta: é wiring, não feature nova.

## Deploy

- 🟢 **Railway (produção) — NO AR** ✅ App publicado e funcional: auth, screener, rankings, stock+gráficos, billing/Asaas+webhook, cron de alertas (GitHub Actions). **(2026-06-17)** B3/BDR 100% via brapi PRO (preço/intraday/5y/fundamentos), Yahoo só pra US. **(2026-06-18) Worker WhatsApp NO AR** → Feed recebendo relatórios das corretoras ao vivo (4º serviço `gorila-whatsapp`). **Falta (opcional)**: sync agendado (cron na Railway p/ B3/BDR), Resend domain, magic-link. Handoff completo: [[Deploy - Railway (Producao)]]

## Backlog (depois do MVP)

- **Suitability / Perfil de Investidor no cadastro** (enriquecer o Passo 4) — questionário de 13 perguntas → Conservador/Moderado/Arrojado. Ver [[Backlog - Suitability (Passo 4 do Cadastro)]]
- ~~**Sync agendado** dos fundamentos~~ → ✅ implementado em 2026-08-14 (rota rotativa + GitHub Actions a cada 30min). Só entra em vigor depois do deploy.
- **Limpeza das linhas antigas de `stock_indicators`** — o churn foi estancado (update-or-create), mas prod tem ~1300 linhas históricas paradas
- **Auditar cache envenenado nos outros services de renda fixa** — `usa`, `international` e `fx` usam o mesmo `cached()` que gravava fallback de falha como dado bom. O `brazil` foi corrigido em 2026-08-14 (zerava o IPCA por 1h e subestimava todo produto IPCA+); os outros três **não foram auditados**.
- **Padronizar formato de erro das rotas** — as rotas devolvem `{ error: "codigo", message }` mas o `apiFetch` do `@gorila/core` espera `{ error: { code, message } }`, então o PWA sempre mostra mensagem genérica.
- **Fila e SSE do Feed são in-process** (`globalThis` + `EventEmitter`) — só funciona com uma instância do web. Trava escala horizontal.
- **Apertar a validação de ticker no extrator do Feed** — está criando entradas de menções soltas (`AZZAS2154`, `AUGO`, `PICS`) e consensos com contagem zero
- Importar carteira do investidor (CSV B3, integração com corretora)
- Backtest de estratégias
- Notificações in-app (não só email)
- Dashboard mobile-first (PWA: **fundação criada em 2026-06-09** — shell, navegação, sessão via proxy `/api`, `@gorila/core`; telas via Figma a seguir. Ver [[Diario/2026-06-09 - Fundacao PWA Mobile]])
- Análise IA mais profunda (já tem `@anthropic-ai/sdk`)
- Internacionalização (UserPreferences.locale já existe)
- Compartilhamento público de watchlist/análise

### Inteligência de Mercado (ideias do sócio, capturadas 2026-07-07)

- **5 features de sinais a partir de dados públicos** (B3, Tesouro, CVM): Radar de Volume Anômalo, Monitor de Derivativos, Monitor de Leilões do Tesouro, Radar de Fundos (CVM), Volatilidade Implícita. Detalhe completo + prioridades em [[Backlog - Inteligencia de Mercado (ideias do socio)]].

### Análise avançada (detalhado 2026-08-15)

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
