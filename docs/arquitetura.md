# Controlly — Arquitetura

> Consolida as decisões técnicas: **SaaS multiusuário**, **web por enquanto** (mobile adiado, não cancelado), **TypeScript full-stack com Next.js**.
> Contexto de produto em [visao-produto.md](visao-produto.md).

---

## 1. Decisões tomadas

| Pergunta | Resposta |
|---|---|
| Uso pessoal ou produto? | **Produto SaaS multiusuário.** Você é o usuário zero na Fase 0, mas multi-tenant no schema desde o commit 1 |
| Plataforma | **Web.** Aplicativo nativo adiado — decidir depois de ter tração |
| Framework | **Next.js** (App Router). Angular avaliado na seção 3 |
| Contrato de API | REST com schemas Zod gerando OpenAPI |
| Banco | PostgreSQL com Row Level Security |
| Stack | TypeScript full-stack — o motor roda no servidor **e** no cliente |
| Rust/Go no backend | **Não.** Carga é I/O-bound e o motor precisa rodar no cliente |
| Modelo de IA | API hospedada, executada **somente no servidor** |

Duas consequências que já valem:

- **"SaaS depois" não permite arquitetura de usuário único agora.** Isolamento de dados financeiros não se retrofita.
- **O plano gratuito não pode incluir conexão bancária** — é o custo que escala e que você não controla.

---

## 2. O que a decisão web-only libera

Descartar o empacotamento nativo **remove o maior risco técnico do projeto** e devolve liberdades que estavam fechadas:

| Antes (empacotado) | Agora (web) |
|---|---|
| Build estática obrigatória, SSR proibido | **SSR liberado** — Next.js volta à mesa |
| Token em Keychain/Keystore | **Cookie `httpOnly` + `SameSite`** — mais seguro e mais simples |
| CORS para origem separada | Mesma origem, CORS deixa de ser problema |
| Consentimento por deep link em device real | **Redirect comum de OAuth** |
| Guideline 4.2 da App Store | Irrelevante por ora |

### O consentimento do Open Finance ficou fácil

Era o ponto mais arriscado da arquitetura anterior: bancos bloqueiam WebView embarcada em tela de autenticação, então o fluxo exigia navegador do sistema e retorno por deep link, comprovável só em device real.

Na web isso é um **redirect padrão**. O usuário vai ao banco, autoriza, volta para o seu `redirect_uri`. Cuidados normais e conhecidos:

- `redirect_uri` registrada no agregador, com allowlist exata
- parâmetro `state` assinado, contra CSRF
- retorno tratado em rota dedicada, que troca o `code` e associa ao usuário logado

**Bônus:** os widgets de conexão dos agregadores (Pluggy Connect, Belvo Widget e similares) são **desenhados para web em navegador**. Você passou do caminho de exceção para o caminho feliz do fornecedor.

> Mobile continua no horizonte. O que precisa ser reavaliado quando voltar está no **Apêndice A** — não jogue fora, mas também não pague por ele agora.

---

## 3. Stack

### Next.js ou Angular?

Você levantou os dois. Ambos são escolhas sérias, e a diferença real não é qualidade — é **forma**.

**Angular é framework de frontend.** Traz DI, roteador, formulários tipados e HttpClient prontos, com pouca fadiga de decisão. Os **signals** dão reatividade granular, que serve o slider de simulação melhor que o modelo de re-render do React. Mas Angular não te dá backend: você continuaria rodando um Fastify ao lado, com dois deploys.

**Next.js é frontend + backend numa peça só.** Para um dev solo na Fase 0, isso é menos infraestrutura, menos deploy e menos contrato para manter. E mantém o ecossistema React, que é o mais profundo justamente para a **linha do tempo de caixa futuro** — visualização customizada com gesto, virtualização e acessibilidade, o componente que define o produto e que não vem pronto em biblioteca nenhuma. Se o mobile voltar como React Native, o conhecimento e parte da lógica atravessam.

