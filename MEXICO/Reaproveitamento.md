# Operação México — o que aproveitar da SaQ e o que criar do zero

Levantado em 19/10/2026 a partir do código do workspace. Premissa: **banco de dados separado** para o México. Complementa `OperacaoMexico.md` e `Pendencias.md`.

## 1. Resumo

- **A SaQ já tem o molde exato.** Crypto (`crypto-apps/`) e Antecipação (`anticipation-apps/`) são produtos com banco próprio, repository próprio e libs próprias. Eles usam o core da SaQ só em modo leitura, para login, permissões e empresa. O México deve seguir o mesmo desenho, numa pasta `mexico-apps/`.
- **A plataforma é reaproveitada como está:** libs base, login/JWT, permissões, CI/deploy.
- **Os padrões são copiados e adaptados:** onboarding, ledger no Valkey, tarifas, webhooks para o cliente, extrato, backoffice.
- **Tudo que fala com o SPEI é novo:** cliente Fintoc, receptor de webhook da Fintoc, máquina de estados do envio, conciliação, CLABE, KYC mexicano e telas.
- **Diferença em relação aos precedentes.** Crypto e Antecipação **não têm ledger próprio**: movimentam dinheiro na conta SaQ do core, em `finance_accountbanktransaction` e no saldo do Valkey. Com banco separado e MXN, o México precisa de **ledger próprio**. Nenhum produto hoje faz isso.

## 2. Estrutura proposta

```
mexico-apps/
  mexico-repository/          novo — copiar o esqueleto do anticipation-repository
  mexico-shared-libraries/    novo — pacote mexico-fintoc-client (molde: crypto-b2b-client)
  mexico-api-v3/              portal do cliente + API pública + receptor de webhook Fintoc (molde: crypto-api-v3)
  mexico-backoffice-api/      molde: anticipation-backoffice-api
  lambdas/                    SPEI OUT, SPEI IN, webhooks para o cliente, conciliação
  cron-jobs/                  conciliação diária, varredura de tarifa (se for essa a opção)
```

### Arquivos de referência

| Para quê | Copiar de |
|---|---|
| Repository (alembic, base model, publicação) | `anticipation-apps/anticipation-repository/`: `migrations/env.py`, `monorepo_setup.py`, `setup.py`, `VERSION`, `Dockerfile` |
| Nomenclatura de tabelas | `anticipation-apps/anticipation-repository/NOMENCLATURA-BANCO.md`. A cópia do crypto é mais antiga |
| Config com dois bancos (`CORE_READ` + `MEXICO_WRITE`) | `crypto-apps/crypto-api-v3/main/config.py`, `main/db.py`, `main/app.py` |
| Cliente de provedor externo | `crypto-apps/crypto-shared-libraries/packages/crypto-b2b-client/` (`BaseAPIClient` + schemas + mappers) |
| Receptor de webhook do provedor (HMAC + dedup por `event_id`) | `crypto-apps/crypto-api-v3/main/routes/v3/request_crypto_b2b_webhook.py` |
| Envio com estados, débito e estorno, SQS + DLQ | `crypto-apps/crypto-api-v3/main/routes/v3/request_crypto_confirmation.py` + `crypto-apps/lambdas/lambda-aws-crypto-order-processing/` (o README explica o fluxo) |
| Backoffice (login, policies, auditoria, i18n) | `anticipation-apps/anticipation-backoffice-api/main/`: `app.py`, `utils/scopes.py`, `middleware/`, `scripts/db/seed_core_anticipation_backofficepolicy.sql` |
| Tela de produto separado no backoffice | `apps-frontend/backoffice-whitelabel-frontend/src/app/(private)/anticipation/` + `src/service/api-anticipation.ts` |

## 3. Por item do escopo

Legenda: **A** = reaproveitar como está · **B** = copiar o padrão e adaptar · **C** = criar do zero

### Lado cliente

