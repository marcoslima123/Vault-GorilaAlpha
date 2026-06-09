# 2026-05-20 — Sprint 01 (Hardening) concluído

← [[00 - Home]] · sessão anterior: [[Diario/2026-05-20 - Setup inicial e mapeamento]]

## Decisões tomadas (4 grandes)

1. **Monetização** = híbrido, **R$ 34,90/mês ou R$ 299/ano**, Asaas como provedor, **garantia 7d** (sem trial). Detalhes em [[Decisoes/2026-05-20 - Modelo de Monetizacao]].
2. **Mercados** = MVP com **B3 + BDR + US** (todos os 3).
3. **Análise IA** = híbrida — heurística no free, Claude Haiku 4.5 no pro. Detalhes em [[Decisoes/2026-05-20 - Mercados e IA]].
4. **Tom da análise** = **descritivo, nunca prescritivo** (risco regulatório CVM).

## Sprint 01 — entregue

Arquivos criados/modificados em `gorilaAlpha-web`:

| Arquivo | Mudança |
|---|---|
| `apps/web/src/proxy.ts` | **novo** — middleware Next.js 16 (renomeado de `middleware.ts`) valida JWT do cookie `ga_access`, redireciona `/login` se inválido |
| `apps/web/src/lib/auth-session.ts` | +`isAdmin()`, `requireAdmin()`, `ForbiddenError` |
| `apps/web/src/lib/api-guards.ts` | **novo** — helpers `guardSession()`/`guardAdmin()` para evitar repetição |
| `apps/web/src/app/api/sync/route.ts` | aplicado `guardAdmin()` em GET e POST |
| `apps/web/src/app/api/fix-markets/route.ts` | aplicado `guardAdmin()` |
| `apps/web/src/app/api/analysis/route.ts` | aplicado `guardAdmin()` (depois vira pro-only no Sprint 07) |
| `.env.example` | +`ADMIN_EMAILS` |
| `.env` | +`ADMIN_EMAILS`, +JWT secrets gerados (estavam faltando!) |
| `docker-compose.override.yml` | +`env_file: .env` (Next.js no monorepo não lia o .env da raiz) |
| `turbo.json` | +`globalEnv` listando vars a propagar para `next-server` |

## Testes (rodados via PowerShell)

| Cenário | Esperado | Resultado |
|---|---|---|
| GET `/screener` deslogado | 307 → `/login` | ✅ |
| POST `/api/sync` sem auth | 401 | ✅ |
| GET `/api/sync` sem auth | 401 | ✅ |
| POST `/api/fix-markets` sem auth | 401 | ✅ |
| GET `/api/stocks?limit=2` (público) | 200 | ❌ **500 — bug pré-existente, não relacionado ao Sprint 01** |

## Achados durante a execução (que não estavam no plano)

1. **`.env` faltava JWT secrets desde sempre** — `/api/auth/me` retornava 500 antes das minhas alterações também. Auth nunca funcionou de verdade em dev até hoje. Gerei secrets novos (48 bytes base64).
2. **Next.js 16 deprecou `middleware.ts` em favor de `proxy.ts`** — só warning, mas renomeei pra seguir convenção atual.
3. **Turbo filtra env vars** — `next-server` não recebia JWT_ACCESS_SECRET mesmo com `env_file` no compose. Precisou de `globalEnv` no `turbo.json`.
4. **`/api/stocks` quebrado** — coluna `stocks.industry` no Prisma schema não existe no banco (`P2022`). **Bloqueia o Sprint 02.**

## Backlog descoberto (prioridade alta para o Sprint 02)

- ❗ Resolver `stocks.industry` — rodar `pnpm --filter @gorila/web prisma migrate dev` ou ajustar schema. Sem isso o Screener não tem dados.
- ⚠️ Yahoo Finance lib pede Node ≥ 22 (estamos em 20). Pode quebrar endpoints de sync.
- ⚠️ Se quisermos lib opensource de pagamento brasileira, ainda precisa avaliar SDK do Asaas (Node).

## Próximo

[[Sprints/Sprint 02 - Screener Funcional]] — começar depois de resolver o `stocks.industry`.
