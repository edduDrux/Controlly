# Controlly — MVP

> O que entra, o que fica de fora, e por quê.
> Produto em [visao-produto.md](visao-produto.md) · Técnico em [arquitetura.md](arquitetura.md)

---

## 1. O que o MVP precisa provar

Uma coisa só:

> **Que ver o comprometimento futuro muda decisão.**

Se você olhar a linha do tempo e mudar de ideia sobre parcelar algo — o produto tem razão de existir. Se olhar, achar interessante e não mudar nada, a tese está errada, e é muito melhor descobrir isso com quatro semanas de trabalho do que com oito meses.

Tudo que não serve a essa prova fica de fora. Sem exceção.

**O que o MVP não é:** lançamento. Ele é a Fase 0 — o plano gratuito e o seu próprio dogfooding. Produto vendável só existe depois da conexão bancária, na Fase 2.

---

## 2. A decisão desconfortável: o MVP é de digitação manual

Isso parece contradizer o pedido original — *"não quero digitar todos os meus gastos manualmente"*. Não contradiz, e a distinção é a coisa mais importante deste documento:

> **Você não vai digitar transações. Você vai digitar compromissos. São coisas de ordem de grandeza diferente.**

| | Quantidade | Esforço |
|---|---|---|
| Transações de um mês | ~80 a 150 | Insuportável. É o que mata app de finanças |
| **Compromissos ativos** | **~10 a 25** | **15 minutos, uma vez** |

Uma compra em 10x é **um cadastro** que gera **dez parcelas**. Vinte compromissos geram algo em torno de duzentas parcelas futuras — a linha do tempo inteira, a partir de quinze minutos de digitação.

O que você odeia é registrar cafezinho. O MVP não precisa de cafezinho: **precisa do que já está comprometido.** E isso é pouco, é estável, e você lembra de cabeça.

**Por que isso é a escolha certa e não corte de canto:**

- Importar OFX e CSV exige parser, deduplicação, normalização de estabelecimento e reconciliação — semanas de trabalho que **reduzem atrito**, não que **testam a tese**.
- Se a tese estiver errada, esse trabalho todo vira lixo.
- Se estiver certa, você constrói a importação sabendo exatamente qual formato de dado importa, porque já viu o modelo funcionando com dados reais.

Importação entra na v1.1, logo depois. Conexão bancária na Fase 2.

---

## 3. A única jornada do MVP

```
cadastra renda mensal
      ↓
cadastra os cartões (fechamento e vencimento)
      ↓
cadastra os compromissos que já tem
      ↓
  ⭐ vê a linha do tempo dos próximos 12 meses
      ↓
   "38% dos meus próximos 6 meses já está gasto"
```

A última linha é o **momento de virada**. Todo o resto do MVP existe para o usuário chegar nela rápido.

Consequência de design: **o formulário de compromisso é a peça mais importante depois da linha do tempo.** Se cadastrar for chato, ninguém chega ao momento de virada e o teste da tese falha por motivo errado — atrito de interface, não tese furada. Vale investir desproporcionalmente em fazer esse formulário rápido.

---

## 4. Escopo — o que entra

**Conta e acesso**
- Cadastro e login
- Multi-tenant com RLS desde a primeira migration, mesmo com um usuário só

**Configuração**
- Renda mensal prevista (um valor recorrente basta)
- Cartões: apelido, **dia de fechamento** e **dia de vencimento** — são campos distintos

**Compromissos**
- Compra parcelada: descrição, valor da parcela, nº de parcelas, data da primeira, cartão
- Assinatura recorrente: valor, dia, sem fim definido
- Geração automática das parcelas
- Editar e excluir
- Marcar parcela como paga

**⭐ Linha do tempo**
- Próximos 12 meses, mês a mês
- Por mês: renda prevista, total comprometido, **saldo livre**
- Percentual comprometido — o número que gera a virada
- Abrir um mês e ver as parcelas que o compõem

Só isso. É pequeno de propósito.

---

## 5. Escopo — o que fica de fora, e por quê

