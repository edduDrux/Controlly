# Controlly — Visão de Produto

> Documento de conceito. Refina a ideia original: *assistente financeiro pessoal, com IA, que se conecta ao banco para importar gastos e parcelas automaticamente.*

---

## 1. Diagnóstico da ideia original

**O que já está certo e deve ser preservado:**

- **Eliminar a digitação manual.** É a causa nº 1 de abandono em app de finanças pessoais. Está correto colocar isso como requisito, não como "nice to have".
- **Foco em parcelas.** Você citou "o que tenho nas parcelas para pagar" quase de passagem, mas é a parte mais valiosa da ideia. Guarde essa frase — ela vira o produto inteiro (ver seção 2).
- **IA sob demanda** ("com base no que o usuário solicitar"), não IA empurrando conselho não pedido. Postura correta.

**O que precisa mudar:**

| Problema | Refinação |
|---|---|
| "Assistente para gerenciar gastos" é categoria saturada (Mobills, Organizze, e os cemitérios Guiabolso/Olivia). Olhar para o passado é commodity. | Reposicionar para **caixa futuro e quitação de parcelas** (seção 2). |
| Conexão bancária está como requisito central, mas é a peça mais lenta, cara e regulada do projeto. Se ela travar, o projeto inteiro trava. | Colocar atrás de um *adapter* de fonte de dados e **começar por importação de arquivo**. O produto precisa ser útil na semana 2, sem banco nenhum (seção 4 e 7). |
| "IA que monta planos" sugere o LLM calculando o plano. Modelo de linguagem erra aritmética e inventa número com confiança. Em app financeiro isso é falha crítica. | **A IA conversa e orquestra; quem calcula é um motor determinístico.** O LLM chama ferramentas (seção 5). |
| Ambiguidade entre "gerenciar os *meus* gastos" e "o que o *usuário* solicitar". | Uso pessoal e SaaS multiusuário são projetos com custo, risco jurídico e arquitetura diferentes. Precisa ser decidido cedo (seção 10). |

---

## 2. Reposicionamento

**De:** app de controle de gastos com IA.
**Para:** **copiloto de compromissos futuros — o app que mostra quanto do seu salário já está gasto antes de você receber, e como sair disso mais rápido.**

O raciocínio:

1. O Brasil é o país do parcelamento. "10x sem juros" é o padrão cultural de consumo.
2. Compra parcelada é **dívida invisível**: não entra em nenhum cadastro de dívida, não aparece como empréstimo, não sensibiliza score. Ela só se materializa na fatura, um mês de cada vez.
3. Nenhum app do mercado trata a **parcela futura como cidadã de primeira classe**. Todos tratam como "transação que ainda não aconteceu".
4. Portanto a pergunta que ninguém responde bem é: *"quanto do meu dinheiro dos próximos 12 meses já está comprometido?"*

Isso é o Controlly. Categorizar gasto passado é o preço de entrada, não o produto.

**Tela-manifesto:** uma linha do tempo de 6 a 24 meses, mês a mês, com renda prevista contra compromissos já assumidos. O usuário vê a "sombra" das compras que já fez se estendendo pelo futuro. Essa tela sozinha justifica o app.

---

## 3. Modelo de domínio

A decisão estrutural mais importante do projeto: **a entidade central não é `Transação`, é `Compromisso`.**

```
Conta            corrente | poupança | cartão de crédito | carteira
Transação        movimento já realizado, com data, valor, contraparte
Compromisso      obrigação contratada: compra parcelada, empréstimo,
                 financiamento, assinatura, boleto recorrente
  └─ Parcela     nº/total, vencimento, valor, status:
                 prevista → faturada → paga
Fatura           ciclo do cartão: agrupa Transações + Parcelas do período
Categoria
Regra            padrão de estabelecimento → categoria / compromisso
Plano            cenário de quitação gerado (imutável, versionado)
```

Consequências dessa escolha:

- Uma compra em 10x cria **1 Compromisso + 10 Parcelas** no momento da compra, não 10 transações que aparecem ao longo do ano.
- O saldo real do usuário passa a ter duas leituras: **saldo disponível** e **saldo livre** (disponível menos compromissos até a próxima renda). A segunda é a que importa.
- A projeção de caixa vira consulta trivial, não relatório.

