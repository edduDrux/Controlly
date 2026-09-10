# Controlly — Arquitetura

> Decorre de [visao-produto.md](visao-produto.md). Resolve as decisões em aberto da seção 10 daquele documento.

---

## 1. Decisões tomadas

| Decisão | Escolha | Consequência principal |
|---|---|---|
| **Escopo** | Uso pessoal primeiro, **SaaS comercial como destino declarado** | Multi-tenant no schema desde o commit 1. Não existe "depois eu adapto" para isolamento de dados financeiros |
| **Plataforma** | **Web mobile-first**, empacotado com **Capacitor** | O front tem que ser SPA estática. Isso elimina SSR e muda autenticação (seção 2) |
| **Open Finance** | Somente leitura, via agregador | Sem ITP, sem autorização própria no BACEN |
| **IA** | API hospedada, executada **só no servidor** | Chave de API nunca vai para o bundle |

O ponto que muda tudo: **"pessoal primeiro, SaaS depois" não é desculpa para arquitetura de usuário único.** Retrofitar isolamento de dados em um app financeiro é como retrofitar freio — tecnicamente possível, e ninguém deveria querer descobrir se funciona. Custa quase nada colocar `user_id` em toda tabela hoje e custa uma reescrita amanhã.

---

## 2. O que o Capacitor impõe

Capacitor não é "o site num app". Ele muda premissas de arquitetura. Estas são as que doem se descobertas tarde:

### 2.1 O app roda de um bundle local

A origem em produção é `capacitor://localhost` (iOS) ou `http://localhost` (Android) — **não** o seu domínio.

**Consequências obrigatórias:**

- **A build tem que ser estática.** SPA client-side. **SSR está fora** — Next.js com server components, Remix SSR e afins não empacotam. Se quiser Next.js, só em `output: 'export'`, e aí você perdeu o motivo de usar Next.js.
- **Separação front/back é obrigatória**, não estilística. Todo dado vem por API.
- **CORS** precisa liberar explicitamente `capacitor://localhost` e `http://localhost`. É a primeira coisa que quebra no primeiro build nativo.

### 2.2 Autenticação por cookie não funciona bem

Origem diferente do backend + restrições de cookie de terceiros = sessão por cookie `httpOnly` é fonte de sofrimento no Capacitor.

**Decisão:** autenticação **por token** (access token curto + refresh token), com o token guardado no **Keychain (iOS) / Keystore (Android)** através de um plugin de armazenamento seguro.

> ⚠️ `@capacitor/preferences` **não é criptografado** — é `UserDefaults`/`SharedPreferences`. Serve para preferência de tema. **Nunca** para token de acesso a dado bancário.

Na web pura (mesmo código rodando no navegador), o token cai em memória com refresh silencioso, não em `localStorage`.

### 2.3 O redirect do Open Finance é o ponto crítico

A jornada de consentimento manda o usuário para o app ou site do banco e traz de volta. No Capacitor isso tem uma armadilha séria:

> **Bancos bloqueiam WebView embarcada em telas de autenticação.** É defesa contra captura de credencial, e está correto. Se você abrir o consentimento na WebView do próprio app, **quebra** — e você vai perder tempo achando que é bug seu.

**Caminho correto:**

```
App  →  @capacitor/browser  →  navegador do sistema
                                (SFSafariViewController / Chrome Custom Tabs)
                                       ↓
                                consentimento no banco
                                       ↓
     ←  deep link (Universal Link / App Link)  ←  callback
                                       ↓
         @capacitor/app  →  listener appUrlOpen  →  troca o code
```

**Isto precisa ser prototipado na primeira semana da Fase 2, antes de qualquer outra coisa dessa fase.** É o maior risco técnico do projeto inteiro, e é o tipo de coisa que só aparece em device real.

Verificar também se o widget de conexão do agregador escolhido (Pluggy Connect, Belvo Widget, equivalentes) é suportado em Capacitor — vários são pensados para web em navegador. **Pergunte ao fornecedor antes de assinar contrato.**

### 2.4 O que o Capacitor te dá de volta

Justifica a escolha e não é pouco:

- **Biometria** (Face ID / digital) para travar o app. Para app financeiro isso é obrigatório, não enfeite.
- **Push nativo** (APNs/FCM). A Fase 3 é feita de alerta — "sua fatura projetada estourou o previsto". Push na web no iOS é frágil; nativo não é.
- **Armazenamento seguro** de token.
- Detecção de screenshot, bloqueio ao ir para segundo plano — higiene de app financeiro.