| Fora | Por quê |
|---|---|
| Importação OFX/CSV | Reduz atrito, não testa a tese. v1.1 |
| Conexão bancária | Fase 2. É o produto pago, não o MVP |
| IA e chat | Fase 1. Sem dados reais no modelo, não há o que explicar |
| Planos de quitação | Fase 1. Primeiro provar que **ver** muda decisão; depois otimizar |
| Categorias e gráfico de gastos | É o app que você **não** está construindo |
| Transações avulsas | O MVP é de compromissos. Gasto do dia a dia não é o produto |
| Detecção de assinatura esquecida | Precisa de histórico de transação |
| Alertas e notificação | Fase 3 |
| Cobrança e assinatura | Fase 4 |
| Empréstimo e financiamento | Cabe no modelo, mas tem cronograma com juros. v1.1 |

---

## 6. O detalhe que quase todo mundo erra

**Compromisso que já está em andamento.**

No dia em que você começa a usar, você não tem compras novas — tem compras **no meio**. "Estou na parcela 3 de 10."

Se o formulário só aceitar compra nova, o app fica inútil no primeiro dia e o usuário nunca chega ao momento de virada. Então:

- O cadastro **precisa** aceitar "já paguei N parcelas"
- As parcelas já pagas entram como `paga`, não somem — o histórico importa para o total do compromisso
- A linha do tempo começa da próxima parcela em aberto

Isso não é refinamento. É requisito de MVP, e é o tipo de coisa que se descobre tarde demais.

**Dois outros que doem:**

- **Valor da parcela, não valor total.** A pessoa lembra "10x de R$ 89,90", não "R$ 899,00". Peça a parcela e derive o total. Com juros os dois nem batem, e o resto da divisão vai para a primeira parcela, pela convenção da seção de dinheiro da arquitetura.
- **Fechamento ≠ vencimento.** Compra depois do fechamento cai na fatura seguinte. Errar isso joga a parcela no mês errado e a linha do tempo passa a mentir — justamente a tela que precisa ser confiável.

---

## 7. Modelo de dados mínimo

```
usuario
renda            valor_centavos, dia_recebimento
cartao           apelido, dia_fechamento, dia_vencimento
compromisso      tipo (parcelada | assinatura), descricao,
                 cartao_id?, valor_parcela_centavos, total_parcelas?,
                 data_primeira
parcela          compromisso_id, numero, vencimento (date),
                 valor_centavos, status (prevista | paga)
```

Sete tabelas contando `usuario`. Todas com `user_id NOT NULL` e RLS ligada.

Repare no que **não** existe ainda: `transacao`, `categoria`, `regra`, `fatura`, `plano`. Entram quando a importação entrar — e o modelo da arquitetura já prevê o lugar delas.

---

## 8. Ordem de construção

A ordem importa tanto quanto o escopo:

1. **Monorepo + `packages/engine`** — tipo `Centavos`, aritmética, arredondamento, geração de parcelas a partir de um compromisso. **Com testes, sem nenhuma UI.**
2. **Schema + RLS + teste de isolamento no CI** — antes de existir dado para vazar.
3. **Auth e casca do app.**
4. **CRUD de cartão e compromisso**, com o formulário rápido da seção 3.
5. **Linha do tempo.**
6. **Polimento do fluxo de entrada** — porque é ele que decide se alguém chega ao momento de virada.

**Por que nessa ordem:** os passos 1 e 2 são onde mora o risco real — lógica de dinheiro e isolamento de dados. Resolvidos primeiro, sem UI para atrapalhar, e testáveis em milissegundos. Interface é a parte fácil de mudar; essas duas não são.

---

## 9. Critérios de sucesso

O MVP deu certo se, depois de quatro semanas:

- **Você usou nas quatro semanas.** Abandono é a resposta mais honesta que existe
- **A linha do tempo bate com a realidade** — confira contra suas faturas de verdade
- **Você tomou pelo menos uma decisão diferente por causa dela.** Este é *o* critério
- **O modelo de domínio aguentou seus casos reais sem gambiarra** — validação técnica, e é a que autoriza construir a importação em cima

Se o terceiro não acontecer, pare e repense o produto antes de construir a Fase 1. É exatamente para isso que o MVP é pequeno.

---

## 10. Anti-escopo

Pedidos que vão aparecer durante a construção e que devem ir para o backlog, não para a sprint:

- "Já que estou aqui, deixa eu adicionar categoria"
- "Seria legal um gráfico de pizza dos gastos"
- "Preciso controlar minha conta corrente também"
- "E se tivesse orçamento por categoria?"
- "Dá para colocar meus investimentos?"

Todos são features razoáveis de um app de finanças. **Nenhum ajuda a provar que ver o comprometimento futuro muda decisão** — e cada um adia a resposta.
