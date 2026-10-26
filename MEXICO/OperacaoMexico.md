# Operação México — o que sabemos até agora

Consolidado em 16/10/2026 a partir de quatro fontes:
- o PRD "México – fase V0" (`PRD-MEXICO-V0.pdf`);
- a reunião de 24/09/2026 (`Granola meeting - Mexico 24-09-2026.txt`);
- a documentação pública da Fintoc;
- o código atual da SaQ (bolsão e mirror).

Ainda não existe código da frente México no workspace.

## 1. Resumo

- A QAS vai operar contas no México, recebendo e enviando dinheiro por **SPEI** (o sistema de transferências instantâneas do Banco de México, equivalente ao Pix).
- O provedor é a **Fintoc**, plataforma de pagamentos e infraestrutura A2A (account-to-account) com operação no Chile e no México. Usamos o produto **Business Accounts API v2** dela.
- Modelo de contas: **uma única conta SAQ na Fintoc** (a Entity da QAS) e **uma `Account` Fintoc por cliente** debaixo dela (padrão B da Fintoc). O cliente final não passa por onboarding na Fintoc.
- A **tarifa** é calculada na SaQ, não pela Fintoc, e vai para uma subconta da QAS.
- A V0 tem três entregas: portal web do cliente, backoffice e API pública documentada.
- O PRD é da Squad CRIPTO (autor Gabriel Barros, 30/09/2026) e ainda é rascunho.

## 2. Escopo da V0 (PRD)

| Frente | Itens |
|---|---|
| Portal do cliente (o app copia depois) | Login; abertura de conta e acompanhamento do onboarding; saldo; dados para receber (CLABE); cobrança SPEI (válida por 72h); envio SPEI com revisão de taxa e total antes de confirmar; extrato com filtros, paginação e detalhe |
| Backoffice (12 telas, no padrão das IPs atuais) | Dashboard; onboarding; contas (dados, extrato, usuários, SPEI IN, SPEI OUT, webhook, tarifas); grupo de acesso; usuários |
| API pública | Autenticação; onboarding; consulta de conta; SPEI IN; SPEI OUT; extrato; webhooks; documentação |

- **Perfis:** só dois, `Cliente:web` (portal e app, sem acesso ao backoffice) e `backoffice:admin`.
- **Critério de aceite:** 100% em SPEI IN, SPEI OUT, abertura de conta, extratos e backoffice.

### Regras de negócio que mais pesam no backend

| Regra | O que exige |
|---|---|
| RN-013 / RN-026 | O mesmo SPEI IN ou o mesmo pedido de envio, recebido de novo, não credita nem debita duas vezes |
| RN-019 / RN-022 | Saldo suficiente para valor + taxa; o valor fica reservado enquanto o envio processa |
| RN-023 a RN-025 | Estados em processamento / concluída / não efetivada. Só fica concluída com a confirmação do parceiro. Se falhar, devolve a reserva. Se o resultado for incerto, fica em acompanhamento |
| RN-029 a RN-032 | A operação grava a taxa que valia no momento. Mudar a taxa não altera o passado. Conta sem taxa cadastrada não é isenta |
| RN-037 / RN-039 | Saldo inicial + movimentos = saldo final. Correção só por lançamento de ajuste, sem editar o original |
| RN-054 / RN-055 | Reenviar webhook não gera movimento; falha de entrega não desfaz operação confirmada |
| RN-059 | A API segue as mesmas regras do portal (saldo, taxa, situação da conta, permissão) |
| RN-066 | Divergência com o parceiro fica registrada para análise, sem mexer no saldo automaticamente |

## 3. Decisões tomadas

| Data | Decisão | Quem |
|---|---|---|
| 24/09 | Uma única conta SAQ na Fintoc; subcontas abertas por API, sem onboarding nem documentos do cliente final | Caio Cesar, Rodrigo Barros, Gabriel |
| 24/09 | Pedir à Fintoc o aumento de 10 para 50 req/s | Rodrigo Barros, com Júnior e Caio |
| 24/09 | Tarifa implementada na SaQ, sem usar a Fintoc para isso; a tarifa vai para uma subconta SAQ | Caio, Júnior |
| 24/09 | Portal web só com extrato, sem transferência | Matheus Falcão |
| 16/10 | Subconta = uma `Account` Fintoc por cliente (padrão B) | Rodrigo |

