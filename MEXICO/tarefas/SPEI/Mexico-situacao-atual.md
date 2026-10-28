# México — o que existe hoje para o SPEI e o que falta

Levantado em 28/10/2026 a partir do código em `mexico-apps/`, dos manifests e do `saq-infra`. Caminhos relativos a `C:\wamp64\www\Noclaf\SAQ\`.

Resumo: só existe o onboarding. Não há nada de SPEI IN, SPEI OUT, cobrança, extrato, saldo ou webhook. O que está pronto para reaproveitar é a base do cliente Fintoc, as policies no seed e a infra da conta `qasmexhml`.

## 1. `mexico-fintoc-client`

Pasta `mexico-apps/mexico-shared-libraries/packages/mexico-fintoc-client/`.

**Cobre hoje** (Business Accounts API v2, só Account):

| Método | Rota |
| --- | --- |
| `create_account` | `POST /v2/accounts` (`entity_id`, `description` até 40 caracteres) |
| `get_account` | `GET /v2/accounts/{id}` |
| `list_accounts` | `GET /v2/accounts`, paginado por `starting_after` (header `Link rel=next`), limite 300 |
| `find_account_by_description` | varre as páginas; reconcilia depois que a chave de idempotência expira (24 h) |

**Base reaproveitável:**
- Auth `Authorization: <sk_test_/sk_live_>`; o modo vem da chave, não da URL. Header `Fintoc-Version: 2026-02-01` fixo (`api_client.py:21`).
- Config: `FINTOC_SECRET_KEY`, `FINTOC_ENTITY_ID`, `FINTOC_BASE_URL` (padrão `https://api.fintoc.com`), `FINTOC_API_VERSION` (`client.py:21-27`). `log_requests=False`.
- `Idempotency-Key` genérico em `_send` (`api_client.py:73-115`), até 255 caracteres. Só existe `account_idempotency_key()` = `mx:account:create:<uuid do pedido>`.
- Retentativa: 429 e falha de rede até 5×, backoff 1/2/4/8/16 s com jitter; só GET ou POST com `Idempotency-Key`; 5xx com resposta não repete.
- `FintocError(ServiceError)` com `code`, `error_code` (campo para regra de negócio), `error_type`, `param`, `is_rate_limited` (`errors.py`).
- Schemas `extra="ignore"`, `frozen`. `FintocAccountSchema`: `available_balance` em centavos, `root_account_number` (CLABE), `root_account_number_id`, `status`, `mode`. `currency` aceita `MXN` e `CLP`.
- Testes cobrem headers, erro, 429, 5xx, rede, paginação. Nenhum de transfer ou webhook.

**Falta para SPEI** (fontes: `entendimento/OperacaoMexico.md:76-113`, `entendimento/Pendencias.md:73-77`):

- SPEI OUT: `POST /v2/transfers` (`account_id`, `amount` centavos, `currency`, `counterparty.account_number` CLABE, `reference_id`, `comment`, `metadata`, `Idempotency-Key`); **assinatura JWS** no header `Fintoc-JWS-Signature` (`nonce`, `ts`), obrigatória em toda chamada que move dinheiro, com chave privada da QAS e pública no Dashboard (nada disso existe; o sandbox do onboarding não precisou, `tarefas/Onboarding/Sandbox-Onboarding.md:84`); `GET /v2/transfers/{id}` para resultado incerto (RN-025); account verification para validar CLABE; return a transfer para devolver SPEI IN.
- SPEI IN e cobrança: `/v2/account_numbers` (CLABEs extras com `metadata` e `options.min_amount`/`max_amount`, desativação com `PATCH status: disabled`); `POST /v2/simulate/receive_transfer` no sandbox.
- Extrato e conciliação: `/v2/movements`, `GET /v2/accounts/{id}/statements`.
- Webhooks da Fintoc: HMAC SHA-256 no header `Fintoc-Signature`; dedup por `transfer.id` / `tracking_key`; eventos fora de ordem (`transfer.inbound.succeeded`, `transfer.outbound.succeeded`, `transfer.outbound.rejected`, `transfer.outbound.failed`); estados `pending`, `succeeded`, `rejected`, `failed`, `return_pending`, `returned`, `reject_failed`; SPEI IN traz `data.account_id`, `account_number.metadata`, `tracking_key`, `reference_id`, `receipt_url` (CEP).
- Schemas novos: Transfer, AccountNumber, Movement, evento de webhook.
- Limites: 100 req/s por organização e **10 req/s para criar Transfer** (pedido de 50 não confirmado).

## 2. Modelo de dados (`mexico-repository`)

Uma migration: `migrations/versions/2f1febf5f622_create_onboarding_account_and_auth_.py`.

**Existe:** `mx_onboardingrequisition`; `mx_company` (`onboarding_requisition_id` UNIQUE, RFC, status `ACTIVE/BLOCKED/CLOSED`); `mx_account` (`company_id`, `currency=MXN`, status `PENDING/ACTIVE/BLOCKED/CLOSED`, `fintoc_account_id`, `fintoc_account_number_id`, `clabe`, todos UNIQUE, **sem coluna de saldo**); auth (`mx_user`, `mx_role`, `mx_policy`, `mx_rolepolicy`, `mx_userrole` por `company_id`/`account_id` e audience `mx-web`/`mx-api`, `mx_backoffice*`, `mx_otptotpdevice`, `mx_userloginattempt`, `mx_userpasswordreset`).

**Não existe:** transação/lançamento (ledger); SPEI OUT com máquina de estados; SPEI IN recebido; cobrança; CLABEs adicionais; inbox de webhooks da Fintoc com dedup; webhooks do cliente com log de entrega; tarifas e histórico; limites; saldo e saldo diário; divergências de conciliação; auditoria (RN-064).