### 2.5 Loja de aplicativos

Dois pontos a planejar, não a descobrir:

- **Apple, Guideline 4.2 (Minimum Functionality).** App que é só um site empacotado é rejeitado. Usar biometria, push e armazenamento seguro nativos resolve — mais uma razão para fazê-los cedo.
- **App financeiro que conecta conta bancária recebe escrutínio extra** nas duas lojas. Política de privacidade publicada, descrição honesta do uso de dados e demonstração de legitimidade precisam estar prontas antes da submissão.

---

## 3. Stack

### Frontend

```
Vite + React + TypeScript        SPA, empacotável pelo Capacitor
TanStack Query                   cache e sincronização de servidor
Tailwind                         mobile-first de verdade
Capacitor                        shell nativo
```

**Ionic UI ou não?** Você citou "Capacitor by Ionic" — vale saber que os dois são separáveis. Capacitor funciona sozinho, sem o framework de UI do Ionic.

- **Ionic React**: componentes que parecem nativos, navegação e gestos prontos. Bom se quer cara de app.
- **Tailwind puro**: controle total. A linha do tempo de caixa futuro é uma visualização customizada — o componente mais importante do produto não vem pronto em biblioteca nenhuma.

**Inclinação:** Capacitor + Tailwind, sem o framework de UI. A tela que define o produto é feita à mão de qualquer jeito, e o Ionic cobra peso de bundle e opinião de layout por componentes que você vai usar pouco. Decisão reversível — dá para adotar Ionic depois.

### Backend

```
Node + TypeScript + Fastify      API HTTP
Drizzle ORM                      schema, migrations, SQL quando precisar
PostgreSQL                       Supabase ou Neon
pg-boss                          fila de jobs, dentro do próprio Postgres
Zod                              validação, compartilhada com o frontend
```

**Fastify e não NestJS.** A estrutura deste projeto vem dos pacotes, não do framework: a lógica mora em `engine`, e a rota é transporte fino. NestJS cobraria cerimônia de módulo e injeção de dependência para organizar algo que já está organizado um nível acima. Fastify entrega roteamento, validação por schema e plugins, e sai da frente.

**Drizzle e não Prisma.** Você vai escrever política de RLS, `SET LOCAL` em transação e consulta de projeção com agregação. Drizzle é fino sobre SQL e não briga com nada disso. Prisma abstrai mais do que ajuda quando o banco é parte do modelo de segurança.

**pg-boss para fila.** O worker de sincronização precisa de agendamento e retentativa. pg-boss usa o Postgres que você já tem — zero infraestrutura nova. Redis e BullMQ só quando o volume justificar; até lá seria custo e uma peça a mais para operar.

**Por que TypeScript no backend, e não Python:** existe uma vantagem específica deste projeto que decide o empate.

> Se o **motor financeiro for um pacote TypeScript puro, sem I/O**, ele roda **no servidor e no cliente com o mesmo código**.

Isso significa simulação "e se eu aportar mais R$ 300?" com slider respondendo **instantaneamente, sem round-trip**, e o mesmo código validando no servidor. Para uma tela de simulação de quitação, isso é a diferença entre brinquedo e ferramenta. Python não te dá isso.

### Contrato da API: REST com OpenAPI, não tRPC

tRPC é tentador — os dois lados são TypeScript. **Mas o Capacitor decide contra.**

> Binário de loja **não pode ser forçado a atualizar.** Um usuário com a versão de três meses atrás vai bater no seu servidor de hoje.

tRPC acopla cliente e servidor em tempo de build e não oferece nenhuma história de versionamento. Com app publicado, isso vira quebra silenciosa em device de terceiro — o pior lugar para descobrir.

**Decisão:** REST, com os schemas Zod de `packages/domain` gerando **OpenAPI**. Você mantém a tipagem ponta a ponta (cliente gerado a partir do mesmo schema) e ganha um contrato explícito, versionável e inspecionável. Se o produto fosse web para sempre, tRPC seria a escolha certa. Não é o caso.

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

