# Backlog — Suitability / Perfil de Investidor (enriquecer o Passo 4 do cadastro)

> Status: **ideia / a implementar.** Capturado em 2026-06-10 a pedido do Marcos.
> Contexto: o cadastro hoje vai `registrar → perfil → preferências (Mercados, passo 3/4) → verificar-email → /bem-vindo`. A ideia é transformar/enriquecer o **passo 4** num **questionário de suitability** (Análise de Perfil do Investidor / API) que classifica o usuário em **Conservador / Moderado / Arrojado** e personaliza a experiência (e atende boa prática regulatória).
> Ligações: [[Sprints/Sprint 07 - Alerts e Paywall]] · o `UserPreferences` já tem `experience`, `goal`, `patrimony` (hoje preenchidos no passo "perfil").

## Como deve influenciar

- Gerar um **perfil de risco** (conservador/moderado/arrojado) a partir das respostas.
- Personalizar recomendações, o tom da IA e os destaques (ex: conservador → mais renda fixa; arrojado → renda variável/derivativos).
- Persistir junto do `UserPreferences` (talvez um JSON `suitability` ou campos novos + a classificação final).

---

## Bloco 1 — Perfil de risco (atitude)

**1. Qual o seu pensamento na hora de investir?**
- **Risco** — Priorizo o rendimento mais alto, ainda que isso represente mais risco para minha carteira.
- **Equilíbrio** — Prefiro as opções que equilibrem melhor risco e retorno.
- **Segurança** — Busco primeiro segurança, mesmo que para isso eu precise abrir mão de rendimentos maiores.

**2. O que é mais importante para você na hora de investir?**
- Rentabilidade e Diversificação
- Segurança e Tranquilidade

**3. Como você classificaria seu nível de conhecimento sobre investimentos?**
- Nenhum
- Básico — Conheço o mercado de renda fixa e fundos.
- Intermediário — Entendo algo sobre o mercado de renda variável e de derivativos.
- Avançado — Sou experiente no mercado de renda variável e de derivativos.

**4. Se algo inesperado acontecer e seus investimentos se desvalorizarem, o que você faria?**
- **Baixa tolerância a risco** — Venderia imediatamente: não quero me expor ao risco de ativos muito inconstantes no curto prazo.
- **Média tolerância a risco** — Entendo que corro esse risco para determinados ativos, mas não para todo o meu patrimônio.
- **Alta tolerância a risco** — Entendo que meu patrimônio está sujeito a flutuações dessa magnitude e não está 100% protegido.

**5. Por quanto tempo você pretende deixar seu dinheiro investido aqui?**
- Menos de 1 ano.
- De 1 a 3 anos.
- De 3 a 5 anos.
- Mais de 5 anos.

**6. Quais investimentos você já fez?** (múltipla escolha)
- Nunca investi
- Poupança
- Previdência Privada
- Títulos de renda fixa
- Fundos de investimento
- Bolsa de valores e Derivativos
- Cripto

---

## Bloco 2 — Experiência por classe de ativo

> Padrão por classe: **(a)** valor investido na classe + **(b)** "no último ano você investiu em X?" (Sim/Não).

**7. Fundos** — Multimercado, Renda Fixa, Ações, Exclusivos, FII, ETF e outros fundos.
- Qual valor você tem investido nesses tipos? (faixa/valor)
- No último ano você investiu em fundos? Sim / Não

**8. Ações** — Ações de empresas.
- Qual valor você tem investido? (faixa/valor)
- No último ano você investiu em Ações? Sim / Não

**9. Renda Fixa** — Títulos Públicos, CDB, Debêntures, Letras de Câmbio/Financeira, LCI/LCA, CRI/CRA.
- Qual valor você tem investido? (faixa/valor)
- No último ano você investiu em Renda Fixa? Sim / Não

**10. Previdência Privada** — PGBL e VGBL.
- Qual valor você tem investido? (faixa/valor)
- No último ano você investiu em Previdência? Sim / Não

**11. Derivativos** — Opções, Futuros, Operações a Termo, Empréstimo de Títulos e COE.
- Qual valor você tem investido? (faixa/valor)
- No último ano você investiu em Derivativos? Sim / Não

---

## Bloco 3 — Liquidez e formação

**12. Sobre seus recursos investidos em outras plataformas, quando você pretende utilizá-los?**
- **Alta necessidade de liquidez** — Nos próximos 6 meses.
- **Média necessidade de liquidez** — Em até 12 meses.
- **Baixa necessidade de liquidez** — Não tenho previsão de utilizar os recursos.

**13. Qual sua formação acadêmica?**
- Ensino Fundamental
- Ensino Médio
- Ensino Superior
- Pós-graduação, Mestrado ou Doutorado

---

## Notas de implementação (quando for fazer)

- É um questionário longo → considerar **multi-step / progressivo** (não tudo numa tela) ou um passo 4 dedicado com seções.
- As faixas de valor dos Bloco 2 precisam de opções definidas (ex: "até R$ 5 mil", "R$ 5–50 mil", "R$ 50–250 mil", "acima de R$ 250 mil") — alinhar com o Marcos.
- Calcular a **pontuação → perfil** (Conservador/Moderado/Arrojado) com pesos por resposta (definir regra).
- Persistir em `UserPreferences` (novo campo `suitability Json?` + `riskProfile String?`) → exigiria migration.
- Reaproveitar o estilo do `register-preferences-screen` (pills/multi-pills) e o `StepIndicator`.