**Recomendação: Next.js.** Menos peças para carregar sozinho, e o ecossistema certo para a tela que importa.

**A ressalva honesta:** se você já é forte em Angular e fraco em React, inverta. Produtividade em stack que você domina vence vantagem de ecossistema, sobretudo em Fase 0, onde tempo até a primeira versão útil é o recurso escasso. Nesse caso a estrutura é Angular + Fastify — que, aliás, é a separação mais limpa das duas.

### ⚠️ A armadilha do Next.js neste projeto

O Next torna fácil colocar lógica em Server Component e Server Action. **Aqui isso é perigoso**, e o motivo é específico:

> O motor precisa rodar **no cliente** para a simulação instantânea. Se a lógica escorrer para código server-only, você perde a funcionalidade que define a tela de planos.

O `engine` fica em `packages/`, sem I/O, importável dos dois lados. Server Action é transporte, não lógica — mesma regra da seção 9. Com Angular + Fastify essa fronteira é física; com Next.js ela é sua responsabilidade.

### Frontend

```
Next.js (App Router) + TypeScript
Tailwind                          mobile-first de verdade
TanStack Query                    cache e sincronização de servidor
Zod                               validação compartilhada com o backend
```

### Backend

```
Next.js route handlers            API da Fase 0 — transporte fino
Drizzle ORM                       schema, migrations, SQL quando precisar
PostgreSQL                        Supabase ou Neon
pg-boss                           fila de jobs, dentro do próprio Postgres
```

**Drizzle e não Prisma.** Você vai escrever política de RLS, `SET LOCAL` em transação e consulta de projeção com agregação. Drizzle é fino sobre SQL e não briga com nada disso. Prisma abstrai mais do que ajuda quando o banco é parte do modelo de segurança.

**pg-boss para fila.** O worker de sincronização precisa de agendamento e retentativa. pg-boss usa o Postgres que você já tem — zero infraestrutura nova.

**`apps/sync` continua serviço separado.** Sincronizar conexões bancárias é processo longo com agenda; não cabe em função serverless. É a única peça que não mora no Next.

### Contrato da API: REST com Zod, não tRPC

Com web-only, tRPC volta a ser viável — mas o ganho é pequeno e o custo não é zero.

Route handlers validando com os schemas Zod de `packages/domain` já te dão tipagem ponta a ponta, **e** um contrato explícito que gera OpenAPI quando precisar. Se o mobile voltar, isso é o que evita reescrever a camada de rede. tRPC acopla cliente e servidor em tempo de build e não tem história de versionamento — mantê-lo fora custa quase nada agora e economiza depois.

### Rust ou Go no backend?

**Não. E aqui a decisão é mais forte do que o normal, por causa da seção acima.**

O argumento clássico para Rust/Go é desempenho de CPU. **Ele não se aplica a este sistema**, que é I/O-bound de ponta a ponta:

| Operação | Gargalo real |
|---|---|
| Buscar transações no agregador | Rede e rate limit do fornecedor |
| Normalizar e inferir parcelas | Trivial (string e regex) |
| Calcular plano de quitação | **Microssegundos** |
| Servir a API | Postgres |
| Orquestrar a IA | Latência do modelo, em segundos |

Amortizar 60 meses para algumas dezenas de compromissos é irrelevante em qualquer linguagem. O tempo de resposta vai ser dominado pelo agregador e pelo modelo — e nenhum dos dois acelera porque o backend é Rust.

**O que decide de vez:** o motor precisa rodar **no cliente**, no navegador, para a simulação instantânea. Em Rust isso significaria compilar para WASM e manter uma fronteira de tipos com o resto do app; em Go, pior ainda. Em TypeScript é `import`. Você escolheria a linguagem mais difícil justamente para o requisito que ela atende pior.

**Onde Go se justificaria um dia:** o **worker de sincronização**. Ele puxa N usuários × M conexões em agenda, é puro fan-out de I/O, não compartilha tipo com a UI e tem contrato estreito — lê do agregador, normaliza, escreve no Postgres. É exatamente o componente que se reescreve isolado.

