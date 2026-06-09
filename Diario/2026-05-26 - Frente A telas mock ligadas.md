# 2026-05-26 — Frente A: telas mock → dados reais

← [[00 - Home]] · sessão anterior: [[Diario/2026-05-26 - Sprint 07B Cron e Email de Alertas]]

## Objetivo

Remover as últimas telas placeholder pra o produto ficar redondo (sem nada "fake"). Sem dependências externas.

## O que ligou

| Tela | Antes | Agora |
|---|---|---|
| **StatsCards** (Screener) | "8.512 ativos" hardcoded | Fetch `/api/stocks/stats`: total real (671), resultados no filtro atual (prop `filteredCount`), nº de mercados, último sync |
| **SettingsProfile** | nome/email fixos ("Marco Silveira") | Nome/email reais via props do server. Editar nome → `PATCH /api/account`. Email read-only. Avatar usa inicial do nome |
| **SettingsPreferences** | theme/currency/language sem efeito | Mercados (B3/BDRs/NYSE/NASDAQ/AMEX/Cripto), setores (8 fixos), volume mínimo — todos via `/api/profile` (GET+POST). Chips toggláveis |
| **AppHeader** (busca topo) | input morto | Autocomplete real via `/api/stocks?search=`. Setas ↑↓ navegam, Enter abre `/stock/[ticker]` ou `/screener?search=`. Atalho `/` foca o campo |

## Endpoints novos

- `GET /api/stocks/stats` — total, por mercado, último sync (guardSession)
- `PATCH /api/account` — atualiza `User.name` (validação 2-80 chars)

## Reaproveitado

- `/api/profile` (GET/POST) já existia do onboarding. **Importante:** usa enums fixos no `profile-validation.ts`:
  - markets: `B3 | BDRs | NYSE | NASDAQ | AMEX | Cripto`
  - sectors: `financeiro | energia | tecnologia | saude | consumo | utilities | imobiliario | industrial`
  - minVolume: `any | 500k | 1m | 5m`
  - alerts: `email | push | both`
  - Alinhei o `SettingsPreferences` a esses valores (primeira tentativa com texto livre deu 422).

## Mudanças de página

- `/settings/page.tsx` agora busca `User{name,email}` + subscription em paralelo e passa pro `SettingsProfile`.

## Validações

| Cenário | Resultado |
|---|---|
| `/api/stocks/stats` | `{total:671, markets:{US:490,B3:146,BDR:35}, lastSync:...}` ✅ |
| `PATCH /api/account` nome válido | 200 ✅ |
| nome < 2 chars | 400 ✅ |
| `POST /api/profile` com enums válidos | 200 + persiste ✅ |
| busca header PETR | retorna PETR3/etc ✅ |

## Limpei depois do teste

- Nome revertido pra "Marcos Lima"
- Preferences zeradas (markets/sectors/minVolume vazios)

## Pendente / observações

- **Tema/moeda/idioma removidos** do SettingsPreferences — não tinham efeito (app é dark fixo, sem i18n). Honesto: só mostro o que realmente funciona. Se quiser theme switch / i18n no futuro, é feature nova.
- **Notificações 🔔 e SYNC do header** — removi os botões decorativos que não faziam nada. Sino pode voltar quando tiver centro de notificações.
- **Mercados do profile (BDRs/NYSE/NASDAQ) ≠ mercados do banco (B3/BDR/US)** — o onboarding usa uma taxonomia, o banco usa outra. Não é problema agora (preferências são só informativas), mas se um dia forem alimentar filtros do screener, precisa mapear.

## Estado geral do produto

Todas as telas principais agora usam **dados reais**. Não há mais mock visível pro usuário (exceto o "AI Sentiment" que já foi removido do Compare e o gráfico que já é real). Produto pronto pra demo.

## Próximo

- **Frente B (deploy)** pra ativar cron real + URL pública, ou
- **7.D Asaas** quando a conta validar, ou
- **Análise IA real** (Frente C)