## 4. Fintoc

### Modelo de dados

| Objeto | O que é | Uso na QAS |
|---|---|---|
| `Entity` | Titular legal, com KYC. Cada organização tem uma Entity raiz | A QAS. Uma só |
| `Account` | Saldo (`available_balance`), sem cheque especial. Estados `active`, `blocked`, `closed` | Uma por cliente + uma de tarifas da QAS |
| Account Number | A **CLABE** (18 dígitos). Cada `Account` nasce com uma CLABE raiz (`is_root`) e pode ter outras, com `metadata`, faixa de valor aceita, desativação e exclusão | Dados para receber do cliente |
| `Transfer` | Uma movimentação SPEI, de entrada ou de saída | SPEI IN / OUT, tarifa, transferência entre clientes |
| `Movement` | Lançamento no saldo, com o saldo depois de cada operação | Conciliação |

### As três formas de montar as contas (guia "run a wallet for end users")

| Padrão | Estrutura | Saldo por cliente na Fintoc | KYC por cliente | Comprovante no nome do cliente |
|---|---|---|---|---|
| A | Uma `Account`, uma CLABE por cliente | Não (só no nosso ledger) | Não | Não |
| **B (escolhido)** | Uma `Account` por cliente, debaixo da Entity da QAS | **Sim** | Não | Não |
| C | Uma `Entity` por cliente | Sim | Sim (só empresa mexicana, até 5 dias úteis) | Sim |

O padrão B é "uma conta só" do ponto de vista jurídico: um titular, um KYC, e o comprovante sai no nome da QAS. Mas a Fintoc controla um saldo separado por cliente. Por isso, dinheiro só passa de uma `Account` para outra por `Transfer`. A doc não diz onde os recursos ficam guardados fisicamente.

### Integração

- **Autenticação:** a chave secreta (`sk_live_...` / `sk_test_...`) vai no header `Authorization`. O modo (teste ou produção) é definido pela chave usada.
- **JWS:** toda chamada de Business Accounts que move dinheiro leva o header `Fintoc-JWS-Signature`.
  - A chave privada fica com a QAS; a pública é cadastrada no Dashboard da Fintoc.
  - Cada assinatura tem `nonce` e `ts`, então é de uso único e expira.
  - Pedido sem assinatura, expirado, repetido ou alterado é recusado.
- **Idempotência:** o header `Idempotency-Key` em `POST /v2/transfers` evita duplicar o envio. Se repetir a chave, a Fintoc devolve o resultado original.
- **Valores:** sempre inteiros em centavos (MXN 590,13 → `59013`).
- **Lista de IPs autorizados (opcional):** configurada no Dashboard.
- **Webhooks:** assinados com HMAC SHA-256 no header `Fintoc-Signature`, que deve ser validado sempre. **Podem chegar fora de ordem.**
- **Limite de requisições:** 10 criações de `Transfer` por segundo por organização. Aumento é pedido ao suporte.

### Endpoints que vamos usar

| Para quê | Endpoint |
|---|---|
| Criar a subconta do cliente | `POST /v2/accounts` com `entity_id` da QAS (ativa na hora) |
| Consultar a subconta | `GET /v2/accounts/{id}`. O `PATCH` só altera a `description`; para parar o recebimento, desativar a CLABE em `PATCH /v2/account_numbers/{id}` |
| CLABEs extras, limites de recebimento, desativar | `/v2/account_numbers` (`options.min_amount` / `options.max_amount`) |
| Enviar SPEI | `POST /v2/transfers` (`account_id`, `amount`, `currency`, `counterparty.account_number`, `reference_id`, `comment`, `metadata`) |
| Devolver um SPEI IN | "Return a transfer" |
| Validar a CLABE de destino antes de enviar | Account verification |
| Movimentos e extrato mensal | `/v2/movements`, `GET /v2/accounts/{id}/statements` |
| Simular SPEI IN (teste) | `POST /v2/simulate/receive_transfer` |

### Estados e eventos de `Transfer` (México)

