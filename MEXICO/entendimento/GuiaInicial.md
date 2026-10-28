# Projeto México — guia para quem está começando

Escrito em 18/10/2026. Se você acabou de entrar no projeto, comece por aqui. Depois leia `Pendencias.md` (a lista do que falta) e `OperacaoMexico.md` (o detalhe técnico).

## O que é o projeto, em uma frase

A QAS vai oferecer no México o mesmo tipo de serviço que a SaQ oferece no Brasil: contas para os clientes receberem e enviarem dinheiro de forma instantânea. No Brasil isso é feito com Pix. No México, com **SPEI**.

## Os termos que você vai ouvir o tempo todo

| Termo | O que significa |
|---|---|
| **SPEI** | O "Pix do México". Sistema de transferência instantânea do Banco do México. |
| **SPEI IN** | Dinheiro entrando na conta do cliente. |
| **SPEI OUT** | Dinheiro saindo da conta do cliente. |
| **CLABE** | O "número da conta" no México. Tem 18 dígitos. É o que o cliente passa para alguém pagar para ele, parecido com a chave Pix. |
| **Fintoc** | A empresa parceira. Ela tem a conexão com o SPEI. A QAS não fala com o banco central mexicano, fala com a Fintoc. É o papel que JD, ONZ e outros provedores fazem para a SaQ no Brasil. |
| **Entity** | Na Fintoc, é o "dono legal" da conta. Vai existir uma só: a QAS. |
| **Account** | Na Fintoc, é uma conta com saldo. Cada cliente nosso vai ter uma `Account` própria, debaixo da Entity da QAS. |
| **Transfer** | Na Fintoc, é uma movimentação de dinheiro, entrando ou saindo. |
| **Webhook** | Um aviso automático que a Fintoc manda para o nosso sistema quando algo acontece, por exemplo "entrou dinheiro na conta X". Nós não ficamos perguntando; ela avisa. |
| **Bolsão e mirror** | Como a SaQ funciona hoje no Brasil. O **bolsão** é a conta real no provedor, onde o dinheiro fica de fato. A **mirror** é a conta virtual de cada cliente, que só existe no nosso sistema. |
| **Ledger** | O nosso registro de saldo e movimentações. Hoje fica no Valkey (um banco em memória, parecido com Redis). |
| **KYC** | "Conheça seu cliente". É a análise de documentos e identidade antes de abrir conta. |
| **Idempotência** | Garantia de que, se o mesmo pedido chegar duas vezes, ele só vale uma. Evita creditar ou debitar em duplicidade. |
| **req/s** | Requisições por segundo. Quantas chamadas podemos fazer à Fintoc em um segundo. Hoje o limite é 10. |
| **Backoffice** | O painel interno da QAS para a equipe administrar clientes, contas, taxas e usuários. |
| **PRD** | O documento de produto que diz o que deve ser construído. O nosso está em `PRD-MEXICO-V0.pdf`. Ainda é rascunho. |

## Como vai funcionar, sem detalhe técnico

1. **A QAS tem uma conta só na Fintoc.** Ela é a dona de tudo. Já passou pelo KYC.
2. **Cada cliente ganha uma conta própria dentro dela**, criada por API, sem precisar mandar documento para a Fintoc. A Fintoc controla o saldo de cada uma.
3. **Cada conta nasce com uma CLABE.** É o que o cliente usa para receber.
4. **Quando entra dinheiro**, a Fintoc avisa por webhook. Nós registramos, descontamos a nossa taxa e creditamos o cliente no nosso ledger.
5. **Quando o cliente quer enviar**, nós conferimos o saldo e os limites, reservamos o valor e pedimos à Fintoc para enviar. Ela avisa depois se deu certo ou não. Se falhar, devolvemos o valor.
6. **A taxa é nossa, não da Fintoc.** Calculamos no nosso sistema e o valor vai para uma conta da QAS.
7. **De tempos em tempos, conferimos** se o saldo que a Fintoc mostra bate com o nosso ledger. Diferença vira um registro para alguém analisar, nunca é corrigida sozinha.

## Diferença em relação ao Brasil que mais confunde