**Regras já fixadas:** `NOMENCLATURA-BANCO.md` (PK `String(36)` UUID para pedidos, transferências e lançamentos; dinheiro `BigInteger` centavos em `amount` com `currency` ao lado; enum VARCHAR(50) `native_enum=False`; prefixo `mx_`). Call de alinhamento (`entendimento/alinhamento_call.txt`): tabela de transação particionada por mês, sem FK (só o id), tudo por fila seguindo o SaQ, monorepo `integration-mx` nos moldes do `integration-jdpi`.

**Policies** (`mexico-permissions/scripts/seed_mx_policies.sql:14-18`, `mx-web,mx-api`): `mexico:account:read`, `mexico:statement:read`, `mexico:charge:create` ("Gerar cobrança SPEI, válida por 72h"), `mexico:spei-out:create`, `mexico:webhook:manage`. Matriz PS06 em aberto; a ordem do `id` é o bit do `pm` no JWT, policy nova vai no fim. Não há policy de leitura de SPEI IN/OUT separada. Backoffice só tem onboarding (`read`, `review`, `create`) e `account:read`.

**Docs desatualizados:** `mexico-repository/readme.md` diz "Idempotency-Key = mx_account.id" (código usa o UUID do pedido); README do fintoc-client diz descrição `mx <mx_account.id>` (lambda usa `<nome> <RFC>`).

## 3. APIs e infra

**`mexico-api-v3`:** só `POST /v3/onboarding/` e `GET /v3/onboarding/{id}` (`X-Onboarding-Token`). **Sem auth de cliente**: não chama `configure_auth_permissions`, sem login web/API, senha nem API key. Config só com `DB_*`: nada de Fintoc, SQS ou Valkey.

**`mexico-backoffice-api`:** auth própria (JWT `aud mx-backoffice`, TOTP), rotas de onboarding; lockout no MySQL porque "o México ainda não tem Valkey".

**Lambda de referência** `mexico-apps/lambdas/lambda-aws-mexico-onboarding-approve-event`: Chalice `@app.on_sqs_message(batch_size=1)`, Pydantic na mensagem, `initialization()` com 3 secrets por ARN, etapas com commit (trava pedido → company/account PENDING → Fintoc fora da transação → grava ids/CLABE e ACTIVE → user web → COMPLETED). 429/rede → exceção e volta para a fila; outros erros da Fintoc, incluindo 5xx → `ERROR`. `MAX_RECEIVE_COUNT=3`. Reconciliação por descrição quando PENDING há mais de 23 h. Publica `ONBOARDING_COMPLETED` em `SQS_ONBOARDING_COMPLETED` (sem consumidor). Deploy `saq-ci` `chalice-deploy`, stage `qasmexhml`, timeout 120 s.

**Conta `qasmexhml` (707337761900):**
- EKS namespace `apps-mexico`: `mexico-api-v3` (ALB público), `mexico-backoffice-api` (ALB interno), `mexico-migrate-cron-job`. Overlay `dev-qasmexhml`, sem autoSync.
- Secrets: `credenciais-rds-qasmexhml-app-user`, `-backend-migrate`, `credenciais-auth-token`, `apps-config-mexico` (`ENVIRONMENT`, `SQS_MEXICO_ONBOARDING_APPROVE`, `API_BASE_URL_BACKOFFICE`), `credenciais-partner` (`FINTOC_SECRET_KEY`, `FINTOC_ENTITY_ID`). **`credenciais-partner` só é lido pela lambda; não está no `SecretProviderClass`** (`config-commons-mexico/base/secretproviderclass.yaml`).
- Filas: só `approve-onboarding` em uso (DLQ, maxReceive 3). O template do tenant (`saq-infra/app-eks/Pulumi.qasmexhml.yaml:188-529`) criou ~40 filas do Pix (`company-notification-*`, `approve-cashout`, `webhook-resend`, `update-cache`...) sem uso. LocalStack local cria `mexico-onboarding-approve`, DLQ e `mexico-onboarding-completed`.
- **Valkey** `qasmexhml-memorydb` (Valkey 7.3, TLS, `Pulumi.qasmexhml.yaml:58-78`) existe e nenhum app usa; endpoint e senha não estão em secret/env do México. Não está no docker-compose.
- Backoffice frontend: `ROUTES.md` e `POLICIES_CLASSIFICATION.md` são cópia do SaQ; as 12 telas do PRD para SPEI, webhook e tarifas não existem.

## 4. O que precisa ser criado

- Filas e lambdas: receptor do webhook Fintoc (HTTP público com HMAC) → fila `mexico-fintoc-webhook` com DLQ; lambda de SPEI IN (crédito, tarifa, notificação); fila e lambda/runner de SPEI OUT (reserva, `POST /v2/transfers` com JWS e idempotência); runner/cron de acompanhamento de transfers `pending`; fila e lambda de webhooks ao cliente com retentativa e log; cobrança de tarifa (Transfer para a Account da QAS, por operação ou varredura); cron de conciliação com Movements; cron de expiração de cobrança (72 h).
- Valkey: ligar o `qasmexhml-memorydb` (secret e env), prefixo `mx:`, scripts Lua no padrão do `saq-cache-client` (pacote `mexico-cache` proposto em `Reaproveitamento.md`); adicionar no docker-compose.
- Secrets: chave privada JWS no `credenciais-partner`; secret do webhook Fintoc; id da Account de tarifas da QAS; montar `credenciais-partner` nos pods que chamarem a Fintoc.
- Rede: URL pública do webhook no Dashboard da Fintoc; allowlist de IP se houver.
- Repositório: `integration-mx` ou lambdas em `mexico-apps/lambdas`.
- Auth do cliente no `mexico-api-v3` (pré-requisito de qualquer rota de SPEI).