### O detalhe técnico que vira diferencial

Open Finance **não entrega o cronograma completo** de parcelas de cartão de crédito de forma confiável. As parcelas futuras de uma compra em 10x frequentemente só aparecem conforme entram em cada fatura.

Ou seja: o Controlly precisa **inferir e projetar** o compromisso a partir de sinais como `PARCELA 3/10`, `03/10`, `PARC 3 DE 10` na descrição, e depois **reconciliar** cada parcela nova que chega contra o compromisso já projetado.

Esse motor de inferência + reconciliação é trabalho real, é defensável, e é exatamente onde os concorrentes são fracos. Vale investir.

---

## 4. Conexão bancária — o caminho realista

### O que existe

No Brasil o caminho regulado é o **Open Finance Brasil**. Você **não** vira participante do Banco Central — a barreira é proibitiva para um projeto individual. Você consome via **agregador**, que é quem tem a autorização.

Nomes a avaliar: **Pluggy**, **Belvo**, **Klavi**, **Iniciador**, **Celcoin**. Pluggy e Belvo costumam ser os mais acessíveis para quem está começando (sandbox e tier de desenvolvedor).

> ⚠️ Preços, cobertura de instituições e regras de consentimento mudam com frequência. Confirme tudo direto na documentação atual de cada fornecedor antes de decidir — nada nesta seção substitui essa checagem.

### Escopo correto: somente leitura

Sua ideia pede só leitura ("somente pegar o que eu gastei"). Isso é ótimo, e vale explicitar por quê:

- **Compartilhamento de dados** (leitura) — é o que você precisa. Escopo mais simples.
- **Iniciação de pagamento (ITP)** — exigiria autorização própria do BACEN. **Fora de escopo, e deve continuar fora.**

Manter o app como *read-only* remove uma classe inteira de risco regulatório, de segurança e de responsabilidade civil. É uma restrição a ser defendida, não uma limitação a ser superada.

### Limitações reais a projetar desde já

| Limitação | Como o produto absorve |
|---|---|
| Consentimento expira (na ordem de até 12 meses) e exige renovação explícita | UX de renovação antecipada, com aviso antes de quebrar a sincronização |
| Sincronização não é tempo real | Nunca prometer saldo "ao vivo"; sempre mostrar carimbo de "atualizado em" |
| Cobertura varia por instituição | Fallback manual sempre disponível, em toda tela |
| Nome de estabelecimento vem sujo (`PAG*XYZ`, `MP *LOJA`) | Camada de normalização + dicionário de merchants (seção 5) |
| Parcelas futuras incompletas | Motor de inferência + reconciliação (seção 3) |

### Fontes de dados como estratégia, não como integração única

```
FonteDeDados (porta)
 ├─ ManualSource       digitação rápida — sempre existe, é o fallback universal
 ├─ ArquivoSource      OFX / CSV de extrato e fatura
 ├─ OpenFinanceSource  via agregador
 └─ FaturaPdfSource    (fase posterior, opcional)
```

Todas normalizam para o mesmo modelo de domínio. Trocar de agregador vira trocar um adapter.

### Uma proibição explícita

**Nunca** pedir a senha do internet banking do usuário para fazer raspagem. É risco jurídico, quebra os termos de uso do banco, transfere a responsabilidade de fraude para você, e é um caminho sem futuro regulatório. Se um fornecedor oferecer isso como atalho, é sinal de alerta sobre o fornecedor.

---

## 5. A camada de IA

### Regra de ouro

> **A IA não calcula. A IA entende o pedido, chama a ferramenta certa e explica o resultado.**

Todo número que o usuário vê sai de código determinístico e testado. O LLM nunca produz um valor monetário por conta própria.

### Motor de planos (código puro, sem IA)

Funções testáveis, cada uma com casos de teste próprios:

- **Estratégias de quitação:** avalanche (maior custo efetivo primeiro), bola de neve (menor saldo primeiro), híbrida.
- **Antecipação de parcelas.** O consumidor tem direito a **redução proporcional dos juros** na quitação antecipada (CDC, art. 52, §2º) — direito muito subutilizado e que rende recomendação concreta.
- **A nuance brasileira que quase todo app erra:** antecipar parcela **sem juros** normalmente **destrói valor**. Se a dívida tem custo efetivo zero, o dinheiro rende mais parado no CDI do que quitando. O motor precisa comparar `custo efetivo da dívida` contra `rendimento livre de risco` e, quando for o caso, **recomendar não antecipar**. Um app que diz "não quite isso ainda, deixe rendendo" ganha confiança instantânea.
- **Projeção de caixa** mês a mês e **capacidade de pagamento** (renda − essenciais − compromissos).
- **Simulação de cenário:** "e se eu colocar R$ 500 a mais por mês?"

### Camada conversacional (LLM com tool calling)

O modelo recebe o pedido em linguagem natural e orquestra ferramentas:

```
listar_compromissos(filtros)
projetar_caixa(meses)
simular_plano(estrategia, aporte_extra, prazo_alvo)
comparar_cenarios(a, b)
explicar_parcela(id)
```

Fluxo: *"quero estar livre do cartão até dezembro"* → o modelo traduz em parâmetros → chama `simular_plano` → recebe números do motor → **explica o resultado e os trade-offs** em português claro.

O valor do LLM está na **tradução e na explicação**, não na conta.

### Guardrails obrigatórios

- Número nunca vem do modelo — só de retorno de ferramenta.
- **Não enviar extrato bruto ao modelo.** Mandar agregados e recortes mínimos. Isso é privacidade, custo e qualidade de contexto ao mesmo tempo.
- Reduzir/mascarar identificadores pessoais antes do envio.
- Enquadramento como **educação financeira**, não recomendação de investimento — mantém o projeto longe das regras da CVM sobre consultoria e análise de valores mobiliários. Revise esse enquadramento com apoio jurídico se o projeto virar produto público.
- Log de qual ferramenta gerou cada número exibido, para auditoria e depuração.

### Categorização: onde a IA custa caro se for mal feita

Chamar o LLM para toda transação é caro e lento. A cascata correta:

```
1. Regra do usuário          (instantâneo, grátis)
2. Dicionário de merchants   (cache compartilhado, grátis)
3. LLM                       (só para desconhecidos)
4. → resultado vira regra    (nunca se paga duas vezes pelo mesmo estabelecimento)
```

Na prática o passo 3 tende a sumir depois das primeiras semanas de uso.

---

## 6. Privacidade e LGPD

Dado financeiro é dado sensível na prática, mesmo quando não é na letra da lei. Tratar como sensível.

- **Criptografia em repouso** para transações e, obrigatoriamente, para tokens de acesso do agregador (cofre de chaves, nunca em variável de ambiente em texto puro em produção).
- **Nunca logar payload** de transação ou de resposta do agregador.
- **Consentimento granular** para IA: usar o app sem IA deve ser possível.
- **Direito de exportar e apagar tudo**, de verdade, incluindo revogar o consentimento no agregador.
- Retenção definida e curta para o que não precisa ficar.

Se for uso estritamente pessoal (self-hosted), boa parte disso simplifica — mas **a criptografia de tokens continua obrigatória**, porque um token vazado dá acesso de leitura à sua vida financeira inteira.

---

## 7. Roadmap

> Revisado após as decisões de escopo e plataforma. A mudança mais honesta: para uso pessoal, a Fase 0 já seria um produto. **Para SaaS, a Fase 0 sozinha não é vendável** — ninguém paga por um app onde ainda importa arquivo à mão. A ordem continua certa, mas o papel de cada fase muda: a Fase 0 vira o **plano gratuito**, não o lançamento.

Detalhamento técnico de cada fase em **[arquitetura.md](arquitetura.md)**.

**Fase 0 — Núcleo (grátis)**
Domínio + RLS, import OFX/CSV, entrada manual, inferência de parcelas, linha do tempo de caixa futuro. Web responsiva.
*Papel:* derisking e dogfooding. Você é o usuário zero e valida o modelo contra a sua vida financeira real, onde os casos estranhos aparecem.
*Saída:* você usa todo dia, sem banco conectado.

**Portão paralelo — Validar a economia unitária**
Preço real por conexão/mês dos agregadores e cobertura de instituições. Em SaaS o agregador é custo variável por usuário e define o preço mínimo da assinatura. **É portão, não tarefa:** se a conta não fechar, o modelo de negócio muda antes de existir código de integração.

