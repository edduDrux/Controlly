# Controlly — Arquitetura

> Consolida as decisões técnicas tomadas: **SaaS multiusuário**, **web mobile-first com aplicativo nativo depois**, **TypeScript full-stack**.
> Contexto de produto em [visao-produto.md](visao-produto.md).

---

## 1. As três forças que moldam tudo

1. **SaaS** → dados de terceiros no mesmo banco. Isolamento e criptografia deixam de ser boa prática e viram requisito.
2. **Aplicativo nativo depois** → a lógica **não pode morar no framework web**. Se morar, o app nativo vira reescrita, não porte.
3. **TypeScript nos dois lados** → o ativo a proteger é o **tipo compartilhado** entre domínio, API, web e futuro app. Toda decisão que quebra isso precisa se justificar muito bem.

---

## 2. Back-end em Rust ou Go?

**Recomendação: não agora. Fique em TypeScript — mas isole o worker de sincronização para manter a porta aberta.**

### Por que não

**A carga deste sistema é I/O-bound, não CPU-bound.** Vale olhar o que o back-end de fato faz:

| Operação | Natureza | Gargalo real |
|---|---|---|
| Buscar transações no agregador | HTTP externo | Rede e rate limit do fornecedor |
| Normalizar, deduplicar, inferir parcelas | String e regex | Trivial |
| Calcular plano de quitação | Aritmética | **Microssegundos** |
| Servir a API | Query + serialização | Postgres |
| Orquestrar a IA | HTTP externo | Latência do modelo (segundos) |

O argumento clássico para Rust/Go é desempenho de CPU. **Aqui ele não se aplica.** Amortizar 60 meses para algumas dezenas de compromissos é irrelevante em qualquer linguagem — inclusive JavaScript. O tempo de resposta vai ser dominado pelo agregador e pelo modelo de IA, e nenhum dos dois fica mais rápido porque seu back-end é Rust.

**O que você perderia é concreto:**

- **Tipos compartilhados.** Hoje uma `Parcela` é definida uma vez e vale no motor, na API, na web e no futuro React Native. Com Rust/Go você passa a gerar tipos através de uma fronteira de linguagem — funciona, mas é atrito permanente, e é justamente a vantagem que te fez escolher TS.
- **Duas linguagens para manter sozinho**, antes de ter receita.
- **Tempo até a Fase 0**, que agora é o recurso mais escasso.

**Rust especificamente é o pior encaixe para esta fase.** Modelagem de domínio em Rust é genuinamente excelente — tipos novos para dinheiro, sem `null`, `match` exaustivo, `rust_decimal`. Mas o custo de desenvolvimento é alto, o ecossistema de SDK de IA é mais fino, e nada disso ajuda a descobrir se o produto tem mercado.

### O único ponto onde Go se justificaria

O **worker de sincronização**. Ele puxa N usuários × M conexões bancárias em agenda, é puro fan-out de I/O, não compartilha tipo nenhum com a interface, e tem contrato estreito: lê do agregador, normaliza, escreve no Postgres.

Se um dia ele virar gargalo de custo ou de confiabilidade, é **exatamente o tipo de componente que se reescreve isolado** — sem tocar em mais nada. Go se sairia bem ali: concorrência barata, binário estático, pouca memória.

**Então a decisão prática é arquitetural, não linguística:** mantenha o sync como serviço separado, comunicando por fila e por Postgres. Você compra a opção de portar depois sem pagar por ela agora.

### Gatilho para reconsiderar

Reabra a discussão quando **medir**, não quando intuir:

- O sync entra nos três maiores itens da conta de infraestrutura
- Você sincroniza milhares de conexões e o perfil de memória do Node é comprovadamente o limite
- Entra alguém no time com força real em Go

**Não é gatilho:** "Rust é mais rápido". Sem perfil de execução na mão, isso é preferência, não engenharia.

> Nota lateral: TS não tem decimal nativo, e Rust/Go têm (`rust_decimal`, `shopspring/decimal`). É uma vantagem real, mas pequena — **inteiros em centavos** fecham a lacuna por completo (seção 5). Não é motivo para trocar de linguagem.

---

## 3. Organização do código

```
controlly/
├─ packages/
│  ├─ core/        domínio + motor de planos + inferência de parcelas
│  │               ZERO I/O, zero framework. 100% testável em memória
│  ├─ contracts/   schemas Zod = fonte única de verdade dos tipos
│  ├─ db/          schema Drizzle, migrations, políticas RLS
│  ├─ sources/     adapters: manual | ofx | csv | openfinance
│  └─ ai/          definição de ferramentas + orquestração do LLM
└─ apps/
   ├─ web/         Next.js App Router, mobile-first
   ├─ sync/        worker de sincronização (o candidato a Go)
   └─ mobile/      React Native — Fase 4
```

