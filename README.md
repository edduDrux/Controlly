# Controlly

Copiloto de compromissos financeiros futuros.

O Brasil é o país do "10x sem juros". Compra parcelada é **dívida invisível**: não entra em cadastro de dívida, não aparece como empréstimo, e só se materializa na fatura — um mês de cada vez.

O Controlly trata a **parcela futura como dívida de verdade**, mostra quanto do seu dinheiro dos próximos meses já está comprometido antes de você receber, e usa IA para explicar como sair disso mais rápido.

## Princípios

- **Sem digitação manual.** Importação por arquivo e, depois, Open Finance. Manual existe só como rede de segurança.
- **A IA não calcula.** Todo número vem de um motor determinístico e testado. O modelo de linguagem entende o pedido, chama a ferramenta e explica o resultado.
- **Somente leitura.** O app lê dados bancários. Nunca inicia pagamento, nunca pede senha de internet banking.
- **Parcela é cidadã de primeira classe.** A entidade central do domínio é `Compromisso`, não `Transação`.

## Stack

Web mobile-first em **Next.js** + TypeScript, PostgreSQL com Row Level Security. O motor financeiro é um pacote TypeScript puro, sem I/O, que roda no servidor **e** no cliente — é o que torna a simulação de quitação instantânea. Aplicativo nativo adiado, não cancelado.

## Estado

Fase de concepção. Sem código ainda.

🎯 **[MVP](docs/mvp.md)** — a conexão bancária como core, o spike de dados que vem antes de tudo, e o que fica de fora.

📄 **[Visão de produto](docs/visao-produto.md)** — posicionamento, modelo de domínio, roadmap, riscos e métricas.

🏗️ **[Arquitetura](docs/arquitetura.md)** — stack, estrutura do projeto, multi-tenant, regras de dinheiro, camada de IA e a economia do SaaS.
