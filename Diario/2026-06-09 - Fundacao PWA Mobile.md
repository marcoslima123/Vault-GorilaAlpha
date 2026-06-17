# 2026-06-09 — Fundação do PWA Mobile

← [[00 - Home]] · sessão anterior: [[Diario/2026-06-09 - Paywall Feed e Asaas]] · roadmap: [[03 - Roadmap]]

## Objetivo

Tirar o `apps/pwa` do estado "esqueleto vazio" e montar a **fundação do app mobile**: estrutura que separa desktop × mobile, reuso dos serviços que já existem no web, e plumbing de PWA instalável. Telas reais virão depois via Figma (skill `implement-figma-mobile`). Marcos não tinha as credenciais novas (Asaas/FRED) neste notebook, então focamos no que não depende delas.

## Decisão de arquitetura (desktop × mobile)

- `apps/web` = desktop **+ todo o backend** (Prisma, secrets, rotas `/api/*`). `apps/pwa` = mobile, **cliente fino**.
- **Reuso de serviços = consumir os MESMOS `/api/*` por HTTP** — zero cópia de lógica de negócio. O PWA usa **rewrites do Next** (`apps/pwa/next.config.ts`, `WEB_INTERNAL_URL`, default `http://localhost:3000`) pra proxiar `/api/*` → web. Mesma-origem no browser → **cookie httpOnly de auth flui sem CORS**. Validado: `GET /api/auth/me` pelo :3001 atravessou o proxy e retornou 401 da rota de auth do web.
- Decisão do Marcos: criar o pacote compartilhado **`@gorila/core` agora**, migrando o client-side reutilizável do web de forma **incremental** (auth completo migra junto com a 1ª tela de login do Figma).

## Entregue

### Pacote `@gorila/core` (novo)
- `http.ts` — `apiFetch` (relativo `/api`, `credentials:include`, classe `ApiError`).
- `session.tsx` — `SessionProvider` + `useSession` (bate em `/api/auth/me`, status loading/authenticated/unauthenticated).
- `types.ts` — `SessionUser`, `SessionStatus`, `ApiErrorShape`.

### `@gorila/ui` (consertado + estendido)
- Tinha `iconTypes` **quebrado** (imports pendurados `@/types/iconTypes` sem alias — só não quebravam por serem `import type` apagados na compilação). Criei `packages/ui/src/types/iconTypes.ts` e corrigi os 5 ícones pra import relativo.
- Adicionei `HomeIcon`, `StarIcon`, `UserIcon`, `ChartIcon` (regra do projeto: ícones sempre da pasta compartilhada).

### PWA plumbing
- `manifest.json` (SVG-only, instalável em Chromium/Android), ícone de marca em `public/icons/icon.svg`, página `offline.html`.
- **Service worker hand-rolled** (`public/sw.js`: network-first em navegação c/ fallback offline, cache-first em estáticos) — registrado só em produção via `sw-register.tsx`. Não usei `@serwist/next` por o Next 16 ser muito novo pra arriscar no build.
- Shell mobile: `AppShell` + `BottomNav` (ícones do `@gorila/ui`), route groups `(app)` (com gate de auth via `useSession`) e `(auth)/login` (placeholder até a tela do Figma).
- Layout raiz com `SessionProvider` + registro do SW.

### Infra Docker
- `.env` na raiz (faltava — o override exige `env_file: .env`).
- Porta **3001 publicada** no `docker-compose.override.yml` + volume de `node_modules` do `core` + `WEB_INTERNAL_URL`.
- Dockerfile: adicionei `COPY packages/core/package.json` nos stages `deps` **e** `dev`.

## Caveats honestos

- **type-check verde** em `@gorila/ui`, `@gorila/core`, `@gorila/pwa`. Web e PWA sobem (Ready), smoke HTTP ok (`/`, `/login`, `/manifest.json`, proxy `/api/auth/me`).
- Vários tropeços de Docker no caminho (registrados): volumes anônimos stale exigem `--renew-anon-volumes`; o **lockfile não resolve dentro do container** (dep `baileys`→`libsignal-node` via git, imagem slim sem git) → rodar `pnpm install --lockfile-only` no **host**.
- **Ícones PNG** (melhor no iOS) ainda não gerados — manifest é SVG-only por ora.
- Auth: hoje o PWA só tem `useSession` (leitura). Login/registro mobile completos migram pro `core` com a tela do Figma.

## Bloqueio no fim da sessão

Fomos rodar `/implement-figma-mobile 58:4753` mas o **MCP `figma-desktop` não estava nesta sessão**: estava configurado no path `C:/Users/marco/OneDrive/Área de Trabalho/gorilaAlpha-web`, e estamos trabalhando em **`C:/dev/gorilaAlpha-web`** (duas cópias do projeto). Adicionei o servidor (`http://127.0.0.1:3845/mcp`) a este projeto via `claude mcp add` (status ✔ Connected), mas os tools só carregam **após reiniciar a sessão**.

## Próximo

- Reiniciar a sessão do Claude Code em `C:/dev/gorilaAlpha-web`, confirmar `figma-desktop` no `/mcp`, e rodar `/implement-figma-mobile 58:4753`.
- Implementar telas da **área logada** primeiro (Marcos ainda não definiu como representar a landing/site público).
- Eventual: gerar PNGs dos ícones; migrar stack de auth completo pro `@gorila/core`.