| Item | Veredito | O que aproveita | O que é novo |
|---|---|---|---|
| Abertura de conta e onboarding (cria a conta e, ao validar, a `Account` Fintoc) | **B** / C | O fluxo do Brasil: requisição → resultado do KYC → aprovação → lambda que cria conta, tarifas, limites e usuários (`apps-v3/lambdas/lambda-aws-onboarding-approve-event/.../process_onboarding_approve.py`). Os usuários são criados no core (`user_helper.py`) | `POST /v2/accounts` + gravar `fintoc_account_id` e a CLABE. Dados e validação mexicanos (RFC/CURP, endereço). O provedor de KYC no México não está definido; o CAF é só Brasil |
| SPEI IN | **B** | O desenho do `process_pix_cashin_pix_key.py`: dedup, cálculo de tarifa, crédito no Valkey via Lua com chave de idempotência, notificação ao cliente | Receptor do webhook Fintoc com validação de `Fintoc-Signature`, dedup por `transfer.id`, busca da conta por `data.account_id` |
| SPEI OUT | **B** / C | A orquestração do crypto (PENDING → débito → chama provedor → webhook → confirma ou estorna) e a lógica de limites do `approve-cashout-runner/helpers/validate_limits.py` | Cliente Fintoc (chave, JWS, `Idempotency-Key`, centavos, 10 req/s) e a máquina de estados `pending`/`succeeded`/`rejected`/`failed`, tolerante a eventos fora de ordem. O `saq_proxy_client` do Brasil é preso a SPI/ISPB/DICT e não serve |
| Extrato | **B** | O padrão de paginação por cursor do `account-statement-api-v3/main/routes/v3/statement.py` e os schemas de `saq-base-api` | As rotas leem tabelas do core, então serão reescritas sobre o ledger do México. Exportações (OFX, PDF, CSV) estão em pt-BR/BRL |

### Lado backoffice

| Item | Veredito | O que aproveita | O que é novo |
|---|---|---|---|
| Gestão do onboarding e das contas | **B** | O formato das rotas de `saq-backoffice-api` (`onboarding.py`, `list_accounts_combined.py`, `close_account.py`) e a estrutura de API separada do anticipation-backoffice | As consultas, sobre o banco do México. Bloquear a conta também faz `PATCH` na `Account` Fintoc e desativa a CLABE |
| Configuração de taxas por conta | **B** | A lógica de `CompanyTariff` (VALUE / PERCENT_WITH_MIN / PERCENT_WITH_MIN_AND_MAX, vigência), `company_tariff_calculator.py` e `tax_split_ledger.py`. A taxa é gravada em transação separada, o que já atende a RN-029 | O modelo de tarifa vive no core e tem campos BRL (`usd_to_brl_exchange_rate`, `today_brazil`, PF/PJ pelo tamanho do documento). Recriar no banco do México, com moeda e centavos. Cobrança por `Transfer` real na Fintoc |
| Administração de usuários | **A** | Usuários, roles, `BackofficePolicy`, whitelabel e as rotas de CRUD do `saq-backoffice-api` (`backoffice_users.py`, `users_web.py`, `user_roles.py`). Login em `auth-token-api-v3`, MFA e senha em `user-api-v3`. Os dois perfis do PRD viram policies novas | Classe `_Mexico` em `saq-permissions` + linhas de policy no core. Um whitelabel México para isolar o tenant. Ver o bloqueio da seção 4 |
| Criação de webhooks | **B** | O motor de entrega: `apps-v3/lambdas/lambda-aws-company-notification-event` (HMAC, plano de envio, log), além do reenvio (`pix-webhook-api-v3` + `webhook-resend-runner`). O motor não depende do conteúdo do payload | Tabelas de cadastro e tipos de evento próprios (`SPEI_IN_RECEIVED`, `SPEI_OUT_*`). Os enums atuais são de Pix e `WebhooksCompany` aponta para a mirror do core. Faltam DLQ e retentativa melhor: hoje são 3 recebimentos e descarte (RN-055) |
| Extratos consolidados e detalhados | **B** | Os padrões de `saq-backoffice-api` (`statement.py`, `daily_balances_sum.py`), o export assíncrono (`report-request-runner`) e o saldo diário (`calculate-account-balance-cron-job`) | As consultas sobre o ledger do México. Conciliação dos `Movement`s da Fintoc contra o ledger; o modelo mais próximo é `PiAccountStatementEntry` |

## 4. Plataforma (libs e infra)

