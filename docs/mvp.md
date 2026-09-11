# Controlly — MVP

> O que entra, o que fica de fora, e por quê.
> Produto em [visao-produto.md](visao-produto.md) · Técnico em [arquitetura.md](arquitetura.md)

---

## 1. A premissa: a conexão bancária é o core

Uma versão anterior deste documento propunha um MVP de cadastro manual, deixando a conexão para depois. **Estava errado, e por um motivo específico.**

Um MVP existe para testar a suposição mais arriscada. Este projeto tem duas:

| Suposição | Se for falsa | Custo de descobrir tarde |
|---|---|---|
| Ver o comprometimento futuro muda decisão | O produto não tem razão de existir | Alto |
| **Dá para extrair parcelas do Open Finance de forma confiável** | **Não existe produto — só uma planilha com login** | **Fatal** |

A segunda é a mais cara de errar, e é a única que não tem plano B. Um MVP manual testaria só a primeira — e ainda por cima mal, porque poderia falhar por atrito de digitação em vez de por a tese ser falsa.

Some a isso o fato de ser SaaS: **ninguém paga por app de digitação manual.** A disposição a pagar mora na automação. Um MVP sem conexão não é um produto menor — é outro produto.

**Portanto: o MVP é "conecte seu cartão e veja seus próximos 12 meses".**

---

## 2. A primeira coisa a fazer, antes de qualquer código de produto

> **Um spike de dados no sandbox de um agregador. Prazo: dias, não semanas.**

A arquitetura registra uma suposição técnica que nunca foi verificada: *"Open Finance não entrega o cronograma completo de parcelas de forma confiável — o Controlly precisa inferir e reconciliar."*

Essa frase é o produto inteiro apoiado numa hipótese. Verifique antes de construir em cima dela.

**O que o spike precisa responder:**

1. As transações de cartão vêm com identificação de parcela? Em campo estruturado, ou só na descrição?
2. Qual a **janela de histórico** disponível? Determina quantos compromissos antigos você consegue reconstruir
3. Uma compra em 10x aparece como **um** registro com cronograma, ou como parcelas soltas entrando a cada fatura?
4. A descrição é suja o bastante para exigir normalização? Quão suja?
5. Faturas futuras já fechadas aparecem?

Sandbox costuma ter tier gratuito e não exige contrato. **Isso derruba o maior risco do projeto por alguns dias de trabalho** — e o resultado define o modelo de dados, não o contrário.

Se o spike mostrar que os dados são piores do que se espera, você ainda tem produto: o motor de inferência vira o diferencial, exatamente como a visão previu. Mas você vai saber disso **antes**, e não depois de construir o resto.

---

## 3. Dois portões comerciais que começam agora

Não bloqueiam o spike, mas bloqueiam a produção. Resolva em paralelo, desde já:

**CNPJ.** Agregadores em geral contratam com pessoa jurídica, não física. Vale perguntar a cada fornecedor se MEI atende — é o caminho mais rápido para ter CNPJ. Descobrir isso na véspera do lançamento é atraso puro.

**Preço — e aqui há uma descoberta que muda o plano.** Levantamento de set/2026 indica **piso mensal fixo**, não custo por conexão: Tecnospeed ~R$ 540/mês (+ ~R$ 1.500 de entrada), Pluggy ~R$ 2.500/mês, Belvo ~R$ 6.000/mês. *(Números não confirmados na fonte — verifique.)*

Piso fixo significa que **você paga igual com 1 ou com 400 usuários**. Não existe soft launch barato. A R$ 2.500/mês, o break-even só do agregador é ~100 assinantes a R$ 24,90.

**Isso não invalida a conexão como core do produto — invalida ela como core do primeiro release.** Ver seção 12.

---

## 4. O manual não sai de cena — e não é concessão

Mesmo com o banco conectado, o cadastro manual **é requisito permanente**, não escopo de consolo:

- **Compromissos anteriores à janela de histórico.** Uma compra de 18 meses atrás em 24x pode não ter origem visível. Sem cadastro manual, ela some da sua linha do tempo — e a linha do tempo passa a mentir
- **Cobertura de instituição.** Nem todo banco responde bem, nem todo cartão aparece
- **Consentimento expira.** Até a renovação, o app precisa continuar funcionando
- **Dinheiro e boleto** não passam por Open Finance

E o custo é baixo: com o modelo de domínio já existente, cadastro manual é um formulário sobre as mesmas tabelas. Não é semana de trabalho — o que era caro é o **parser de OFX e CSV**, que continua fora.

> A regra: **conexão é o caminho feliz, manual é a rede de segurança.** Sempre visível, nunca obrigatório.

---

## 5. A jornada do MVP

```
cria conta
      ↓
conecta o cartão  ← o momento que define o produto
      ↓
sync + inferência de parcelas
      ↓
   ⭐ linha do tempo dos próximos 12 meses
      ↓
   "38% dos meus próximos 6 meses já está gasto"
      ↓
completa o que faltou, manualmente (opcional)
```

Duas telas carregam o produto: a **conexão** e a **linha do tempo**. Todo o resto é apoio.

---

## 6. Escopo — o que entra

**Conta e acesso**
- Cadastro e login
- Multi-tenant com RLS desde a primeira migration

**Conexão bancária**
- Widget do agregador, com redirect de consentimento (na web é redirect comum)
- Uma instituição para começar — a que você usa
- Sync de transações de cartão de crédito
- Status visível: "atualizado em", consentimento a expirar, falha de sync

**⭐ Motor de inferência de parcelas**
O coração técnico do MVP:
- Reconhecer `PARCELA 3/10`, `03/10`, `PARC 3 DE 10` e as variações que o spike revelar
- Agrupar parcelas dispersas num único compromisso
- Projetar as parcelas futuras que ainda não entraram em fatura
- Reconciliar cada parcela nova contra o compromisso já projetado, sem duplicar

**Compromissos**
- Gerados pela inferência, revisáveis pelo usuário
- Cadastro manual — parcelada e assinatura
- Editar, excluir, marcar parcela como paga

**⭐ Linha do tempo**
- Próximos 12 meses: renda prevista, total comprometido, saldo livre, percentual
- Abrir um mês e ver as parcelas que o compõem

**Configuração**
- Renda mensal prevista
- Cartões: fechamento e vencimento

---

## 7. Escopo — o que fica de fora

| Fora | Por quê |
|---|---|
| Importação OFX/CSV | O parser é caro e a conexão cobre o mesmo terreno melhor |
| IA e chat | Fase seguinte. Primeiro os dados precisam estar certos |
| Planos de quitação | Fase seguinte. Primeiro provar que **ver** muda decisão |
| Categorias e gráfico de gastos | É o app que você **não** está construindo |
| Conta corrente | O produto é sobre compromissos, e eles vivem no cartão |
| Detecção de assinatura esquecida | Depois, com histórico acumulado |
| Alertas e notificação | Fase 3 |
| Cobrança e assinatura | Fase 4 — mas o preço já precisa ser conhecido, ver seção 3 |
| Múltiplas instituições | Uma só no MVP. A segunda é configuração, não arquitetura |

---

## 8. Três armadilhas que decidem se funciona no dia 1

**Compromisso já em andamento.** No primeiro dia ninguém tem compras novas — tem compras no meio. "Estou na parcela 3 de 10." A inferência precisa disso, e o cadastro manual também. Se só aceitar compra nova, o app nasce inútil.

**Valor da parcela, não valor total.** A pessoa lembra "10x de R$ 89,90", não "R$ 899,00" — e com juros os dois nem batem. Peça a parcela e derive o total; o resto da divisão vai para a primeira parcela.

**Fechamento ≠ vencimento.** Compra depois do fechamento cai na fatura seguinte. Errar joga a parcela no mês errado, e a linha do tempo mente justamente onde precisa ser confiável.

---

## 9. Ordem de construção