**Fase 1 — Inteligência (pago)**
Motor de planos com testes, chat com tool calling, categorização em cascata.
*Saída:* o app te diz algo sobre seu dinheiro que você não sabia.

**Fase 2 — Open Finance (pago)**
Sandbox → um banco → demais. Reconciliação de parcelas. Na web o consentimento é redirect comum, o que tirou o maior risco técnico que o projeto tinha.
*Para SaaS esta fase é existencial, não opcional:* é ela que entrega a promessa de "sem digitação".

**Fase 3 — Automação**
Alerta de fatura projetada, gasto-fantasma, comprometimento excessivo — por e-mail e web push.

**Fase 4 — Comercial**
Assinatura, CNPJ, LGPD completa.

**Depois — Mobile, se a tração justificar**
Ver Apêndice A da arquitetura.

---

## 8. Riscos e mitigação

| Risco | Mitigação |
|---|---|
| **Custo do agregador inviabiliza o preço de assinatura** | Portão paralelo à Fase 0. Validar antes de construir a integração |
| **Vazamento de dados financeiros de terceiros** | RLS no banco, criptografia de tokens em cofre, nunca logar payload, auditoria. Escopo somente leitura limita o dano máximo |
| Custo de IA por usuário sem teto | Cascata de categorização, cache de merchants, rate limit por usuário, agregados em vez de extrato bruto |
| Consentimento expira e o usuário some | Renovação proativa com aviso antecipado; o app degrada para manual, não quebra |
| LLM inventa número | Tool calling obrigatório + testes do motor + log de origem de cada valor exibido |
| Concorrente com capital copia | O fosso não é a feature, é o motor de inferência e reconciliação de parcelas — exige tempo e dados reais |
| Escopo infinito (investimento, imposto, orçamento familiar…) | O produto é **compromissos futuros**. Pedido fora disso vai para o backlog |
| Regra regulatória muda | Tudo de Open Finance isolado atrás do adapter |

---

## 9. Métricas que importam

**Produto**
- % do comprometimento futuro visível — quanto dos próximos 12 meses o app enxerga
- Meses de antecipação de quitação obtidos com um plano seguido
- Transações digitadas manualmente por mês (deve tender a zero após a Fase 2)
- Taxa de acerto da categorização automática
- **Planos gerados vs. planos seguidos** — a segunda é a única que prova valor

**Negócio (a partir da Fase 4)**
- Custo de agregador + IA por usuário ativo
- Margem por assinante
- Conexões bancárias por usuário — dirige o custo
- Conversão do gratuito para o pago
- Retenção no mês 3 — em finanças pessoais é onde o abandono aparece

---

## 10. Decisões tomadas

| Pergunta | Resposta |
|---|---|
| Uso pessoal ou produto? | **Pessoal primeiro, SaaS comercial como destino.** Multi-tenant no schema desde o commit 1 |
| Plataforma | **Web mobile-first.** Aplicativo nativo adiado, não cancelado |
| Framework | **Next.js** (App Router) |
| Agregador | Definir na Fase 2 — mas validar preço e cobertura desde já |
| Modelo de IA | API hospedada, executada **somente no servidor** |
| Stack | **TypeScript full-stack** — o motor roda no servidor e no cliente com o mesmo código |
| Rust/Go no backend | **Não.** Carga é I/O-bound e o motor precisa rodar no cliente. Reconsiderar só para o worker de sync, com medição |

O detalhamento técnico dessas escolhas, e o que cada uma impõe, está em **[arquitetura.md](arquitetura.md)**.

Duas consequências que já valem aqui:

- **"SaaS depois" não permite arquitetura de usuário único agora.** Isolamento de dados financeiros não se retrofita.
- **O plano gratuito não pode incluir conexão bancária** — é o custo que escala e que você não controla. O gratuito é manual/OFX + linha do tempo; o pago é banco + IA. O paywall tem a mesma forma do roadmap.

---

## Resumo em uma frase

**Controlly é o app que trata parcela futura como dívida de verdade, mostra quanto do seu futuro já está comprometido, e usa IA para explicar como sair disso — com os cálculos feitos por código testado, não pelo modelo de linguagem.**
