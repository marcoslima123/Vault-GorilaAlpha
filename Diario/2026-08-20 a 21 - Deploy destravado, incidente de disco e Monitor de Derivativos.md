# 2026-08-20 e 21 — Deploy destravado, incidente de disco e o Monitor de Derivativos

← [[00 - Home]] · sessão anterior: [[Diario/2026-08-19 - Radar de Volume Anomalo e os letreiros]] · ver [[Deploy - Railway (Producao)]]

> Dois dias com três coisas: **o deploy automático voltou a funcionar** depois de meses quebrado, um **incidente de disco cheio** que eu causei, e as **fases 1 e 2 do Monitor de Derivativos**.
>
> A caçada ao deploy custou horas e passou por três hipóteses erradas antes da certa. O registro do caminho vale mais que o resultado — está em [[Deploy - Railway (Producao)]] como checklist ordenado.

---

## 1. A caçada ao deploy

14 commits pushados, **nenhum deploy**. O app continuava servindo o build de 18/08.

### Hipótese 1: o CI está vermelho por causa dos testes ❌ (parcialmente certo)

Eu tinha quebrado 6 testes com a mudança do consenso: o mock do Prisma não tinha `deleteMany` e dois casos afirmavam o comportamento antigo. Real, e corrigido — mas não era a causa.

### Hipótese 2: o build está quebrado ❌ (falso positivo caro)

`pnpm build` falhava nos dois apps:

```
TypeError: Cannot read properties of null (reading 'useContext')
Export encountered an error on /_global-error/page
```

Testei em `e84c4e81`, anterior à sessão: falha igual. Descartei React duplicado (uma cópia só), `SessionProvider`, `SwRegister` e o layout — removendo cada um. Fornecer `global-error.tsx` explícito não resolveu.

**A prova definitiva veio do Docker:**

```
docker build -f Dockerfile .      → pnpm build:web  DONE 25.7s
docker build -f Dockerfile.pwa .  → pnpm build:pwa  DONE 13.9s
```

**Os dois passam em ambiente limpo.** A falha era o `node_modules` do meu container de dev estar velho — provei achando `lucide-react` ainda instalado, dependência removida dias antes.

> ⚠️ **Rodar `pnpm build` no container de dev dá falso positivo.** O `--renew-anon-volumes` restaura o `node_modules` da imagem, que fica atrás do lockfile. Só o `docker build` faz `pnpm install` limpo e é prova confiável. Isso me custou mais de uma hora.

### Hipótese 3: Wait for CI segurando ⚠️ (contribuía)

O `backfill-history` rodava de hora em hora e **falhava com 404**, porque a rota não estava publicada. Circular: o cron falha porque não deployou, e não deploya porque o cron falha. Corrigido — 404 virou warning com saída zero.

### ✅ A causa real: Watch Paths

O que resolveu foi Marcos olhar a aba Deployments e dizer que havia *"histórico de **skipped**, nada de mais"*.

**"Skipped" era a resposta.** A Railway descarta o deploy quando o push não casa com o Watch Paths. O serviço web tinha:

```
/apps/web/**
```

**Com barra no início.** A sintaxe é estilo gitignore e a barra impedia o casamento, fazendo todo push ser descartado **antes de buildar**.

E mesmo com a sintaxe certa, o padrão estaria errado: o web depende de `packages/core` e `packages/ui` via `transpilePackages`, mais `pnpm-lock.yaml` e `Dockerfile`. Mudança em `packages/core` não deployaria.

**Correção: campo apagado.**

### 🎯 O resultado que fecha o ciclo

O primeiro deploy depois disso foi o **primeiro teste real do `start:railway`** corrigido em 18/08 com migration pendente:

```
prisma migrate status → Database schema is up to date! (18/18)
fixed_income_cache    → removida sozinha no boot
```

O problema que travava deploy desde junho está resolvido de verdade, não por acaso.

## 2. Limpezas

**Consenso vazio** — `recalculateConsensus` criava linha mesmo sem nenhuma recomendação nem preço-alvo, deixando registros mortos como o `ADSK` em produção. Agora apaga em vez de gravar. Exceção guardada: se há preço-alvo sem rating, a linha vale — alvo sozinho ainda é informação.

**`fixed_income_cache` removida** — órfã desde que o contrato de cache passou a usar `cache_entries`. Migration escrita à mão porque o Prisma bloqueia drop destrutivo em modo não-interativo, **com `DROP TABLE IF EXISTS`** — lição direta do `full_text`, onde uma migration sem guarda travou o deploy por dois meses.

## 3. 🚨 Incidente: disco cheio e 1.717 processos órfãos

Os dois `docker build` do diagnóstico inflaram o VHDX. O C: chegou a **0 bytes**.

Com o disco cheio o daemon travou, e aí veio o efeito que eu não previ: **meus loops de retry não tinham timeout no comando interno**. Cada iteração de `for i in $(seq 1 90); do docker ...` deixou um processo pendurado.

```
processos docker orfaos: 1717
```

Sozinhos sufocavam a máquina — nem `docker ps` respondia.

**Recuperação:** Marcos liberou ~14 GB (npm cache 9,3 GB + Temp 4,4 GB), `wsl --shutdown`, matar os 1717 zumbis, reiniciar o Docker Desktop. **Nada foi perdido** — o volume `pgdata` sobreviveu com as 671 ações, 36 mil barras de histórico e as 69 linhas de opções.

