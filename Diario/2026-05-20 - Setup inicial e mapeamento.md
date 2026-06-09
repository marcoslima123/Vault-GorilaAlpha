# 2026-05-20 — Setup inicial e mapeamento

Voltar para [[00 - Home]].

## O que foi feito

### Ambiente local
- Instalado **pnpm 10.33.4** via `npm install -g pnpm@10` (Node 20.11.1; pnpm 11 exigiria Node ≥22.13).
- Diagnóstico do erro `sh: 1: turbo: not found` no Docker:
  - Projeto é monorepo pnpm + Turborepo, mas o `Dockerfile` usava `npm ci` com `package-lock.json` inexistente.
  - Volumes anônimos do `docker-compose.override.yml` mascaravam `node_modules`, deixando container sem `turbo` no PATH.
- Corrigido **Dockerfile** (estágios `deps`, `builder`, `runner`, `dev`) para usar `corepack`+`pnpm install --frozen-lockfile`.
- Corrigido **docker-compose.override.yml** com volumes anônimos para cada workspace (`apps/web/node_modules`, `apps/pwa/node_modules`, `packages/*/node_modules`, `.next/`).
- Atualizado **`.dockerignore`** com `**/node_modules` e `**/.next`.
- Rebuild `--no-cache` + up: web ✓ porta 3000, pwa ✓ porta 3001 (interno), Postgres healthy.

### Pendências do ambiente
- [ ] Expor porta 3001 do PWA no `docker-compose.yml` se for usar.
- [ ] Rodar `pnpm --filter @gorila/web prisma migrate deploy` (já tem 1410 stocks; só rodar quando criar migrations novas).
- [ ] Considerar atualizar Node para ≥22.13 para usar pnpm 11.

### Mapeamento da área logada
Gerado em [[01 - Estado Atual]]:
- 7 rotas autenticadas em `(app)/*` — todas com UI esqueleto, maioria sem backend conectado.
- Auth completa: custom JWT + Google OAuth + OTP via Resend.
- 1410 stocks já populados no banco (Yahoo Finance via `/api/sync`).
- Gaps críticos: rotas sem middleware server-side, `/api/sync` aberto, stock detail com mock hardcoded.

### Recomendação
Em [[02 - Proximos Passos]]: começar por hardening de auth → Screener funcional → Stock Detail → Watchlist → Rankings → Compare → Alerts + Paywall.

## Decisões tomadas

1. **pnpm 10 (não 11)** — Node 20 não suporta pnpm 11. Caminho de upgrade para Node 22 fica como backlog.
2. **Manter custom JWT** — não migrar para NextAuth. Auth atual funciona e tem features que NextAuth pediria horas pra replicar.
3. **Conectar antes de criar** — primeiro sprint produtivo é Screener funcional (API já pronta), não construir feature nova.

## Perguntas abertas (para o produto)

- Freemium ou free + premium? Define onde colocar paywall.
- Mercados de foco no MVP: só B3 ou já incluir BDR + US?
- Score IA é heurística ou Anthropic SDK (já está nas deps)?

## Próximo passo concreto

Iniciar [[Sprints/Sprint 01 - Hardening de Auth]]: implementar `middleware.ts` e proteger `/api/sync`.