| Status | Evento | Significado |
|---|---|---|
| `pending` | — | Em processamento. O valor já sai do `available_balance` |
| `succeeded` | `transfer.inbound.succeeded` / `transfer.outbound.succeeded` | Liquidada |
| `rejected` | `transfer.outbound.rejected` | Recusada pelo Banxico ou pelo banco de destino (motivo nos códigos SPEI). Estado final |
| `failed` | `transfer.outbound.failed` | Erro no processamento. O saldo volta |
| `return_pending` / `returned` | — | Devolução em andamento / devolvida |
| `reject_failed` | — | A rejeição não foi concluída |

No SPEI IN, o webhook traz `data.account_id`, `account_number.metadata`, `tracking_key` (a clave de rastreo), `reference_id` e `receipt_url`, que é o CEP, o comprovante do Banxico.

## 5. Como a SaQ faz hoje (Brasil) e como fica no México

### Hoje: bolsão + mirror

- **Bolsão** (`AccountBank` com `is_bolsao`): a conta real no provedor (JD, ONZ, S3...). O dinheiro fica nela, e o DICT só conhece ela.
- **Mirror** (`AccountBankMirror`, com `bolsao_id`): a conta virtual de cada cliente, com company, whitelabel e onboarding próprios.
- **Saldo:** fica no Valkey, nas chaves `{account_mirror:<id>}:balance` e `{account:<id>}:balance`. É alterado por scripts Lua atômicos (`saq-cache-client`).
  - Cada operação grava uma chave própria, por exemplo `cashin:credit:<end_to_end_id>`. Se o mesmo evento chegar de novo, o saldo não muda.
  - O débito desconta o saldo bloqueado e é recusado se o disponível ficar negativo.
- **Recebimento:** a chave Pix leva à mirror. A transação grava a mirror e o bolsão, e o crédito vai **só para a mirror**, já sem a tarifa (`process_pix_cashin_pix_key.py`).
- **Envio:** a SaQ debita da mirror o valor + tarifa, com limites (`validate_limits.py`). Depois o provedor envia pela conta do bolsão. Se der erro, o débito é estornado.
- **Transferência interna:** quando o destino também é conta SaQ, não sai Pix. É débito e crédito no ledger (`create_internal_transfer.py`).
- **Conciliação:** o extrato da conta no provedor (`PiAccountStatementEntry`) é comparado com o ledger, e o saldo consolidado soma as mirrors de cada bolsão.

### México: mapeamento para o padrão B

| SaQ (Brasil) | México (Fintoc, padrão B) |
|---|---|
| Bolsão | Entity da QAS + `Account` de tarifas/receita da QAS |
| Mirror (saldo só na SaQ) | Mirror **1:1 com uma `Account` Fintoc**. Guardar `fintoc_account_id` e a CLABE |
| Chave Pix → mirror | `data.account_id` do webhook → mirror |
| `end_to_end_id` como chave de idempotência | `transfer.id` / `tracking_key` |
| Envio sai do bolsão | Envio sai da `Account` **do cliente** |
| Tarifa = lançamento interno | Tarifa = `Transfer` real da `Account` do cliente para a `Account` da QAS |
| Transferência interna = só ledger | `Transfer` entre as duas `Account`s |
| Um extrato (bolsão) contra a soma das mirrors | Um extrato por cliente (`Movement`s da `Account`) contra a mirror |
| Status da mirror | Status da mirror + desativar a CLABE (`PATCH /v2/account_numbers/{id}`, `status: disabled`). A `Account` não tem bloqueio por API |

O ledger no Valkey continua: é ele que faz a verificação rápida de saldo, o bloqueio, os limites e a idempotência do nosso lado. A Fintoc passa a ser um segundo registro do saldo de cada cliente, conferido na conciliação.

## 6. Fluxos propostos

### Abertura de conta
1. O cliente é aprovado no cadastro da QAS (o que a QAS exige ainda está aberto, ver seção 8).
2. A QAS chama `POST /v2/accounts` com o `entity_id` da QAS e recebe `acc_...` + a CLABE raiz.
3. A QAS cria a mirror com `fintoc_account_id` e a CLABE, e inicia o saldo no Valkey.