Então a providência é arquitetural, não linguística: **mantenha o sync como serviço separado**, falando por fila e por Postgres. Você compra a opção de portar sem pagar por ela agora.

**Gatilho para reabrir:** o sync entrar nos três maiores itens da conta de infraestrutura, ou o perfil de memória do Node virar limite comprovado com milhares de conexões. **Não é gatilho:** "Rust é mais rápido" sem perfil de execução na mão.

> Nota: TS não tem decimal nativo e Rust/Go têm. É uma vantagem real, mas pequena — inteiros em centavos (seção 5) fecham a lacuna por completo.

---

### Banco

**PostgreSQL** com **Row Level Security**. **Supabase** encurta o caminho para um dev solo: Postgres, Auth e RLS prontos.

O contexto do usuário é definido por você, no route handler:

```
Sessão (cookie httpOnly)
   ↓ handler valida e extrai o user_id
   ↓ abre transação
   ↓ SET LOCAL app.user_id = '<uuid>'
   ↓ queries rodam sob RLS
```

Escreva as políticas contra `current_setting('app.user_id', true)::uuid`, **não** contra `auth.uid()`. Duas vantagens: funciona com acesso direto ao Postgres, e trocar o provedor de auth depois vira trocar a validação de sessão — não reescrever política nenhuma.

---

## 4. Multi-tenant e isolamento

Regras que valem desde a primeira migration:

1. **Toda tabela de domínio tem `user_id NOT NULL`.** Sem exceção, mesmo enquanto o usuário é você.
2. **Row Level Security ligada em todas elas.** Isolamento no banco, não na aplicação. Um `WHERE user_id = ?` esquecido vira vazamento de extrato bancário entre clientes — o pior incidente possível para este produto. RLS transforma esse bug em zero linhas retornadas.
3. **Nenhuma query de domínio roda com role que faz bypass de RLS.** Service role só em migration e job administrativo.
4. **Teste de isolamento no CI**: usuário A não enxerga nada de B. Automatizado, rodando sempre, desde antes de existir um usuário B.

> ⚠️ **Armadilha de pool de conexões — e com Next.js ela é pior.** Defina o contexto com `SET LOCAL` **dentro da transação**. `SET` comum sobrevive à conexão devolvida ao pool e vaza contexto entre requisições. Runtime serverless reaproveita conexão de forma agressiva, então o erro falha de modo silencioso e intermitente — o pior tipo.

---

## 5. Dinheiro: regras invioláveis

- **Nunca float. Nunca.** `0.1 + 0.2 !== 0.3`, e em app financeiro isso vira centavo errado em fatura.
- Armazenar em **centavos como inteiro** (`bigint`). Formatar só na borda de exibição.
- Arredondamento explícito e documentado em toda divisão — parcelamento gera resto, e o resto tem que ir para algum lugar definido (convenção: sobra na primeira parcela, como a maioria das operadoras faz).
- Toda operação monetária passa pelo motor, com teste. Nenhuma aritmética de dinheiro solta em componente de UI.

---

## 5.1 Datas: o calendário do cartão é traiçoeiro

Tão corrosivo quanto float, e menos lembrado:

- **Fechamento ≠ vencimento.** Compra após o fechamento cai na fatura seguinte. São dois campos, não um.
- Vencimento é **dia de calendário**: coluna `date`, não `timestamptz`.
- "Hoje" é `America/Sao_Paulo` via IANA — nunca offset fixo no código.
- Vencimento em fim de semana ou feriado desloca conforme regra do emissor. Explicite a regra.

Errar isso joga uma parcela no mês errado, e a linha do tempo — a tela que define o produto — passa a mentir.

---

## 6. Camada de IA

```
Cliente  →  seu backend  →  API do modelo
             (chave aqui)
```

