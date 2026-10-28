# Pix IN e Pix OUT no SaQ — referência para o SPEI

Levantado em 28/10/2026 a partir do código do workspace (`apps-v3/`, `shared/`). Serve de modelo para o SPEI IN/OUT do México. Caminhos relativos a `C:\wamp64\www\Noclaf\SAQ\`.

## 1. Princípios comuns

- **A API não move dinheiro.** Grava a intenção numa tabela e publica numa fila SQS. Runners e lambdas fazem o resto.
- **Saldo rápido no Valkey (MemoryDB)**, com scripts Lua atômicos no `saq-cache-client`: reserva, confirmação, estorno, crédito e limites diário, mensal e noturno. Toda operação leva uma chave de idempotência (id da transação ou e2e).
- **O extrato `finance_accountbanktransaction` é a fonte da verdade.** O saldo no MySQL (`AccountBank.amount` / `AccountBankMirror.amount`) é convergido depois pelo `runners/account-balance-update-runner`, que lê `finance_accountbanktransactionpending` (provavelmente alimentada por trigger; não confirmado no código).
- **Tarifa** é linha separada no extrato (`PIX_CASHIN_TAX`, `PIX_CASHOUT_TAX`, splits), calculada por `company_tariff_calculator.py` + `CompanyTariff.calculate_tax`, com splits em `tax_split_ledger.py`.
- **PSP abstraído** pela lib `shared/saq-pix-shared-libraries/packages/saq-proxy-client` (`PROXY_PROVIDER=JD|ONZCLOUDPIX`, adapters `jdpi/adapter.py` e `onzcloudpix/adapter.py`) e pelos contratos de `saq-proxy-schemas`.
- **Notificação ao cliente** por fila dedicada por tipo de evento (`saq_utilities/decorators/decorators.py`, `_COMPANY_QUEUE_ENV_BY_TYPE`), lambdas `lambda-aws-company-notification-*` disparam o webhook com header `hmac` assinado pela `company.hmac_key`, timeout 5 s, log em `log-notification`.
- **Auditoria**: `finance_logtransaction` em cada mudança de status (`add_log_transactions` → `SQS_LOG_NOTIFICATION`).
- As variáveis `SQS_*` chegam aos pods pelo secret `apps-configs.json` via CSI, não pelos manifests. Para lambdas ficam em `shared/saq-ci/lambdas/config/<lambda>.json`.
- Atenção: `apps-v3/pix-webhook-api-v3` **não recebe o PSP**. Só tem `/v3/webhooks/*` (logs e reenvio dos webhooks que o SaQ manda ao cliente, fila `webhook-resend`).

## 2. Pix OUT (cash-out)

### Componentes

| Componente | Papel |
| --- | --- |
| `apps-v3/cashout-api-v3` (`/v3/cashouts`, legado `/v2/finance/*`) | Autentica, valida conta, cria `Transaction`, idempotência, maker/checker, publica em `approve-cashout` ou `approve-cashout-manual`. |
| `apps-v3/runners/approve-cashout-runner` | Coração do fluxo: validações, DICT, tarifa, limites, débito no cache, envio ao PSP. `--manual` é a instância da fila manual. |
| `apps-v3/lambdas/lambda-aws-check-payment-order` | Confirma a ordem (cache do PACS.002 ou polling). Sucesso: extrato + tarifa + confirm no cache. Falha: ERROR + refund no cache. |
| `apps-v3/integration-onz/.../onzcloudpix-long-polling-pacs-002-runner` | "Webhook" da ONZ: grava o resultado em `cashout:payment-order:{id}` (TTL 15 min) e enfileira `check-payment-order`. |
| `runners/register-internal-transfer-runner` | Liquidação on-us (destino no mesmo ISPB). |
| `apps-v3/dict-api-v3` | Consulta DICT de pré-visualização para o portal; grava cache que o runner reaproveita (TTL 900 s). |
| `apps-v3/saq-backoffice-api/main/routes/cashout_approvals.py` | Alçada do backoffice para `AWAITING BANK CONFIRMATION`. |
| `lambda-aws-company-notification-cashout-created-event`, `-cashout-success-event`, `lambda-aws-company-notification-event` | Webhooks `PIX_CASHOUT_CREATED`, `SUCCESS`, `ERROR`, `CANCELED`. |
| `lambda-aws-account-balance-block-event` | Bloqueio judicial/administrativo no cache (`{account}:balance:blocked`). |
| `lambda-aws-update-cache-event` | Recarrega cache de user/company/parâmetros (é de onde a API lê `account.block_cashout`). |
| `lambda-aws-cashout-monitor` | Teste sintético a cada 10 min, não é reconciliador. |

Filas (`saq-infra/app-eks/Pulumi.prod.yaml:948-1095`): `approve-cashout` e `approve-cashout-manual` (visibility 60 s, maxReceive 15, DLQ), `check-payment-order` (60 s, 15, DLQ), `register-internal-transfer`, `company-notification*` (maxReceive 3), `log-notification`.

### Fluxo

1. **Requisição.** `POST /v3/cashouts/` (`cashout-api-v3/main/routes/v3/create_cashout.py`), body discriminado por `type` KEY / MANUAL / COPY_AND_PASTE. Variantes: `/self-approvals` (exige create + approve) e `/fractional/key|manual` (quebra em lotes).
2. **Validações síncronas.** Escopo `Cashout.CREATE`; audience API exige header `hmac`, audience WEB exige `mfa`; conta mirror com policy do usuário; integração bancária compatível com `PROXY_PROVIDER`; `account.block_cashout`. **Não valida saldo.**
3. **Persistência.** Id sequencial do Valkey (`get_transaction_id`); `helpers/create_transaction.py` grava `Transaction` em `NEW` (ou `PROCESSING` se self-approve), `idempotency_key` = header ou uuid7, `request_data_json`, `type_transaction` INTEGRATION/INTERNAL, `transfer_type=PIX`. Se não é self-approve, insere `CashoutPendingUserApproval`. `IntegrityError` na chave única: mesmo body → replay da resposta; body diferente → 409. `@event_collector` publica `log_transaction` e `PIX_CASHOUT_CREATED`.
4. **Maker/checker.** `GET /v3/cashouts/pending` e `POST /v3/cashouts/approvals` (lote). Regras: policy APPROVE na conta, status `NEW`, `AccountBankRule.block_cashout`, aprovador ≠ criador. Aprovou: `PROCESSING`, apaga a pendência, `send_approve_cashout` publica `{transaction_id, user_id}`.
5. **Runner** (`helpers/process_cashout.py`). Até 5 mensagens, long-poll 10 s. `get_transaction` faz `SELECT ... FOR UPDATE` e só aceita `PROCESSING` (idempotência contra reentrega). Rate limit do DICT: reenfileira com delay 30–120 s até 5 vezes.
6. **`validate_cashout`** (`helpers/validate_cashout.py`), na ordem: conta/mirror não CLOSED e `registration_update_deadline`; resolução do destino (QR, chave interna, manual, `internal_transfer`); `block_external_cashout`; `validate_limits(simulate=True)` (tarifa obrigatória, limite por operação `AccountTransactionLimit`, simulação da reserva); chave externa → gera e2e (`generate_end_to_end_id(ispb)`) e consulta DICT (`consult_pix_key`, reaproveita cache); documento, `BankRegisterISPB.is_active_cashout`, `ClientBlackList`; `cashout_pj_only_trusted_recipients`; `cashout_web_only_trusted_recipients`; alçada por destinatário (`check_approval_rules.py`, contador Valkey por documento/dia) → `TransactionPendingApproval` + `AWAITING BANK CONFIRMATION`. Falha vira `ERROR` ou `CANCELLED` com `erro_descriptor`, e notifica.
7. **Reserva** (`helpers/register_payment_order.py` → `validate_limits(simulate=False)` → Lua `debit_account_with_limit.py`): checa `saldo - bloqueado - (valor + tarifa) >= 0` e os limites mensal/diário/noturno (fuso America/Sao_Paulo, noite 22h–6h), faz `DECRBY` e grava a reserva `{account_mirror:ID}:transaction:debit:{txid}`. Idempotente pela chave de reserva. Nada no banco de saldo.
8. **Envio ao PSP.** Interno: `create_internal_transfer.py` liquida na hora (SUCCESS, par débito/crédito no extrato, confirm no cache, `PIX_CASHOUT_SUCCESS` + `PIX_CASHIN_RECEIVED`, depois `register-internal-transfer`). Externo: `saq_proxy_client.register_payment_order` (JDPI `POST /spi-api/jdpi/spi/api/v2/op` com `Chave-Idempotencia` = `idempotency_key`; ONZ `POST /v3/icom/pacs/008`), grava `uuid` e `transactionId`. Falha → ERROR e estorno da reserva. Sucesso → publica em `check-payment-order` `{"id", "type":"PIX_CASHOUT"}` com delay do provider (JDPI imediato; ONZ 90 s, máx. 900).
9. **Confirmação** (`lambda-aws-check-payment-order/chalicelib/helpers/check_payment_order_cashout.py`): `FOR UPDATE`; `_resolve_payment_order` (cache do PACS.002 ou polling 3× com sleep 2–4 s; JDPI `GET /op/{transactionId}`). Ainda `PROCESSING` → exceção → SQS reentrega a cada 60 s até 15× → DLQ (transação fica `PROCESSING` com saldo reservado; não há reconciliador automático). SUCCESS → `AccountBankTransaction` PIX_CASHOUT, tarifa = `reserva - amount` em linhas próprias, status SUCCESS, `operationUuid` = e2e, `transaction_confirm_debit_account` apaga a reserva. ERROR → status ERROR + Lua `refund_debit_account_with_limit` + decremento do contador do destinatário. Reentrega com status já final repete só confirm/refund.
10. **Alçada do backoffice** (`cashout_approvals.py`): APPROVED → transação volta a `PROCESSING` e é reenviada para a fila (o runner pula a alçada). REJECTED → `CANCELED`, `PIX_CASHOUT_CANCELED`, audit log. Sem estorno porque a reserva só acontece depois da alçada.
11. **Notificação**: webhooks por tipo; consulta por polling `GET /v2/finance/status-cashout`. Não há e-mail ao cliente no fluxo.

### Modelo de dados (`shared/saq-repository/saq_repository/models/finance/finance_models.py`)

- `Transaction` → `finance_transaction`: `amount` (sem coluna de tarifa), `status` (`NEW`, `APPROVED`, `CANCELED`/`CANCELLED`, `WAITING REPROCESSING`, `PROCESSING`, `SUCCESS`, `AWAITING BANK CONFIRMATION`, `ERROR`, `PENDENT`, `REFUNDED`), `type_transaction`, `transfer_type`, `channel`, `company_id`, `account_id`, `account_mirror_id`, destino (`key`, `key_type`, `recipient_*`, `recipient_instution`, `to_account_type`, `internal_transfer`), ids (`key_reference_id` e2e gerado, `uuid`, `transactionId` do PSP, `operationUuid` e2e confirmado), `idempotency_key` (36, unique), `erro_descriptor`, `approved_date`, `user_created_id`, `request_data_json`, `metadata`, `tag`, `sent_webhook`.
- `CashoutPendingUserApproval` (maker/checker), `TransactionPendingApproval` (alçada, `PENDING/APPROVED/REJECTED`, `rule_description`, `rejection_reason`).
- `AccountBankTransaction` → extrato: `type_transaction` (catálogo), `transaction_amount`, `transaction_obj_id`, `transaction_ref_obj_id` (tarifa → principal), `uuid`, contraparte, `internal_transfer`, `channel`.
- `AccountTransactionLimit` (por transação, diário, mensal, noturno), `CashoutFractionalBatchLimit`, `AccountBankRule`, `TrustedRecipient`, `ClientBlackList`, `BankRegisterISPB`, `DictConsultBlock`, `AccountBalanceBlock`.
- Valkey, prefixo `{account_mirror:ID}`: `:balance`, `:balance:blocked`, `:limit:daily:YYYYMMDD`, `:limit:monthly:YYYYMM`, `:limit:night:YYYYMMDD`, `:transaction:debit:{txid}`. Globais: `cashout:payment-order:{txid}`, `cashout:retention:{txid}`.

## 3. Pix IN (cash-in)

### Componentes

| Componente | Papel |
| --- | --- |
| PSP JD (JDPI) | Chama `POST .../validar`, `/validar/async`, `POST /credito`, `POST /credito/devolucao` do SaQ (audience `JDPI`, escopo `Jdpi.SIST_GCT`). |
| PSP ONZ CloudPix | PACS.008 (crédito) e PACS.002 (confirmação) por long-polling em runners. |
| `integration-jdpi/apps/saq-validate-account-api-v3` | Validação síncrona (`routes/validate_credit.py`), lote em `validate_credit_async.py`. |
| `integration-jdpi/apps/lambda-aws-pix-batch-validate-account` e `lambda-aws-pix-validate-account` | Lote → mensagens individuais → `validate_pending_cashin` + `saq_proxy_client.validate_account`. |
| `integration-jdpi/apps/pix-register-credit-api-v3` | `routes/credito.py`: registra a liquidação e publica no SNS `pix-webhook` (tenacity 3× a cada 10 s). |
| `integration-onz/.../pacs-008-runner` e `pacs-002-runner` | Equivalentes ONZ: validação (responde ACSP/RJCT) e registro do crédito. |
| `runners/check-register-credit-runner` | Rede de segurança: 120 s (síncrono) ou 600 s (lote) depois, se o crédito validado não foi registrado e o SPI diz SUCCESS, reprocessa. |
| SNS `pix-webhook` → SQS `pix-webhook` e `log-notification` | Definição em `saq-infra/app-eks/Pulumi.<stage>.yaml:597-700`. |
| `lambdas/lambda-aws-pix-webhook-event` | Núcleo do crédito (batch_size 1). |
| `apps-v3/pix-copy-and-paste-api-v3` | Cria a cobrança/QR dinâmico (`routes/v2/create_pix_copy_and_paste.py`), status, listagem, chargeback de QR. |
| `apps-v3/pix-qrcode-decode-api-v3` | Serve o payload JWS ao PSP pagador: `GET /v2/qr/cob/{id}`, `/cobv`, `/rec` (404 se não existe, 410 se não está NEW ou expirou). |
| `apps-v3/cashin-api-v3` | `POST /v2/finance/chargeback` (devolução), `GET /v2/finance/cashin/{e2e}`, status do chargeback. |
| `lambda-aws-company-notification-qr-code-created-event` / `-paid-event` / `lambda-aws-company-notification-event` | Webhooks `QR_CODE_COPY_AND_PASTE_CREATED`, `_PAID`, `PIX_CASHIN_RECEIVED`, refunds. |
| Libs | `saq-cashin-validation` (regras e persistência de `PixCreditTransaction`), `saq-jdpi-client`, `saq-onzcloudpix-client`, `saq-proxy-schemas`, `saq-cache-client`. |

### Fluxo: cobrança (QR dinâmico)

1. Audience API, escopo `QrCode.CREATE`, header `hmac`.
2. Conta mirror por agência e conta, com policy; `CLOSING` é rejeitada.
3. `qr_code_random_amount_enabled` soma um valor aleatório.
4. Limite: `account_mirror.max_amount_for_qr_code` ou global `pix_qrcode_max_amount`.
5. `due_date` = +7 dias; `expiration_date` = 10 min sem multa, 14 dias com multa.
6. Geração: `generate_immediate_dynamic_qr_code` (cob) ou `generate_charge_dynamic_qr_code` (cobv). JD gera remoto; ONZ gera local com assinatura KMS.
7. Insere `PixQrCode` em `NEW` com `conciliation_id` (txid), `uuid`, `payload_jws`, `pix_copy_and_paste`, mirror e bolsão. Id sequencial do Valkey.
8. Notifica `QR_CODE_COPY_AND_PASTE_CREATED`.
9. O PSP pagador busca o JWS no `pix-qrcode-decode-api-v3`.

### Fluxo: recebimento

1. **Validação síncrona** (SLA BACEN). `validate_credit` → `helpers/validate_cashin.py` → `saq_cashin_validation.validate_cashin` (`services/cashin.py:_evaluate_cashin`). Grava `PixCreditTransaction` (VALID/INVALID, `payload_json`, `account_bank_id`, `pix_qr_code_id`). Aprovado → agenda `check-register-credit` com 120 s. Lote: `PixCreditBatch` com `Chave-Idempotencia`.
2. **Registro.** `credito.py:credito` exige VALID; se já tem `credit_identifier`, devolve o mesmo resultado; senão grava `request_identifier`, `credit_identifier`, `credit_at` e publica `{"type":"partner","partner":"jd","end_to_end_id","liquidation_at","partner_data"}` no SNS.
3. **Lambda.** `process_partner_webhook.py` relê o payload validado do banco (exige VALID), faz parse (`parse_cashin_liquidation_request` ou `parse_refund_liquidation_request` se `is_restitution`).
4. **Despacho** (`process_pix_cashin.py`): `PIX_AUTOMATIC` → manual com `pix_automatic=True`; `receiver_conciliation_id` → `process_pix_cashin_qrcode`; `key` → `process_pix_cashin_pix_key`; senão `process_pix_cashin_manual` (resolve conta por documento, agência, conta).
5. **Crédito via QR** (`process_pix_cashin_qrcode.py`): `FOR UPDATE` no `PixQrCode`, status `PAID`, payer, ISPB, `loc_id` = e2e; `AccountBankTransaction` PIX_CASHIN com valor **bruto** + linhas de tarifa; `cashin_credit_account(mirror, e2e, bruto - tarifa)` no Valkey; após commit `cashin_confirm_credit_account` e `confirm_temporary_payment_qr_code`; notifica `QR_CODE_COPY_AND_PASTE_PAID`; `publish_cashin_document_routes` (filas de produto, ex. `anticipation-payments-received`).
6. **Chave e manual**: análogo, idempotência por `AccountBankTransaction(uuid=e2e, type=PIX_CASHIN)`; notifica `PIX_CASHIN_RECEIVED`.
7. **Webhook ao cliente**: `WebhooksCompany` por tipo/conta/mirror ou webhook global do whitelabel, header `hmac`, POST com timeout 5 s, log em `SQS_LOG_NOTIFICATION`.

### Regras de validação (`saq-cashin-validation/services/cashin.py`), na ordem

1. Conta não encontrada (documento + agência + conta + integração) → **AC03**.
2. Conta não ACTIVE → **AC07**.
3. Recebedor na blacklist (`block_receiver_cashin`) → **DS04**.
4. ISPB pagador com `is_active_cashin=False` → DS04.
5. Bloqueios: `block_cashin`, `block_external_cashin`, por modalidade manual/chave/QR → DS04.
6. `cashin_only_same_document` → DS04.
7. Pagador na blacklist → DS04.
8. Chave que não pertence à conta → AC03.
9. Escolha da mirror (pela chave; manual pela agência/conta ou `default_company_cashin`) e as mesmas checagens nela.
10. Limite por transação → AM02 (BE17 se QR).
11. QR dinâmico → **BE17** se: não NEW, conta diferente, expirado, devedor ≠ pagador (flag), pagador PJ (flag), valor diferente, já tem crédito VALID, trava Valkey `pixqrcode:paid:` com outro e2e.
12. Pix Automático: exige `PixRecurrenceCharge` ACCEPTED.
13. Não-prod: ISPB de teste do BACEN aprovado direto.

### Idempotência e devolução

- `end_to_end_id` UNIQUE em `PixCreditTransaction`; `IntegrityError` com payload igual → mesmo resultado; diferente → DS04 "Payload divergente". `/credito` idempotente por `credit_identifier`; lote por `PixCreditBatch.identifier`.
- Valkey: `{account_mirror:<id>}:cashin:credit:<e2e>` evita crédito duplo se o banco der rollback depois do INCRBY. Trava `pixqrcode:paid:<conciliation_id>` SET NX TTL 30 min.
- **Chargeback** (`cashin-api-v3`): `hmac` ou `mfa`; até 90 dias; contas não CLOSED; sem chargeback PENDING; soma ≤ original; saldo. Externo: débito no cache → linha PIX_CHARGEBACK → `register_payment_order_refund` (MD06, e2e próprio) → PENDING e `check-payment-order` com `type PIX_CHARGEBACK`. Interno: `Transaction` REFUNDED + PIX_CASHOUT_REFUND/PIX_CHARGEBACK + `register-internal-transfer`. Devolução recebida passa pelo mesmo pipeline com `is_restitution` (`services/refund.py`, `process_pix_refunded.py`).
- Retries: `pix-webhook` maxReceive 15 (DLQ), `company-notification*` 3, `check-register-credit` 3, `pix-validate-account` 3, `check-payment-order` 15.

### Modelo de dados

- `PixCreditTransaction` → `finance_pixcredittransaction`: `end_to_end_id` UNIQUE, `validation_status` PENDING/VALID/INVALID, `validation_error_code/_message`, `payload_json`, `is_restitution`, `batch_id`, `account_bank_id`, `pix_qr_code_id`, `request_identifier`, `credit_identifier`, `credit_at`.
- `PixQrCode` → `finance_pixqrcode`: `status` NEW/PROCESSING/PAID/EXPIRED/CANCELED/CHARGEBACK, `value`, `original_value`, `chargeback_value`, `conciliation_id`, `uuid`, `loc_id`, `expiration_date`, `due_date`, `fine_*`, `debtor_*`, `payload_jws`, `pix_copy_and_paste`, mirror e bolsão, `tag`, `payment_date`.
- `AccountBankTransaction`: tipos PIX_CASHIN, PIX_CASHIN_TAX, PIX_CASHIN_TAX_INTERMEDIATE, PIX_CASHIN_TAX_RECEIVE, PIX_CHARGEBACK, PIX_CASHOUT_REFUND, PIX_AUTOMATIC_CASHIN; `transaction_amount` bruto, `uuid` = e2e, `tx_id`, `pix_qr_code_id`, `transaction_ref_obj_id`, dados do pagador.
- `ChargebackHistoryPix`, `ChargebackHistoryPixQrCode` (`idempotency_key` e `uuid` UNIQUE), `CashinDocumentRoute`.

### Achados no código (a confirmar antes de copiar)

1. QR que não está NEW nem PAID na liquidação só gera warning e retorna: dinheiro liquidado sem crédito (`process_pix_cashin_qrcode.py:98-100`).
2. Chave desativada entre validação e liquidação retorna em silêncio (`process_pix_cashin_pix_key.py:371-374`).
3. `process_pix_cashin_qrcode.py:114` testa `debtor_document` antes de copiar `debtor_name`.
4. A mirror no manual é escolhida duas vezes com lógicas diferentes (validação vs liquidação).
5. `lambda-aws-cashout-monitor/.chalice/config.json` tem segredos de produção versionados em texto puro. Rotacionar e mover para Secrets Manager.
