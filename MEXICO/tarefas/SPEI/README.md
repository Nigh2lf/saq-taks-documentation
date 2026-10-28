# Tarefa: SPEI IN / SPEI OUT

Status em 28/10/2026: **não iniciada**. Esta pasta reúne o levantamento feito para começar: como o Pix funciona no SaQ (modelo a copiar), o que o México já tem, o mapeamento de um para o outro, a ordem de construção proposta e as decisões em aberto.

| Arquivo | Conteúdo |
| --- | --- |
| `Pix-SaQ-referencia.md` | Pix IN e Pix OUT no SaQ como estão no código: componentes, fluxo passo a passo, modelo de dados, regras, idempotência, retries. |
| `Mexico-situacao-atual.md` | O que existe no México (fintoc-client, modelo de dados, infra da conta `qasmexhml`) e o que falta. |
| `../../entendimento/OperacaoMexico.md` | Endpoints, estados e eventos da Fintoc; fluxos propostos. |
| `../../entendimento/Reaproveitamento.md` | De onde copiar cada padrão do SaQ. |
| `../../entendimento/PRD-MEXICO-V0.pdf` | Regras RN-011 a RN-014 (SPEI IN), RN-017 a RN-026 (SPEI OUT), RN-027 a RN-032 (tarifas), RN-033 a RN-044 (extrato), RN-050 a RN-057 (webhooks). |

## 1. Escopo (PRD V0)

- **SPEI IN**: receber na CLABE da conta; referência por recebimento; entra no extrato; sem duplicidade; taxa pela configuração vigente. Tela 1.1: dados para receber e "geração de uma cobrança que pode ser compartilhada com um terceiro" (válida por 72 h segundo `V0.md` e o seed de policies).
- **SPEI OUT**: valor > 0 e dentro dos limites; saldo para valor + taxa; mostrar valor, taxa e total antes de confirmar; reservar saldo; estados processando / concluída / não efetivada; só concluir com confirmação do parceiro; estorno em falha; resultado incerto fica em acompanhamento; sem duplicidade. Tela 1.2: "padrão do Pix por agência e conta".
- **Tarifa**: vai para uma Account de receita da QAS via Transfer, por operação ou por varredura (a decidir).
- **Webhooks ao cliente**, **extrato** e **API pública** completam a V0.

## 2. Mapeamento Pix (SaQ) → SPEI (Fintoc)

| Pix no SaQ | SPEI no México |
| --- | --- |
| PSP JDPI/ONZ atrás do `saq-proxy-client` | Fintoc atrás do `mexico-fintoc-client` (mesma ideia de fachada; um adapter só) |
| Bolsão + mirror por cliente | Uma `Account` da Fintoc por cliente debaixo da Entity da QAS (padrão B); sem mirror |
| Chave Pix, agência e conta, DICT | CLABE; sem DICT; "account verification" da Fintoc para validar a CLABE antes de enviar |
| e2e (`end_to_end_id`) | `tracking_key` (clave de rastreo) e `transfer.id` |
| Pix IN: validação síncrona pelo PSP ("posso creditar?") + registro + SNS/SQS + lambda | **Não há validação síncrona.** O dinheiro cai e chega o webhook `transfer.inbound.succeeded`. O SPEI IN começa no passo da lambda: receptor valida HMAC `Fintoc-Signature`, deduplica por `transfer.id`/`tracking_key`, enfileira; lambda credita |
| Cobrança = QR dinâmico (cob/cobv) com `PixQrCode` | Fintoc não tem QR. Candidato: CLABE dedicada por cobrança via `/v2/account_numbers` com `min_amount`/`max_amount` e `metadata`, desativada após 72 h. **Inferência, não decidido** |
| Pix OUT: `Transaction` → fila → runner (reserva Lua + `register_payment_order`) → `check-payment-order` → lambda confirma/estorna | Mesmo desenho: `mx_transaction` → fila → runner (reserva no Valkey `mx:` + `POST /v2/transfers` com `Idempotency-Key` e **JWS**) → webhook `transfer.outbound.*` ou `GET /v2/transfers/{id}` → confirma/estorna |
| `Chave-Idempotencia` no JDPI | `Idempotency-Key` na Fintoc (24 h) |
| Assinatura KMS no ONZ | JWS `Fintoc-JWS-Signature` (`nonce`, `ts`), chave privada da QAS, pública no Dashboard |
| Limites diário/mensal/noturno, fuso America/Sao_Paulo | Parametrizar fuso (America/Mexico_City) e janela do SPEI |
| Tarifa como linha no extrato + split para a conta da SaQ | Linha no extrato + Transfer para a Account de receita da QAS (consome o limite de 10 req/s se for por operação) |
| Webhooks ao cliente por fila/tipo com HMAC | Igual: copiar `lambda-aws-company-notification-event` |
| Extrato `finance_accountbanktransaction` + `account-balance-update-runner` | `mx_transaction` particionada por mês, sem FK; conciliação com `/v2/movements` |
| Rate limit do DICT | 100 req/s por organização, 10 req/s em Transfer |