**O que decide de vez:** o motor precisa rodar **no cliente**, dentro do bundle do Capacitor, para a simulação instantânea. Em Rust isso significaria compilar para WASM e manter uma fronteira de tipos com o resto do app; em Go, pior ainda. Em TypeScript é `import`. Você escolheria a linguagem mais difícil justamente para o requisito que ela atende pior.

**Onde Go se justificaria um dia:** o **worker de sincronização**. Ele puxa N usuários × M conexões em agenda, é puro fan-out de I/O, não compartilha tipo com a UI e tem contrato estreito — lê do agregador, normaliza, escreve no Postgres. É exatamente o componente que se reescreve isolado.

Então a providência é arquitetural, não linguística: **mantenha o sync como serviço separado**, falando por fila e por Postgres. Você compra a opção de portar sem pagar por ela agora.

**Gatilho para reabrir:** o sync entrar nos três maiores itens da conta de infraestrutura, ou o perfil de memória do Node virar limite comprovado com milhares de conexões. **Não é gatilho:** "Rust é mais rápido" sem perfil de execução na mão.

> Nota: TS não tem decimal nativo e Rust/Go têm. É uma vantagem real, mas pequena — inteiros em centavos (seção 5) fecham a lacuna por completo.

---

### Banco

**PostgreSQL** com **Row Level Security**. **Supabase** encurta o caminho para um dev solo: Postgres, Auth e RLS prontos.

**Um detalhe que muda a implementação:** como o Capacitor exige API separada, você **não** usa o PostgREST do Supabase. O Fastify conecta direto no Postgres. Então o contexto do usuário é definido por você:

```
JWT do Supabase Auth
   ↓ Fastify valida e extrai o user_id
   ↓ abre transação
   ↓ SET LOCAL app.user_id = '<uuid>'
   ↓ queries rodam sob RLS
```

Escreva as políticas contra `current_setting('app.user_id', true)::uuid`, **não** contra `auth.uid()`. Duas vantagens: funciona com API própria, e trocar o Supabase Auth depois passa a ser trocar a verificação do JWT — não reescrever política nenhuma.

---

## 4. Multi-tenant e isolamento

Regras que valem desde a primeira migration:

1. **Toda tabela de domínio tem `user_id NOT NULL`.** Sem exceção, mesmo enquanto o usuário é você.
2. **Row Level Security ligada em todas elas.** Isolamento no banco, não na aplicação. Um `WHERE user_id = ?` esquecido em um endpoint vira vazamento de extrato bancário entre clientes — o pior incidente possível para este produto. RLS transforma esse bug em zero linhas retornadas.
3. **Nenhuma query de domínio roda com role que faz bypass de RLS.** Service role só em migration e job administrativo.
4. **Teste de isolamento no CI**: usuário A não enxerga nada de B. Teste automatizado, rodando sempre, desde antes de existir um usuário B.

> ⚠️ **Armadilha de pool de conexões.** Com PgBouncer ou driver serverless, defina o contexto do usuário com `SET LOCAL` **dentro da transação**. `SET` comum sobrevive à conexão devolvida ao pool e vaza contexto entre requisições — isso é pior do que não ter RLS, porque falha de forma silenciosa e intermitente.

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

- **A chave de API nunca vai para o bundle.** Bundle de Capacitor é um zip: qualquer pessoa extrai. Toda chamada passa pelo seu backend.
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
| Agregador Open Finance | Por conexão/mês. **Dominante.** Um usuário com 3 contas custa 3× |
| API do modelo | Por uso. Controlável com cache e cascata de categorização |
| Infra | Marginal no começo |

> ⚠️ Preços de agregador mudam e variam por volume. Confirme na fonte antes de modelar qualquer coisa a sério.

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

**Hospedagem:** web é build estática (Cloudflare Pages, Vercel, Netlify — indiferente). **API e sync precisam de processo longo** — Fly.io, Railway ou Render. Serverless não serve para o worker de sincronização, que roda em agenda e mantém conexão de fila.

---

## 9. Estrutura do projeto