> 📌 **Regra que fica:** todo comando dentro de loop de retry leva `timeout N`. E build Docker neste projeto exige `docker builder prune` logo depois.
>
> 🔧 **Pendente:** o VHDX segue com 56 GB reservados. Compactar exige PowerShell **como administrador**:
> ```
> wsl --shutdown
> diskpart → select vdisk file="...\docker_data.vhdx" → attach vdisk readonly → compact vdisk → detach vdisk
> ```

## 4. Monitor de Derivativos — fases 1 e 2

### Fase 1: o coletor

Não dá para fazer backfill: **a chain da brapi devolve o dia corrente e não existe endpoint histórico**. A média de 30 dias que o Edmar pediu só existe depois de ~30 pregões coletando — cerca de 6 semanas. Cada dia sem coletor empurra o primeiro sinal em um dia, então a Fase 1 veio primeiro mesmo sem tela.

| Peça | Arquivo |
|---|---|
| Provider | `services/providers/brapiOptions.ts` |
| Coletor | `services/signals/option-collector.ts` |
| Cron | `POST /api/cron/collect-options` |
| Agendamento | `collect-options.yml` — 18h40 BRT, dias úteis |
| Tabela | `option_volumes`, uma linha por ativo por pregão |

**Só os 3 vencimentos mais próximos.** Medido: `PETR4` tem R$ 76,6M no primeiro contra R$ 8,5M no segundo e R$ 1,1M no terceiro. São 4 chamadas por ativo em vez de 34.

**Guarda o strike concentrado, não só o total** — é o que distingue "muito volume" de "alguém montando posição".

**Medido:** 80 ativos em 52s, 69 gravados, 11 pulados por não terem opções relevantes, R$ 511M somados.

### 🐛 O bug que quase estragou a série

A primeira versão gravou `ABEV3` em **12/08** e `VALE3` em **20/08** no mesmo lote.

O campo `date` de cada contrato é o **último pregão em que aquele contrato negociou**, não a sessão atual. Eu somava volumes de dias diferentes numa linha só e arquivava sob data arbitrária.

Corrigido: soma **só os contratos da sessão mais recente** da chain. Se isso tivesse passado, a média de 30 dias seria construída sobre lixo — e só apareceria daqui a seis semanas.

### Fase 2: o painel na página da ação

Quatro métricas que já funcionam com um único pregão:

| | Opções | Put/call | Concentração | ÷ à vista |
|---|---|---|---|---|
| `ITSA4` | R$ 25,6M | **39,4** | **93%** put 13,75 | 0,10x |
| `PETR4` | R$ 92,4M | 0,14 | 5% call 34,61 | 0,05x |
| `VALE3` | R$ 53,0M | 0,91 | 10% call 72,20 | 0,03x |

O contraste `ITSA4` × `PETR4` é o ponto do Edmar em números: **93% num strike único versus 5% espalhado**. Quem monta posição concentra; quem faz hedge de carteira espalha.

Quando não há histórico, o painel diz isso e o tooltip explica que a B3 não publica volume passado de opções.

### 🐛 Três bugs achados testando a Fase 2

1. **O backfill nunca atualizava série já preenchida** — só olhava ações com menos de 40 barras. Isso **pararia o próprio Radar de Volume em quatro dias**, porque ele exige pregão recente: o letreiro esvaziaria sozinho sem ninguém entender por quê.
2. **Minha primeira correção foi circular** — usei como referência o pregão mais recente da própria tabela. Se ninguém avança, a referência nunca avança. Trocado por âncora no calendário, pulando fim de semana.
3. **O pareamento com o volume à vista falhava sempre** — barras de preço gravadas às **03:00 UTC** (meia-noite de Brasília) contra data das opções em `Date` puro às **00:00 UTC**. Comparava timestamp em vez de dia.

Depois das correções: 133 ações atualizadas, 654 linhas novas.

## Commits

```
41310a8e  feat: painel de derivativos na pagina da acao (fase 2)
e2fe6ad1  feat: coletor diario de volume de opcoes (fase 1)
68dff6e2  ci: 404 nos crons deixa de ser falha
bc4ff670  test: corrige os testes de consenso e adiciona telas de erro
4979897a  chore: consenso vazio nao e gravado e tabela orfa removida
```

## Estado

- Produção com 18/18 migrations, deploy automático funcionando
- Coletor de opções rodando diariamente — **relógio das 6 semanas começou em 20/08**
- Local: 671 ações, 36 mil barras, 69 linhas de opções

## Próximo

1. **Fase 3 do Monitor de Derivativos** — só depende do tempo. Em ~30 pregões há média para comparar e o sinal liga no letreiro. A engine é a mesma do Radar de ações.
2. **Compactar o VHDX** (precisa de admin).
3. Esvaziar o `Custom Build Command` da Railway (está sendo ignorado, mas quebra o build se um dia for honrado) e preencher `Healthcheck Path` com `/api/ping`.
4. **Rankings no PWA** — aguardando o Marcos ajustar os menus.
5. Tier 1 restante: Monitor de Leilões do Tesouro e Radar de Fundos (CVM), ambos 🟢 alta e com fonte pública.