## 3. Ordem de construção proposta

1. **Auth do cliente** no `mexico-api-v3`: login web, definição de senha, API key `mx-api`, `configure_auth_permissions`. Pré-requisito de qualquer rota de SPEI.
2. **Ledger**: `mx_transaction` (particionada por mês, sem FK, UUID, `amount` centavos + `currency`) e o Valkey `qasmexhml-memorydb` ligado com prefixo `mx:` e scripts Lua copiados do `saq-cache-client` (pacote `mexico-cache`). Decidir a fonte de verdade do saldo (item 4 abaixo).
3. **Receptor do webhook da Fintoc**: HTTP público, HMAC, inbox com dedup, publica na fila `mexico-fintoc-webhook` (DLQ). Onde fica: `mexico-api-v3` ou `integration-mx`.
4. **SPEI IN**: lambda que consome a fila, grava o lançamento bruto + tarifa, credita o líquido no Valkey com chave por `tracking_key`, notifica o cliente. Depois a **cobrança 72 h** (account numbers + cron de expiração).
5. **SPEI OUT**: rota grava `mx_transaction`, fila, runner (validações, reserva, `POST /v2/transfers` com JWS), fila de check, lambda de confirmação ou estorno; runner de acompanhamento de `pending`.
6. **Webhooks ao cliente** (cadastro + fila + lambda com retentativa e log) e **extrato** (copiar `account-statement-api-v3`).
7. **Tarifa** (Transfer para a Account da QAS) e **conciliação** (Movements).
8. **Backoffice**: telas de SPEI IN, SPEI OUT, webhooks, tarifas e extrato; policies novas no fim do seed.

## 4. Decisões em aberto

1. **Escopo do portal**: reunião de 24/09 diz "portal só extrato"; PRD de 30/09, seed e `V0.md` incluem SPEI OUT e cobrança.
2. **Cobrança 72 h**: o que é (CLABE dedicada vs CLABE raiz + referência), valor fixo ou não, o que fazer com pagamento após expirar (devolver ou creditar).
3. **JWS**: gerar o par de chaves, cadastrar a pública no Dashboard, guardar a privada no `credenciais-partner`, implementar a assinatura.
4. **Fonte de verdade do saldo**: Valkey + `mx_transaction` (padrão SaQ) ou `available_balance` da própria Fintoc. A Fintoc também guarda o saldo no padrão B.
5. **Tarifa**: por operação (um Transfer a cada SPEI, consome rate limit e pode ter custo) ou varredura diária. Confirmar se a Fintoc cobra Transfer entre Accounts. Criar a Account de receita da QAS. Modelos e valores (PS05) indefinidos.
6. **Limites e estados (PS02, PS03)**: por operação e por dia; conta `BLOCKED` (desativar CLABE com `PATCH account_numbers`); devolução de SPEI IN.
7. **Catálogo de webhooks para o cliente (PS07)**: eventos, payload, retentativa, DLQ.
8. **Onde fica o receptor do webhook Fintoc** e qual URL/allowlist cadastrar.
9. **Rate limit**: aumento para 50 req/s não confirmado.
10. **Comprovante SPEI** sai no nome da QAS no padrão B; confirmar com produto.
11. **Policies**: matriz PS06; faltam leitura de SPEI IN/OUT e as de backoffice.
12. **Produção**: não existe conta AWS de produção do México.

## 5. Pré-requisitos de infra (conta `qasmexhml`)

- Montar `credenciais-partner` no `SecretProviderClass` dos pods que chamarem a Fintoc.
- Secret do webhook Fintoc (HMAC) e chave privada JWS.
- Endpoint e senha do Valkey em secret/env; Valkey no docker-compose local.
- Filas novas (`mexico-fintoc-webhook`, SPEI OUT, check, notificação) ou reaproveitar as ~40 do template do tenant que estão ociosas.
- URL pública do webhook cadastrada no Dashboard da Fintoc.