### SPEI IN
1. O pagador envia para a CLABE do cliente.
2. Chega o webhook `transfer.inbound.succeeded`. A QAS valida a `Fintoc-Signature`.
3. Se o `transfer.id` já foi processado, encerra. Isso cobre a RN-013.
4. A QAS busca a mirror pelo `data.account_id`, calcula a tarifa e grava a transação.
5. Credita a mirror com o valor menos a tarifa. A tarifa fica reservada para a cobrança da seção "Tarifa".
6. Notifica o cliente pelo webhook da QAS.

### SPEI OUT
1. O cliente pede o envio no portal ou na API. A QAS mostra valor, taxa e total (RN-020).
2. A QAS debita da mirror o valor + tarifa, com limites (RN-019 e RN-022).
3. `POST /v2/transfers` saindo da `Account` do cliente, com `Idempotency-Key` = id da transação, JWS e `metadata` com o id interno.
4. A operação fica `pending` até o webhook:
   - `succeeded`: confirma.
   - `rejected` ou `failed`: estorna o débito na mirror.
   - Sem resposta: a operação continua em acompanhamento, consultando `GET /v2/transfers/{id}` (RN-025).
5. Os eventos podem chegar fora de ordem, então a máquina de estados só avança para estados finais.

### Tarifa
O valor entra cheio na `Account` do cliente na Fintoc, e a tarifa precisa sair dela por `Transfer`. Há duas opções:

| Opção | Prós | Contras |
|---|---|---|
| Um `Transfer` por operação | Simples; o saldo da Fintoc bate com o da SaQ o tempo todo | Dobra as chamadas e consome o limite de req/s; se a Fintoc cobrar por transferência, cada tarifa gera custo |
| Varredura periódica (ex.: diária por cliente) | Uma chamada por cliente por dia | Até a varredura, o saldo da Fintoc fica maior que o disponível na SaQ. O envio tem que ser validado pelo saldo da SaQ |

A transferência em lote da Fintoc existe no México, mas **só pelo Dashboard, sem API**. Uma varredura automática teria que chamar `POST /v2/transfers` cliente a cliente.

### Conciliação
A QAS compara, por cliente, os `Movement`s da `Account` com os lançamentos da mirror. Divergência vira um registro para análise, sem ajuste automático (RN-066).

## 7. Volume e limite de requisições

| Volume mensal | Média por segundo |
|---|---|
| 5 milhões | ~1,9 |
| 50 milhões | ~19 |
| 100 milhões | ~39 |

- 10 req/s dão cerca de 864 mil por dia. O limite vale só para **criação de `Transfer`**: envios dos clientes, cobranças de tarifa e transferências internas. O SPEI IN chega por webhook e não conta.
- Até 5 milhões/mês, o limite só aperta nos picos. Em 50 milhões, só os envios já passam de 10/s.
- O número a pedir à Fintoc é o **pico de `Transfer` por segundo**, já incluindo a estratégia de tarifa.

Preço citado na reunião:
- O preço da Fintoc cai a partir de 600 mil transações.
- Alvo de 0,40 por transação no volume alto (cerca de 56 milhões).
- A moeda (provavelmente MXN) e se é custo ou preço ao cliente não ficaram claros.

## 8. Em aberto

### Para a Fintoc (via Caio Cesar)
- [ ] Aprovação do aumento de 10 para 50 req/s, por e-mail.
- [ ] Credenciais e chaves de teste da conta SAQ.
- [ ] Confirmar que "subconta" no acordo é `Account` (padrão B), e não só CLABE (padrão A).
- [ ] Custo por `Transfer` e se `Transfer` entre `Account`s da mesma organização tem custo e é imediato no México.
- [ ] Limite de quantidade de `Account`s por Entity (a doc só fala em 1 milhão de CLABEs por organização).
- [ ] Onde os recursos ficam guardados e em nome de quem (pergunta regulatória).
- [ ] Se o "uso da conta" aceita clientes de apostas, caso o piloto seja uma bet.
- [ ] Como o "operador chinês" opera direto na conta principal. Pela doc, parece ser o padrão A (CLABE por cliente numa `Account` só).

