# GorilaAlpha — Vault de Progresso

Vault Obsidian para mapear o progresso do projeto **GorilaAlpha** (web app de análise fundamentalista de ações B3/BDR/US).

## Mapas principais

- [[01 - Estado Atual]] — snapshot do que existe hoje (rotas, auth, schema, APIs, UI)
- [[02 - Proximos Passos]] — recomendação priorizada
- [[03 - Roadmap]] — sprints planejados

## Sprints

- [[Sprints/Sprint 01 - Hardening de Auth]]
- [[Sprints/Sprint 02 - Screener Funcional]]
- [[Sprints/Sprint 03 - Stock Detail Real]]
- [[Sprints/Sprint 04 - Watchlist]]
- [[Sprints/Sprint 05 - Rankings]]
- [[Sprints/Sprint 06 - Compare]]
- [[Sprints/Sprint 07 - Alerts e Paywall]]

## Decisões

- [[Decisoes/2026-05-20 - Modelo de Monetizacao]] — híbrido (free + pro), **R$ 34,90/mês ou R$ 299/ano**, Asaas como provedor, garantia 7d (sem trial)
- [[Decisoes/2026-05-20 - Mercados e IA]] — MVP com **B3 + BDR + US**, análise IA **híbrida** (heurística free / Claude Haiku 4.5 pro), tom **descritivo** (não prescritivo)

## Diário

- [[Diario/2026-05-20 - Setup inicial e mapeamento]]
- [[Diario/2026-05-20 - Sprint 01 Hardening concluido]]
- [[Diario/2026-05-20 - Sprint 02 Screener concluido]]
- [[Diario/2026-05-20 - Sprint 03 Stock Detail concluido]]
- [[Diario/2026-05-21 - Botao atualizar dados]]
- [[Diario/2026-05-21 - Sprint 04 Watchlist concluido]]
- [[Diario/2026-05-21 - Sprint 05 Rankings concluido]]
- [[Diario/2026-05-21 - Sprint 06 Compare concluido]]
- [[Diario/2026-05-21 - Sprint 3.5 Grafico Historico concluido]]
- [[Diario/2026-05-21 - Sprint 07A e 07C Alerts e Paywall]]
- [[Diario/2026-05-26 - Sprint 07B Cron e Email de Alertas]]
- [[Diario/2026-05-26 - Frente A telas mock ligadas]]
- [[Diario/2026-05-26 - Analise IA real (estrutura + fallback)]]
- [[Diario/2026-05-27 - Grafico interativo Lightweight Charts]]
- [[Diario/2026-05-27 - Inteligencia Historica MM200]]

## Localização do código

`C:\Users\Marcosdeli\Desktop\Projetos\GorilaAlpha\gorilaAlpha-web`

Monorepo pnpm + Turborepo:
- `apps/web` — Next.js 16.2.2 (porta 3000)
- `apps/pwa` — Next.js 16.2.2 (porta 3001)
- `packages/config` — config compartilhada
- `packages/ui` — design system

Stack: Next.js 16 + React 19 + Prisma 5 + Postgres 16 + pnpm 10 + Turborepo 2.

## Regras do projeto (de `CLAUDE.md`)

1. **PROIBIDO criar comentários no código** (sem `//`, `/* */`, JSDoc, TODO, FIXME)
2. **Ícones SEMPRE de `src/components/ui/icons/`** (sem lucide-react, sem SVG inline)

## Como subir o ambiente

```powershell
cd C:\Users\Marcosdeli\Desktop\Projetos\GorilaAlpha\gorilaAlpha-web
docker compose up -d
```

- Web: http://localhost:3000
- PWA: http://localhost:3001 (precisa expor a porta no compose)
- Swagger: http://localhost:3000/api/docs
- Postgres: localhost:5433 (user: gorila, pass: gorila, db: gorila_alpha)