1. **Spike de dados no sandbox** — sem código de produto. Só descobrir o que existe
2. **`packages/engine`** — `Centavos`, aritmética, geração e inferência de parcelas, **modelada sobre o que o spike encontrou**. Com testes, sem UI
3. **Schema + RLS + teste de isolamento no CI** — antes de existir dado para vazar
4. **Auth e casca do app**
5. **Conexão e sync** — widget, consentimento, ingestão, reconciliação
6. **Linha do tempo**
7. **Cadastro manual** como complemento
8. **Polimento do onboarding** — da criação de conta até a linha do tempo preenchida

O passo 1 antes do 2 é o ponto todo deste documento: **o modelo de domínio deve ser desenhado sobre dados reais, não sobre suposição.**

---

## 10. Critérios de sucesso

**Técnico** — vem primeiro, porque habilita o resto:
- A inferência acerta os compromissos parcelados do seu cartão real
- A linha do tempo bate com suas faturas de verdade
- Nova fatura reconcilia sem duplicar parcela

**Produto** — depois de quatro semanas de uso:
- Você usou nas quatro semanas
- **Você tomou pelo menos uma decisão diferente por causa da linha do tempo.** Este é *o* critério

**Negócio:**
- Custo real por usuário conectado é conhecido, e o preço fecha

---

## 11. O que isso muda no modelo de negócio

Nada na **forma** do paywall — mas inverte a **ordem de construção**.

```
GRÁTIS   cadastro manual + linha do tempo      custo marginal ~ zero
PAGO     conexão automática + IA               onde está o seu custo
```

Antes, a ideia era construir o gratuito primeiro e o pago depois. Agora você **constrói o pago primeiro**, e o gratuito é o que sobra quando se remove a conexão — porque o cadastro manual já existe como rede de segurança.

Era melhor assim — **até o preço do agregador entrar na conta.** Ver seção 12.

---

## 12. A correção que o preço do agregador impõe

O piso fixo de mensalidade (seção 3) cria um problema que nenhuma escolha de arquitetura resolve:

> Para ligar a conexão você precisa de ~100 assinantes. Para ter 100 assinantes você precisa de um produto. Para ter o produto você precisa da conexão.

**Como sair do laço, sem abrir mão da conexão como core:**

**1. Spike primeiro, e ele é grátis.** Pluggy oferece acesso completo à API por 14 dias, sem cartão, incluindo conexão real de Open Finance em produção. Isso responde as cinco perguntas da seção 2 a **custo zero** e sem contrato. É a jogada de maior retorno do projeto inteiro: em duas semanas você sabe se o produto é possível.

**2. Verifique a Tecnospeed antes de tudo.** A ~R$ 540/mês, o break-even cai para ~22 assinantes. Se atender, o laço praticamente desaparece.

**3. Lance o gratuito primeiro — agora por economia, não por risco.** O plano gratuito (cadastro manual + linha do tempo) tem custo marginal perto de zero. Ele constrói audiência e lista de espera enquanto você não pode pagar o piso. A conexão liga quando a lista converte.

Repare: é a mesma sequência que uma versão anterior deste documento propunha, mas o motivo é outro e mais forte. Não é "manual é um MVP mais barato de testar" — a conexão continua sendo o core do produto, como você apontou. É **"o piso do agregador não permite ligar a conexão antes de ter assinantes"**. A restrição é financeira, não de produto.

**O que o MVP continua sendo:** conectar e enxergar. O que muda é o momento de virar a chave da conexão para o público — e que, até lá, o gratuito segura a audiência.

---

## 12. Anti-escopo

Pedidos que vão aparecer e que devem ir para o backlog:

- "Já que estou aqui, deixa eu adicionar categoria"
- "Seria legal um gráfico de pizza dos gastos"
- "Preciso controlar minha conta corrente também"
- "E se tivesse orçamento por categoria?"
- "Dá para colocar meus investimentos?"

Todos são features razoáveis de um app de finanças. **Nenhum ajuda a provar que a conexão entrega parcelas confiáveis e que ver o futuro muda decisão** — e cada um adia a resposta.
