# GorilaAlpha — Vault de Progresso

Vault Obsidian para mapear o progresso do projeto **GorilaAlpha** (web app de análise fundamentalista de ações B3/BDR/US).

## Mapas principais

- [[Levantamento Tecnico e de Produto (2026-08-15)]] — **levantamento completo** por leitura do código: stack, todas as features, integrações, schema, planos, gaps, métricas e mapa de rotas
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

## Backlog detalhado

- [[Backlog - Inteligencia de Mercado (ideias do socio)]] — 5 features de sinais a partir de dados públicos (B3, Tesouro, CVM)
- [[Backlog - Adversarial Verification e Tournament]] — multi-agente Bull/Bear/Quant + juiz, com score de confiança 0–100
- [[Backlog - Correlacao Macro e Fundamentos]] — macro (Selic, câmbio, commodities) × desempenho histórico por setor
- [[Backlog - Suitability (Passo 4 do Cadastro)]] — questionário de perfil de investidor no cadastro

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
- [[Diario/2026-05-29 - Feed de Relatorios de Corretoras]]
- [[Diario/2026-06-08 - Feed teste visual e wiring WhatsApp]]
- [[Diario/2026-06-09 - Modulo Renda Fixa]]
- [[Diario/2026-06-09 - Paywall Feed e Asaas]]
- [[Diario/2026-06-09 - Fundacao PWA Mobile]]
- [[Diario/2026-06-17 - PWA em producao, brapi PRO e incidente de disco]]
- [[Diario/2026-06-17 - brapi PRO 100% (token morto, graficos, MM200, Balanco)]]
- [[Diario/2026-06-18 - Worker WhatsApp em producao (Feed ao vivo)]]
- [[Diario/2026-06-19 - Watchdog WhatsApp, incidente full_text e skeletons]]
- [[Diario/2026-08-07 - Ambiente local destravado, brapi fora e cotacao separada dos fundamentos]] ⚠️ **pendência que trava o próximo deploy**
- [[Diario/2026-08-13 - Quality gate destravado, cron de sync e Renda Fixa no PWA]] ⚠️ **pendência do deploy continua aberta**
- [[Diario/2026-08-16 - Hardening de API, contrato de erro unificado e cache envenenado]] ⚠️ **migration 0.1 vira runbook; ainda não executada em prod**
- [[Diario/2026-08-17 - Validacao de ticker pela B3, limpeza aplicada e lucide removido]] — gate PASS em Node 22 · regra #2 do CLAUDE.md fechada · ⚠️ **0.1 continua o único bloqueador do deploy**
- [[Diario/2026-08-18 - IA no Ranking e arquitetura de cache]] — IA no pódio dos Rankings · **arquitetura de cache que obriga classificar o resultado**
- [[Diario/2026-08-19 - Radar de Volume Anomalo e os letreiros]] — 1ª das 5 ideias do sócio entregue · churn de escrita 2500x corrigido
- [[Diario/2026-08-20 a 21 - Deploy destravado, incidente de disco e Monitor de Derivativos]] — ✅ **deploy automático voltou** · coletor de opções rodando

## Localização do código

`C:\Users\Marcosdeli\Desktop\Projetos\GorilaAlpha\gorilaAlpha-web`

Monorepo pnpm + Turborepo:
- `apps/web` — Next.js 16.2.2 (porta 3000)
- `apps/pwa` — Next.js 16.2.2 (porta 3001)
- `packages/config` — config compartilhada
- `packages/ui` — design system (componentes, ícones, tokens)
- `packages/core` — client HTTP, sessão e tipos consumidos pelo PWA

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
- PWA: http://localhost:3001
- Swagger: http://localhost:3000/api/docs
- Postgres: localhost:5433 (user: gorila, pass: gorila, db: gorila_alpha)

> ⚠️ **Feature funciona em prod mas "some" no local?** É o **volume anônimo de `node_modules`**, que sobrevive a `up -d` e a `restart` e tem precedência sobre a imagem — carrega Prisma Client de schema antigo e dependências faltando. Rebuildar **não** resolve:
> ```powershell
> docker compose up -d --force-recreate --renew-anon-volumes app
> ```
> Detalhe do diagnóstico completo em [[Diario/2026-08-07 - Ambiente local destravado, brapi fora e cotacao separada dos fundamentos]].

> Se o banco local estiver atrás do schema, rodar `npx prisma migrate deploy` dentro do container (`apps/web`).

## Verificação antes de entregar

```powershell
docker compose exec -T app sh -c 'cd /app && pnpm gate'
```

`pnpm gate` roda prisma generate, type-check dos 4 workspaces, eslint, dependency-cruiser (fronteiras web↔pwa↔packages), auditoria do `globalEnv` do Turbo e checagem de scoring duplicado. Substitui o antigo `pnpm lint` + `pnpm type-check`.

> ⚠️ Exige **Node >= 22** — o `dependency-cruiser` roda em `^22||^24||>=26`. Em Node 20 as 3 etapas de fronteira falham e o gate dá FAIL mesmo com type-check e eslint limpos.

> O projeto **tem** vitest (`pnpm --filter @gorila/web test`) e 6 arquivos de teste, além de Stryker. O `CLAUDE.md` afirmava o contrário; corrigido em 2026-08-17.

> ⚠️ Type-check quebrando com erro de sintaxe em `.next/dev/types/routes.d.ts`? É escrita parcial do dev server, não código nosso. Apagar o arquivo e reiniciar o container.
