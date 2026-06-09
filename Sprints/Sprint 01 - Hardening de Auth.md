# Sprint 01 — Hardening de Auth

**Esforço:** 1–2 dias · **Status:** ✅ concluído em 2026-05-20

← [[03 - Roadmap]] · próximo: [[Sprints/Sprint 02 - Screener Funcional]]

## Objetivo

Fechar as duas brechas de segurança visíveis hoje:
1. Rotas `(app)/*` só protegidas client-side.
2. Endpoints de admin (`/api/sync`, `/api/fix-markets`) abertos.

## Entregáveis

- [x] `apps/web/src/proxy.ts` — middleware Next.js 16 (renomeado de `middleware.ts` para `proxy.ts` por deprecação). Valida JWT do cookie `ga_access` via `jose` em edge runtime. Redireciona para `/login?from=...` se inválido.
- [x] Matcher: `/screener`, `/stock`, `/watchlist`, `/compare`, `/rankings`, `/settings`.
- [x] Helpers `isAdmin()` e `requireAdmin()` em `apps/web/src/lib/auth-session.ts` lendo whitelist `ADMIN_EMAILS` do env.
- [x] Novo `apps/web/src/lib/api-guards.ts` com `guardSession()` e `guardAdmin()` para evitar try/catch repetido nos route handlers.
- [x] `requireAdmin()` aplicado em `POST/GET /api/sync`, `POST /api/fix-markets`, `POST /api/analysis`.
- [x] `ADMIN_EMAILS` adicionado em `.env.example` e `.env` local.
- [x] Stocks endpoints (`/api/stocks`, `/api/tickers`) continuam públicos por enquanto — paywall vem no [[Sprints/Sprint 07 - Alerts e Paywall]].
- [x] Decisão: admin via whitelist por env (não via campo `role` no banco). Suficiente enquanto há 1 admin. Migração para DB-role fica como backlog se a base crescer.

## Mudanças adicionais necessárias durante a execução

- [x] **Geração de JWT secrets** — `.env` estava sem `JWT_ACCESS_SECRET` / `JWT_REFRESH_SECRET` / `NEXTAUTH_SECRET`. Auth nunca funcionou de fato antes; gerados via PowerShell.
- [x] **`env_file: .env` no `docker-compose.override.yml`** — `.env` do monorepo não era injetado no container (Next.js só lê de `apps/web/.env`).
- [x] **`globalEnv` no `turbo.json`** — Turbo filtra env vars; precisou declarar quais passar pro processo filho `next-server`.

## Critério de pronto

- ✅ Acessar `/screener` deslogado → 307 redirect para `/login?from=/screener` (sem JS, via proxy.ts).
- ✅ `POST /api/sync` sem auth → 401 `{"error":"unauthorized"}`.
- ✅ `GET /api/sync` sem auth → 401.
- ✅ `POST /api/fix-markets` sem auth → 401.
- ⏳ Admin com sessão válida consegue sync: não testado end-to-end por aqui (requer login UI); a lógica é trivial (`isAdmin()` checa `email` na whitelist de `ADMIN_EMAILS`).

## Notas técnicas (aplicadas)

- **`proxy.ts` é o novo nome do `middleware.ts` no Next.js 16.** A função exportada virou `proxy()` em vez de `middleware()`. Doc: https://nextjs.org/docs/messages/middleware-to-proxy
- O `proxy.ts` valida JWT via `jose` direto (edge-compatible). NÃO importa `auth-session.ts` porque essa cadeia traz `node:crypto` que não roda em edge.
- `jose.jwtVerify` já valida assinatura **e** `exp` claim — token expirado é tratado como inválido automaticamente.

## Backlog descoberto durante o Sprint

- ❗ **Bug pré-existente em `/api/stocks`:** Prisma referencia coluna `stocks.industry` que não existe no banco. Sintoma: 500 com `P2022`. Precisa rodar `prisma migrate dev` ou ajustar o schema. Bloqueia o [[Sprints/Sprint 02 - Screener Funcional]].
- **Yahoo Finance warning:** lib `yahoo-finance2` agora pede Node ≥ 22 (estamos em 20). Pode quebrar em algum endpoint de stocks.
- **Considerar `passThroughEnv` em vez de `globalEnv`** se essas vars começarem a invalidar cache do turbo em build.