No Brasil, o dinheiro de todos os clientes fica junto no bolsão e só o nosso sistema sabe quanto é de cada um. No México, a Fintoc também sabe: cada cliente tem uma `Account` com saldo separado. Isso traz uma consequência importante: a nossa taxa precisa sair da conta do cliente e ir para a conta da QAS por uma transferência real na Fintoc. Ainda não decidimos se fazemos isso em cada operação ou uma vez por dia.

## O que será entregue nesta primeira fase (V0)

| Para quem | O que |
|---|---|
| Cliente (portal web) | Login, abrir conta, ver saldo, ver a CLABE para receber, enviar dinheiro, consultar extrato. Há dúvida se o envio entra agora ou só o extrato. |
| Equipe QAS (backoffice) | Aprovar cadastros, ver e bloquear contas, cadastrar taxas, criar usuários, acompanhar webhooks e extratos. |
| Integradores (API) | Os mesmos recursos do portal, via API, com documentação. |

## Onde o projeto está hoje

- **Nenhuma linha de código foi escrita.**
- As decisões de desenho já estão tomadas: uma conta QAS, uma `Account` por cliente, taxa calculada por nós.
- Estamos esperando a Fintoc em vários pontos, e várias decisões de produto ainda não foram fechadas.

## O que está travando, em linguagem simples

**Dependemos da Fintoc (quem fala com ela é o Caio Cesar):**
- Ainda não temos as chaves de acesso ao ambiente de teste. Sem elas, não dá para testar nada.
- Precisamos confirmar que o contrato permite uma `Account` por cliente, e não só uma CLABE por cliente. São modelos diferentes.
- Pedimos para subir o limite de 10 para 50 req/s. Ainda não foi formalizado nem aprovado.
- Não sabemos quanto custa cada transferência, nem se mover dinheiro entre contas nossas custa algo.
- Perguntas regulatórias: onde o dinheiro fica guardado, se aceitam clientes de apostas.

**Dependemos de decisões internas:**
- O portal vai ter envio de dinheiro ou só extrato? Uma reunião disse uma coisa, o PRD diz outra.
- Como vai ser o nosso cadastro de clientes, já que a Fintoc não analisa documentos do cliente final.
- Quanto vamos cobrar de taxa e como.
- Quais limites de valor por operação e por dia.
- Quem é o primeiro cliente que vai usar.
- Se vamos criar serviços novos ou reaproveitar o que a SaQ já tem.

**O PRD precisa de revisão:** faltam regras, a numeração pula e cita telas que não estão descritas.

## O que já dá para fazer sem esperar ninguém

- Desenhar as tabelas e o ledger do México, baseado no que já existe de mirror na SaQ.
- Escrever o código que conversa com a Fintoc: autenticação, assinatura das chamadas e chave de idempotência. A documentação pública dela é suficiente.
- Escrever o receptor de webhooks, com validação da assinatura e proteção contra duplicidade.
- Montar a máquina de estados do envio: em processamento, concluído, recusado, falhou. Os avisos da Fintoc podem chegar fora de ordem, então isso precisa ser pensado desde o início.

## Quem é quem

| Pessoa | Papel |
|---|---|
| Caio Cesar | Contato com a Fintoc. Credenciais, limite de req/s, dúvidas contratuais. |
| Gabriel Barros | Autor do PRD. Produto. |
| Rodrigo Barros | Desenvolvimento. Revisão da doc da Fintoc. |
| Matheus Falcão | Liderança técnica. Decide escopo e alocação de pessoas. |
| Jose Vieira Junior | Negócio. Define o piloto e contrata QA. |
| Munir | Admin do grupo de WhatsApp do México. |

## Por onde começar na prática

1. Ler este arquivo e `Pendencias.md`.
2. Pedir para entrar no grupo de WhatsApp do México (Munir adiciona).
3. Ler as páginas da Fintoc listadas no fim de `OperacaoMexico.md`. A mais importante é "Run a wallet for end users".
4. Olhar como a SaQ faz hoje no Brasil: `AccountBank` e `AccountBankMirror` no `saq-repository`, e o saldo no `saq-cache-client`. O México vai seguir a mesma ideia.
5. Perguntar ao Caio se as chaves de teste já saíram.