```
controlly/
├─ apps/
│  ├─ web/                    Vite + React + Capacitor
│  │  ├─ ios/  android/       gerados pelo Capacitor, versionados
│  │  ├─ src/
│  │  │  ├─ features/         organizado por domínio, não por tipo de arquivo
│  │  │  │  ├─ timeline/      ⭐ a tela que define o produto
│  │  │  │  ├─ compromissos/
│  │  │  │  ├─ importacao/
│  │  │  │  ├─ planos/        simulação, roda engine no cliente
│  │  │  │  ├─ assistente/    chat com a IA
│  │  │  │  └─ auth/
│  │  │  ├─ components/ui/    primitivos compartilhados
│  │  │  └─ lib/              cliente de API, storage seguro, ponte Capacitor
│  │  ├─ capacitor.config.ts
│  │  └─ vite.config.ts
│  │
│  ├─ api/                    Fastify
│  │  └─ src/
│  │     ├─ routes/           transporte fino: valida → autoriza → engine → serializa
│  │     ├─ plugins/          auth, db, contexto de RLS, tratamento de erro
│  │     ├─ services/         orquestração com I/O: repositórios, agregador, IA
│  │     └─ server.ts
│  │
│  └─ sync/                   worker — o candidato a Go no futuro
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
│  │  └─ src/
│  │     ├─ entidades/
│  │     └─ api/              contratos de request/response → gera OpenAPI
│  │
│  ├─ db/                     Drizzle: schema, migrations, políticas RLS
│  │  └─ src/rls/             políticas versionadas junto do schema
│  │
│  ├─ importers/              OFX, CSV e adapters de Open Finance
│  │  └─ src/openfinance/     ← trocável por agregador
│  │
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
      web  ──────→  engine  ←──────  api  ──→  db, importers, ai
                  (puro, sem I/O)              (só servidor)
                                                  ↑
                                                sync
```

### Três regras que sustentam a arquitetura

**1. `engine` não conhece I/O.** Nenhum `fetch`, nenhum `pg`, nenhum `fs`. É o que permite rodá-lo no bundle do cliente para a simulação instantânea. Vale prender isso com regra de lint ou `dependency-cruiser` no CI — porque a violação é fácil, tentadora, e só dói meses depois.

**2. `web` nunca importa `db` nem `ai`.** A chave de API e a credencial do banco não podem chegar perto do bundle — que é um zip que qualquer pessoa extrai. Mesma prisão por lint.

**3. Rota é transporte, nunca lógica.** No máximo ~20 linhas. Se a regra for respeitada, a lógica não gruda no Fastify, o `engine` continua testável em memória, e trocar de framework um dia é trabalho de tarde.

`packages/engine` é o coração e o único lugar onde existe aritmética de dinheiro. Se ele ganhar dependência de rede ou de banco, a decisão da seção 3 foi perdida — ele deixa de rodar no cliente e a simulação vira round-trip.

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

## 10. Roadmap revisado

**Fase 0 — Núcleo (grátis)**
Domínio + RLS, import OFX/CSV, entrada manual, inferência de parcelas, linha do tempo de caixa futuro. Web responsiva.
*Saída:* você usa todo dia, sem banco conectado.

**Portão paralelo à Fase 0 — Validar a economia unitária**
Obter de Pluggy, Belvo e Klavi o **preço real por conexão ativa/mês**, a cobertura de instituições e o suporte a Capacitor. Em SaaS o agregador é custo variável por usuário: 3 contas conectadas custam 3×, antes de qualquer receita. Isso define o preço mínimo da assinatura — e é muito melhor descobrir agora do que na Fase 2.

**Fase 0.5 — Casca nativa**
Capacitor, biometria, armazenamento seguro, build nas duas plataformas.
*Por que agora:* descobrir dor de empacotamento com 5 telas, não com 40. E resolve a Guideline 4.2 cedo.

**Fase 1 — Inteligência (pago)**
Motor de planos com testes, chat com tool calling, categorização em cascata.

**Fase 2 — Open Finance (pago)**
**Semana 1: provar o fluxo de deep link em device real.** Depois sandbox → um banco → demais. Reconciliação de parcelas.

**Fase 3 — Automação**
Push de fatura projetada, gasto-fantasma, alerta de comprometimento.

**Fase 4 — Comercial**
Assinatura, CNPJ, LGPD completa, lojas.

---

## 11. Decisões que ficam para depois

- Ionic UI ou Tailwind puro — reversível, decidir ao construir a linha do tempo
- Provedor de Postgres — Supabase é o padrão adotado; a política de RLS foi escrita para não depender dele
- Qual agregador — decidir na Fase 2, **mas validar preço, cobertura e suporte a Capacitor agora**
- Preço da assinatura — depende do custo real do agregador