### Para produto e liderança
- [ ] **Portal:** só extrato (reunião de 24/09) ou com SPEI OUT e cobrança (PRD de 30/09)?
- [ ] **Onboarding:** a QAS faz o próprio cadastro e análise com documentos (como no PRD), já que a Fintoc não faz KYC do cliente? Quais perfis entram: pessoa física, empresa, estrangeiro (PS01, PS08)?
- [ ] O comprovante SPEI no nome da QAS é aceitável para os clientes? (Só o padrão C mostra o nome do cliente.)
- [ ] Estratégia de cobrança da tarifa: por operação ou varredura.
- [ ] Modelos e valores de tarifa (PS05), limites e estados (PS02, PS03), matriz de permissões (PS06), catálogo de webhooks (PS07).
- [ ] Primeiro cliente piloto.
- [ ] Stack: serviços novos (como `anticipation-apps`) ou reaproveitar o core da SaQ (usuários, scopes, backoffice, `saq-cache-client`)? Existe o "Plano de Desenvolvimento Técnico — México v.0" citado no PRD?

### Pontos a corrigir no PRD (rascunho)
- Faltam RN-015, RN-016, RN-063 e PS04.
- A numeração das seções pula (4 → 5.1 → 15.2).
- "Login administrativo" aparece duas vezes.
- O texto fala em "11 telas do portal", mas só 4 estão descritas.
- Há referência a "sugestões da seção 6" que não existem.

## 9. Pessoas e próximos passos (reunião de 24/09)

| Pessoa | Papel / ação |
|---|---|
| Caio Cesar | Contato técnico com a Fintoc. Pedir o aumento de req/s, compartilhar credenciais, investigar o operador chinês |
| Gabriel Araujo Almeida Barros | Autor do PRD. Levantar pontos técnicos da doc da Fintoc e enviar ao Caio |
| Rodrigo Barros | Revisar a doc da Fintoc com o Mateus (tarifas e limites) |
| Jose Vieira Junior | Definir o piloto; contratar QA |
| Matheus Falcão | Alocar mais um dev para o México |
| Munir | Admin do grupo de WhatsApp do México; adicionar Rodrigo e Gabriel |

## 10. Fontes

- `PRD-MEXICO-V0.pdf` (Gabriel Barros, 30/09/2026)
- `Granola meeting - Mexico 24-09-2026.txt`
- Fintoc (consultado em 16/10/2026):
  - [Run a wallet for end users](https://docs.fintoc.com/guides/business-accounts/use-cases/run-a-wallet-for-end-users)
  - [Business Accounts overview](https://docs.fintoc.com/guides/business-accounts/overview/index)
  - [Account](https://docs.fintoc.com/guides/business-accounts/overview/account-data-model)
  - [Account Number](https://docs.fintoc.com/guides/business-accounts/overview/account-number-data-model)
  - [Transfer](https://docs.fintoc.com/guides/business-accounts/overview/transfer-data-model)
  - [Send transfers](https://docs.fintoc.com/guides/business-accounts/outbound-transfers/index)
  - [Receive transfers](https://docs.fintoc.com/guides/business-accounts/inbound-transfers/index)
  - [Create more accounts](https://docs.fintoc.com/guides/business-accounts/manage-accounts/creating-more-accounts)
  - [Batch transfers](https://docs.fintoc.com/guides/business-accounts/outbound-transfers/batch-transfers)
  - [KYC requirements in Mexico](https://docs.fintoc.com/guides/business-accounts/kyc-requirements/mexico)
  - [Security controls](https://docs.fintoc.com/guides/resources/security-controls)
  - Índice completo: <https://docs.fintoc.com/llms.txt>
- Código SaQ:
  - `shared/saq-repository/saq_repository/models/finance/finance_models.py` (`AccountBank`, `AccountBankMirror`)
  - `shared/saq-shared-libraries/packages/saq-cache-client` (saldo no Valkey)
  - `apps-v3/lambdas/lambda-aws-pix-webhook-event/chalicelib/helpers/process_pix_cashin_pix_key.py`
  - `apps-v3/runners/approve-cashout-runner/helpers/{validate_limits,create_internal_transfer}.py`