- **A chave de API nunca vai para o cliente.** Qualquer pessoa lê o bundle no navegador. Toda chamada passa pelo seu backend — use `import 'server-only'` no pacote `ai` para o build garantir isso.
- **Tool calling obrigatório.** O modelo chama `simular_plano`, `projetar_caixa`, `listar_compromissos`. Números só saem do motor determinístico.
- **Não enviar extrato bruto.** Agregados e recortes mínimos — privacidade, custo e qualidade de contexto ao mesmo tempo.
- **Rate limit por usuário no backend.** Sem isso, um usuário curioso com um loop vira uma fatura de API surpreendente.
- Categorização em cascata: regra → dicionário de merchants → modelo (só desconhecidos) → resultado vira regra.

---

## 7. A economia do SaaS

Aqui está o insight mais valioso desta seção, e ele resolve um problema de produto e um de custo ao mesmo tempo.

**Seus custos variáveis por usuário:**

| Custo | Comportamento |
|---|---|
| Agregador Open Finance | **Piso mensal fixo, não custo por conexão.** Ver abaixo — muda a forma do negócio |
| API do modelo | Por uso. Controlável com cache e cascata de categorização |
| Infra | Marginal no começo |

### O piso do agregador, e por que ele domina tudo

Levantamento de setembro de 2026 (fontes no fim da seção):

| Fornecedor | Custo de entrada | Mensalidade |
|---|---|---|
| Tecnospeed | ~R$ 1.500 | **~R$ 540** |
| Pluggy | — | ~R$ 2.500 |
| Belvo | — | ~R$ 6.000 |

> ⚠️ Números de relato público e de busca, não confirmados na fonte — as páginas dos fornecedores não foram acessíveis daqui. **Confirme antes de decidir qualquer coisa.**

**A consequência é estrutural e inverte uma premissa anterior deste documento.** O custo do agregador **não escala com o número de usuários** — é piso fixo. Você paga igual com 1 ou com 400 usuários.

Isso muda tudo:

- **Não existe soft launch com dez usuários.** O piso corre desde o primeiro dia
- **Break-even só do agregador**, a R$ 2.500/mês: ~100 assinantes a R$ 24,90, ou ~126 a R$ 19,90 — antes de infraestrutura, IA e impostos
- **Acima do piso a margem é excelente**, porque o custo marginal do usuário seguinte é quase zero. O problema é atravessar o vale, não escalar

**Consequência prática:** a conexão bancária não pode ser o que você liga no lançamento. Ela é o que você liga **quando já tem assinantes suficientes para cobrir o piso** — o que exige construir audiência antes, com o plano gratuito.

Tecnospeed a ~R$ 540/mês é 4,6× mais barato que Pluggy e derruba o break-even para ~22 assinantes a R$ 24,90. **Se atender ao caso de uso, muda o projeto inteiro** — é a primeira verificação a fazer.

**A conclusão que decide o produto:**

O plano gratuito **não pode incluir conexão bancária.** É o único custo que escala linearmente e que você não controla.

E isso cai perfeitamente onde precisa:

```
GRÁTIS   importação manual + OFX/CSV, compromissos,
         linha do tempo de caixa futuro
         → custo marginal ~ zero

PAGO     conexão bancária automática + assistente de IA
         → exatamente onde está o seu custo
```

Repare que **o paywall tem a mesma forma do roadmap.** Fases 0 e 1 são o plano gratuito. Fases 2 e 3 são o pago. Isso não é coincidência — é o produto avisando que a sequência está certa. E o gratuito é genuinamente útil sozinho, o que faz dele aquisição em vez de isca.

**Nota comercial:** agregadores em geral contratam com CNPJ, não com pessoa física. Isso é um portão prático para a Fase 2 — resolver antes, não durante.

---

## 8. LGPD, na condição de controlador

Enquanto for só você, quase nada disso morde. **No primeiro usuário externo, tudo passa a valer:**

- Política de privacidade publicada e honesta
- Base legal definida por finalidade; consentimento **granular** para a IA (usar o app sem IA precisa ser possível)
- Contrato de tratamento de dados com o agregador (ele é operador, você é controlador)
- Exportar e apagar tudo, de verdade, incluindo revogar o consentimento no agregador
- Plano de resposta a incidente, escrito antes de precisar dele
- Canal de contato do titular

