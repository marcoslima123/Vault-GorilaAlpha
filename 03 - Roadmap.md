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
- 🟢 Sprint 09 — Renda Fixa (BR + Internacional): **código completo em 2026-06-09** (BCB+FRED reais, comparador c/ IR+câmbio, calculadora, curva, advisor IA, 4 tabs). 🟡 falta validação visual + câmbio/US no app real. Tesouro Direto bloqueado (Cloudflare) → produtos via BCB.

## Deploy

- 🟢 **Railway (produção) — NO AR (2026-06-11)** ✅ App publicado e funcional: auth, screener, rankings, stock+gráficos (US ao vivo Twelve Data / B3 cache+brapi), dados (671 ações), billing/Asaas+webhook, **cron de alertas (GitHub Actions)**. Branch `dev`, commit `28bac920`. **Falta**: worker WhatsApp + sync agendado (mantém B3 fresco + completa ~38) + (opcional) Resend domain + magic-link. Handoff completo + env vars + como continuar em outra máquina: [[Deploy - Railway (Producao)]]

## Backlog (depois do MVP)

- **Suitability / Perfil de Investidor no cadastro** (enriquecer o Passo 4) — questionário de 13 perguntas → Conservador/Moderado/Arrojado. Ver [[Backlog - Suitability (Passo 4 do Cadastro)]]
- Importar carteira do investidor (CSV B3, integração com corretora)
- Backtest de estratégias
- Notificações in-app (não só email)
- Dashboard mobile-first (PWA: **fundação criada em 2026-06-09** — shell, navegação, sessão via proxy `/api`, `@gorila/core`; telas via Figma a seguir. Ver [[Diario/2026-06-09 - Fundacao PWA Mobile]])
- Análise IA mais profunda (já tem `@anthropic-ai/sdk`)
- Internacionalização (UserPreferences.locale já existe)
- Compartilhamento público de watchlist/análise