| Peça | Veredito | Observação |
|---|---|---|
| `saq-base-api` | **A** | Health, CORS, envelope de erro, paginação, métricas. Só traz `en` e `pt-BR`; para `es-MX`, adicionar o locale (mudança pequena na lib) |
| `saq-base-repository` | **A** | Sessões async com vários bancos nomeados, `SessionManager`, `BaseRepository`, lock nomeado do MySQL |
| `saq-utilities` (AWS, HTTP, auth/JWT, exceções, tracing) | **A** | `BaseAPIClient` serve de base para o cliente Fintoc |
| `saq-utilities` (helpers BR) | não usar | CPF/CNPJ, ISPB, E2E, chave Pix, "R$", UTC-3 fixo, dias úteis sem feriados do México. O México precisa de validador de CLABE (dígito próprio), RFC/CURP, MXN, America/Mexico_City |
| `saq-permissions` | **A** | Via sessão `CORE_READ`, como o crypto faz. Adicionar `_Mexico` |
| `saq-cache-client` | **B** | Ler user/roles/company do cache, sim. Para o saldo, copiar o padrão das Lua scripts com **prefixo de chave próprio**. As funções atuais usam `account:<id>` / `account_mirror:<id>`, e IDs do México colidiriam com os do core |
| `saq-repository` (core) | só leitura | Usuários, company, whitelabel, policies. Nada de conta, transação ou tarifa: é tudo Brasil |
| `saq-ci` | **A** | `docker-kustomize-deploy.yml`, `chalice-deploy.yml`, `publish-codeartifact.yml`. Adicionar o tipo de segredo `rds_mexico` |
| Kubernetes-manifest | **A** | Criar `mexico-apps/config-commons-mexico/` no molde de `crypto-apps/config-commons-crypto/` |
| saq-infra | **B** | Novo RDS (ou schema), repositórios no ECR e segredos |
| `microservice-template` | não usar | Desatualizado e com sabor Pix. Usar o crypto-api-v3 como template |

### Frontends

| Repo | Veredito | Observação |
|---|---|---|
| `backoffice-whitelabel-frontend` (Next 15) | **B** | Hospeda um módulo `mexico/` igual ao `anticipation/`, com base URL própria e o mesmo token de login. As telas do Brasil não servem: CPF/CNPJ e Pix estão espalhados, `format-currency` é fixo em BRL e as strings estão em pt-BR fixo |
| `portal-whitelabel-frontend` (Next 16) | **B** / C | Já é i18n (pt/en/zh); falta `es`. Extrato, saldo, webhooks e usuário servem de base, desde que moeda e locale sejam parametrizados. CLABE, SPEI OUT e cobrança são telas novas |
| `onboarding-whitelabel-frontend` | B / **C** | O shell e o stepper servem. Campos e validação são todos do Brasil (CPF/CNPJ, CEP, sócios) |
| `qas-frontend-libs` | — | Vazio, sem commits |

## 5. Decisões que destravam o desenho

1. **Ledger do México.** Com banco separado, o saldo e as transações não podem ir para `finance_accountbanktransaction` do core. Proposta: tabela de lançamentos própria no banco do México + saldo no Valkey com prefixo próprio. É a parte mais nova do projeto, porque nenhum produto atual tem ledger próprio.
2. **Onde ficam usuário e conta.** No core, `UserRole` e `WebhooksCompany` têm FK para `finance_accountbankmirror`, e a permissão por conta (`has_policy_by_account_id`) depende disso. Há dois caminhos:
   - **(i)** Criar uma mirror "fantasma" no core para cada conta mexicana. Reaproveita mais do core, mas coloca contas MXN no modelo de conta do Brasil.
   - **(ii)** Manter usuário, login e policies no core, e o vínculo usuário↔conta, os webhooks e as tarifas no banco do México. **Recomendado:** mantém o banco separado de verdade e ainda reaproveita o login e o backoffice.
3. **Valores.** O core usa `DOUBLE`. No México, guardar em centavos inteiros (ou `DECIMAL`), como a Fintoc, com coluna de moeda.
4. **Idioma.** Espanhol ou português nas telas e nas mensagens decide se `saq-base-api` e o portal ganham `es`.

## 6. Achado à parte

`apps-v3/microservice-template/requirements.txt` tem um token pessoal do GitHub (PAT) gravado nas URLs `git+https`. Avisar o time para revogar.