`packages/core` é a peça inegociável. Sem I/O, sem `fetch`, sem acesso a banco, sem dependência de Next. Só funções puras sobre o domínio. É o que torna o motor testável de verdade e o que sobrevive a qualquer troca de framework.

### API-first sem over-engineering

Você vai querer um app nativo depois, mas **não crie um serviço de API separado agora** — é peso sem retorno na Fase 0.

A regra que entrega o mesmo resultado de graça:

> **Route handler é transporte, nunca lógica.** No máximo ~20 linhas: valida a entrada, autoriza, chama `core`, serializa a saída.

Se essa regra for respeitada, extrair `apps/api` na Fase 4 é mecânico, porque não existe lógica presa ao Next para desgrudar. Se for violada, o app nativo custa uma reescrita. É a regra mais barata de seguir e a mais cara de ignorar.

**Contrato:** tRPC resolve bem, já que web e React Native são ambos TS. Mantenha os schemas Zod em `contracts` para conseguir gerar OpenAPI depois, se surgir cliente externo.

---

## 4. Multi-tenancy

Com dados financeiros de terceiros, filtrar por `user_id` na aplicação **não é suficiente** — um `where` esquecido vaza dados de outra pessoa.

**Defesa no banco: Row Level Security do Postgres.**

- Toda tabela de dados do usuário carrega `user_id`
- Política RLS: a linha só é visível se `user_id = current_setting('app.user_id')::uuid`
- A aplicação define esse valor por transação
- Um `where` esquecido passa a retornar **vazio**, não os dados de outra pessoa

⚠️ Com pool de conexões (PgBouncer, serverless), use `SET LOCAL` **dentro da transação** — `SET` normal vaza contexto entre requisições, o que seria pior que não ter RLS.

---

## 5. Dinheiro e datas

### Dinheiro: inteiros em centavos. Sem exceção.

- **Nunca** `float`, `double`, `real` ou o tipo `money` do Postgres
- Armazene `bigint` em centavos; formate só na borda de exibição
- `0.1 + 0.2 !== 0.3` — em app financeiro isso vira erro de centavo em relatório e destrói confiança
- Um tipo nomeado (`type Centavos = number`) documenta a unidade e evita somar reais com centavos por engano

### Datas: o calendário do cartão é traiçoeiro

- **Fechamento ≠ vencimento.** Uma compra depois do fechamento cai na fatura seguinte. Modele os dois.
- Vencimento é `date`, não `timestamptz` — é dia de calendário, não instante
- "Hoje" é `America/Sao_Paulo`, via IANA, nunca offset fixo no código
- Vencimento em fim de semana ou feriado desloca — regra do emissor, precisa ser explícita

Esses detalhes parecem menores e são exatamente onde app de finanças perde credibilidade.

---

## 6. Segurança

O token do agregador dá acesso de leitura à vida financeira inteira de alguém. Trate como o ativo mais sensível do sistema.

- **Criptografia envelopada** com cofre de chaves (KMS/Vault). Nunca chave em variável de ambiente em texto puro em produção
- **Nunca logar payload** de transação ou resposta do agregador. Log estruturado com redação por padrão
- Trilha de auditoria de todo acesso a dado financeiro
- Escopo somente leitura, sem iniciação de pagamento — limita o dano máximo de um vazamento
- Rotação de chave prevista desde o início; retrofit é caro

---

## 7. Testes: onde eles realmente importam

Cobertura uniforme é desperdício. Concentre onde o erro é caro:

| Área | Rigor | Por quê |
|---|---|---|
| Motor de planos (`core`) | **Máximo** — casos de borda, propriedades | Número errado destrói a confiança do usuário |
| Inferência e reconciliação de parcelas | **Máximo** — corpus real de descrições sujas | É o diferencial do produto |
| Políticas RLS | **Alto** — teste que prova isolamento entre tenants | Falha aqui é vazamento |
| Dinheiro e datas | **Alto** | Erro silencioso e corrosivo |
| Adapters de fonte | Médio, com fixtures gravadas | Formato externo muda |
| UI | Fluxos críticos | Retorno decrescente |

O motor não faz I/O — então esses testes rodam em milissegundos e podem ser exaustivos. Esse é o retorno concreto de manter `core` puro.

---

## Resumo das decisões

| Decisão | Escolha | Motivo em uma linha |
|---|---|---|
| Linguagem | TypeScript full-stack | Tipos compartilhados entre domínio, API, web e app |
| Rust/Go no back-end | Não agora | Carga é I/O-bound; reconsiderar só para o worker de sync, com medição |
| Domínio | `packages/core` puro, sem I/O | Testável, e sobrevive a troca de framework |
| API | Route handler fino, sem serviço separado ainda | Torna o app nativo um porte, não uma reescrita |
| Isolamento | Postgres RLS | `where` esquecido retorna vazio, não dados alheios |
| Dinheiro | Inteiros em centavos | Ponto flutuante erra centavo |
| Sync | Serviço isolado por fila | Preserva a opção de portar para Go |
