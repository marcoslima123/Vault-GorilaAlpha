# 2026-06-18 — Worker WhatsApp em produção (Feed ao vivo)

← [[00 - Home]] · ver [[Deploy - Railway (Producao)]] · sessão anterior: [[Diario/2026-06-17 - brapi PRO 100% (token morto, graficos, MM200, Balanco)]]

> A última feature de produto construída-mas-não-ligada entrou no ar: **o worker do WhatsApp (Baileys) está em produção na Railway e os relatórios das corretoras caem no Feed automaticamente.** Sessão de muita briga com a instabilidade do Baileys em ambiente headless.

## Resultado
✅ **Feed recebendo relatórios reais ao vivo.** 4º serviço Railway `gorila-whatsapp` rodando 24/7, monitorando o grupo das corretoras, baixando PDFs → `/api/reports/process` (web) → Claude extrai → Feed.

## Setup na Railway (como ficou)
- Serviço **`gorila-whatsapp`**: Config-as-code `/railway.whatsapp.json` (usa `Dockerfile.whatsapp`), Root Directory vazio.
- **Volume montado em `/data`** (cria pelo canvas: botão direito no serviço → Attach Volume, OU Cmd+K → "Volume"; NÃO fica em Settings). Essencial pra sessão do WhatsApp persistir entre restarts.
- **Env do worker:**
  - `NEXT_PUBLIC_APP_URL` = URL pública do web
  - `REPORTS_PROCESS_SECRET` = `fa8dccc50f818d8a6d8952f058bc997b62921d68e0ed0b0780880ef1ddea3dcd` (o MESMO no web)
  - `WHATSAPP_SESSION_PATH` = `/data/whatsapp-session`
  - `WHATSAPP_GROUP_IDS` = `120363317359033134@g.us` (VT Relatórios 5)
  - `WHATSAPP_PAIRING_NUMBER` = `5534992013939` (número dedicado do bot)
  - `WHATSAPP_HISTORY_LIMIT` = **`0`** (ver gotcha abaixo)
- **Env do web** (confirmar): `ANTHROPIC_API_KEY` (processa o PDF) + `REPORTS_PROCESS_SECRET` (mesmo do worker).

## Pareamento (lições)
- **QR em ASCII nos logs da Railway NÃO escaneia** (nem iPhone 14 Pro Max). Solução: o worker agora loga um **link `api.qrserver.com`** que renderiza um QR nítido pra abrir no navegador (commit `8782d4e`). ⚠️ manda o desafio de login pra um terceiro — risco baixo num pareamento único, mas registrado.
- **Código de pareamento** (`WHATSAPP_PAIRING_NUMBER`): o Baileys gera um código de 8 chars pra digitar em "Conectar com número de telefone" (commit `59a40cd`). Funciona, mas foi instável (ver loop abaixo); o **QR pelo link foi o que pareou de fato**.
- Códigos `515`/`503`/`408` no log são a dança normal do Baileys (515 = restart após parear). Só vira problema se entrar em loop.

## Os 3 bugs que travavam o pareamento (todos corrigidos)
1. **Sessão logout poisonava o volume** → todo restart falhava sem dar pra apagar (sem shell na Railway). Fix: worker **limpa `/data/whatsapp-session` sozinho** ao detectar logout e reconecta (commit `db53e8c`).
2. **Crash ENOENT** ao limpar a sessão (race: `saveCreds` gravava `creds.json` enquanto a pasta era apagada). Fix: remove o listener `creds.update` antes do `rm` (commit `8782d4e`).
3. **Loop infinito de logout** (o pior): o timer de `requestPairingCode` disparava de novo **a cada reconexão, inclusive após conectar** → pedir pareamento numa sessão ativa fazia o WhatsApp deslogar → auto-clear apagava a sessão boa → repetia pra sempre. Fix: flag `everConnected` + `pairingTimer` cancelado no `"open"`; pareamento só antes de conectar (commit `c6f464c`). **Esse foi o fix que estabilizou.**

## Gotcha importante: histórico/backfill
`WHATSAPP_HISTORY_LIMIT > 0` liga `syncFullHistory` no Baileys → ele tenta baixar TODO o histórico do grupo no connect → **estoura o tempo (408) e nunca estabiliza**. Mantido em **`0`** (sem backfill). Ingestão é só ao vivo. Se um dia quiser backfill, fazer com cautela (provavelmente não vale).

## Validação
Não dava pra testar postando (o Marcos não posta os PDFs). Validado esperando os relatórios reais do dia seguinte → **caíram no Feed automaticamente.** ✅

## Pendências / notas
- Bot roda com **número dedicado** (Baileys = não-oficial, risco de ban). Se deslogar/banir, reparear pelo link do QR nos logs.
- O Feed tinha 10 relatórios de seed manual; agora soma os reais que chegam.
- Plano free mostra 6, pro mostra todos.
- (Da sessão) **cache do gráfico foi religado** com guarda de cobertura (commit `14260fe`) — tinha ficado sem cache (lento) e voltou rápido sem mascarar histórico curto.