Criptografia em repouso para transações e, obrigatoriamente, para tokens do agregador — em cofre de chaves. **Nunca logar payload** de transação ou de resposta do agregador.

---

### Ferramental e hospedagem

```
pnpm workspaces + Turborepo      monorepo
Vitest                           testes de engine e API — rápidos
Playwright                       E2E só dos fluxos críticos
```

**Hospedagem:** Next.js em Vercel, Netlify ou self-hosted — indiferente na Fase 0. **`apps/sync` precisa de processo longo** — Fly.io, Railway ou Render. Serverless não serve para worker com agenda e conexão de fila.

---

## 9. Estrutura do projeto

```
controlly/
├─ apps/
│  ├─ web/                    Next.js (App Router) — front + API da Fase 0
│  │  └─ src/
│  │     ├─ app/
│  │     │  ├─ (app)/         rotas autenticadas
│  │     │  │  ├─ timeline/   ⭐ a tela que define o produto
│  │     │  │  ├─ compromissos/
│  │     │  │  ├─ importacao/
│  │     │  │  ├─ planos/     simulação — engine roda no CLIENTE
│  │     │  │  └─ assistente/ chat com a IA
│  │     │  ├─ (auth)/
│  │     │  └─ api/           route handlers: transporte fino
│  │     ├─ components/ui/    primitivos compartilhados
│  │     └─ lib/              sessão, contexto de RLS, cliente de API
│  │
│  └─ sync/                   worker — processo longo, fora do Next
│     └─ src/jobs/            sincronizar-conexao, renovar-consentimento,
│                             reconciliar-parcelas
├─ packages/
│  ├─ engine/                 ⭐ TS puro, ZERO I/O — servidor E cliente
│  │  └─ src/
│  │     ├─ dinheiro/         Centavos, aritmética, arredondamento
│  │     ├─ compromissos/     inferência de parcelas, reconciliação
│  │     ├─ projecao/         linha do tempo de caixa futuro
│  │     └─ planos/           avalanche, bola de neve, antecipação
│  │
│  ├─ domain/                 tipos + schemas Zod — fonte única da verdade
│  ├─ db/                     Drizzle: schema, migrations, políticas RLS
│  ├─ importers/              OFX, CSV e adapters de Open Finance
│  └─ ai/                     ferramentas + orquestração — SOMENTE servidor
│
├─ docs/
├─ turbo.json
└─ package.json               pnpm workspaces
```

### Direção das dependências

```
        ┌─────────── domain ───────────┐
        │       (tipos e schemas)      │
        ↓                              ↓
   componentes ──→ engine ←── route handlers ──→ db, importers, ai
    (cliente)     (puro)         (servidor)      (só servidor)
                                                      ↑
                                                    sync
```

### Três regras que sustentam a arquitetura

**1. `engine` não conhece I/O.** Nenhum `fetch`, nenhum `pg`, nenhum `fs`. É o que permite rodá-lo no cliente para a simulação instantânea. Prenda com lint ou `dependency-cruiser` no CI — a violação é fácil, tentadora, e só dói meses depois.

**2. Código de cliente nunca importa `db` nem `ai`.** No Next.js essa fronteira é mais fácil de furar que num front separado, porque tudo mora no mesmo app. Use `import 'server-only'` nesses pacotes: o build quebra na hora em vez de vazar credencial para o bundle.

**3. Route handler e Server Action são transporte, nunca lógica.** No máximo ~20 linhas: valida, autoriza, chama `engine`, serializa.

`packages/engine` é o coração e o único lugar onde existe aritmética de dinheiro. Se ganhar dependência de rede ou de banco, a simulação vira round-trip e a tela de planos perde o que a torna útil.

---

## 9.1 Testes: onde eles realmente importam

Cobertura uniforme é desperdício. Concentre onde o erro é caro:

| Área | Rigor | Por quê |
|---|---|---|
| Motor financeiro (`engine`) | **Máximo** — casos de borda e propriedades | Número errado destrói a confiança |
| Inferência e reconciliação de parcelas | **Máximo** — corpus real de descrições sujas | É o diferencial do produto |
| Políticas RLS | **Alto** — teste que prova isolamento entre tenants | Falha aqui é vazamento de extrato |
| Dinheiro e datas | **Alto** | Erro silencioso e corrosivo |
| Importadores | Médio, com fixtures gravadas | Formato externo muda |
| UI | Só fluxos críticos | Retorno decrescente |

O `engine` não faz I/O — esses testes rodam em milissegundos e podem ser exaustivos. É o retorno concreto de mantê-lo puro.

---

## 10. Roadmap

**Fase 0 — Núcleo (grátis)**
Domínio + RLS, import OFX/CSV, entrada manual, inferência de parcelas, linha do tempo de caixa futuro.
*Saída:* você usa todo dia, sem banco conectado.

**Portão paralelo — Validar a economia unitária**
Preço real por conexão/mês dos agregadores e cobertura de instituições. Em SaaS o agregador é custo variável por usuário e define o preço mínimo da assinatura. **É portão, não tarefa.**

**Fase 1 — Inteligência (pago)**
Motor de planos com testes, chat com tool calling, categorização em cascata.

**Fase 2 — Open Finance (pago)**
Sandbox → um banco → demais. Reconciliação de parcelas. Sem o risco de deep link, esta fase ficou consideravelmente mais barata.

**Fase 3 — Automação**
Alerta de fatura projetada, gasto-fantasma, comprometimento excessivo. Por e-mail e web push.

**Fase 4 — Comercial**
Assinatura, CNPJ, LGPD completa.

**Depois — Mobile, se a tração justificar**
Ver Apêndice A.

---

## 11. Decisões que ficam para depois

- Caminho mobile — reabrir com tração, ver Apêndice A
- Provedor de Postgres — Supabase é o padrão adotado; a política de RLS foi escrita para não depender dele
- Qual agregador — decidir na Fase 2, **mas validar preço e cobertura agora**
- Preço da assinatura — depende do custo real do agregador

---

## Apêndice A — Se o aplicativo voltar ao roadmap

Preservado da análise anterior, quando o empacotamento com Capacitor estava em consideração. Nada disso custa nada agora; tudo volta a valer no dia em que mobile entrar.

### O que reavaliar

- **React Native** vs **empacotar a web**. RN dá experiência nativa de verdade e compartilha `engine`, `domain` e o cliente de API — o motivo pelo qual manter React na web hoje tem valor de opção.
- **Push nativo.** A Fase 3 é feita de alerta. Web push funciona, mas no iOS é frágil. Se alerta virar o núcleo do produto, isso sozinho justifica o app.
- **Biometria** para travar um app financeiro. Na web, WebAuthn resolve parcialmente.

### As armadilhas que já mapeamos, para não redescobrir

- **Empacotar a web exige build estática.** SSR sai. Se você tiver ido fundo em Server Components, migrar custa — é o preço embutido na escolha do Next.js hoje, e é consciente.
- **Bancos bloqueiam WebView embarcada** em tela de autenticação. Num app, o consentimento precisa abrir no navegador do sistema e voltar por deep link (Universal Link / App Link). **Prototipar em device real antes de qualquer outra coisa** — é o maior risco técnico do caminho mobile.
- **Guideline 4.2 da App Store:** app que é só site empacotado é rejeitado. Biometria, push e armazenamento seguro nativos resolvem.
- **Armazenamento de token:** Keychain/Keystore. `@capacitor/preferences` **não é criptografado** — nunca para token bancário.
- **App financeiro que conecta conta bancária recebe escrutínio extra** nas duas lojas. Política de privacidade publicada e descrição honesta do uso de dados prontas antes de submeter.
- Verificar se o widget do agregador escolhido suporta o ambiente do app — vários são pensados para navegador.
