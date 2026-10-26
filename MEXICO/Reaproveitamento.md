# Operação México — o que aproveitar da SaQ e o que criar do zero

Levantado em 19/10/2026 a partir do código do workspace. Premissa, confirmada pelo TL em 20/10/2026: o México é um **produto totalmente separado**, com banco, backoffice (API e frontend) e autenticação próprios. O core da SaQ não entra no caminho de login. As decisões estão registradas na seção 8. Complementa `OperacaoMexico.md` e `Pendencias.md`.

## 1. Resumo

- **A SaQ tem o molde, mas o México vai além dele.** Crypto (`crypto-apps/`) e Antecipação (`anticipation-apps/`) são produtos com banco próprio, repository próprio e libs próprias, mas usam o core da SaQ em modo leitura para login, permissões e empresa. O México segue a mesma estrutura de pastas (`mexico-apps/`) e **não abre o banco do core**: usuário, login e policies ficam no banco do México (seção 6).
- **A plataforma é reaproveitada como está:** libs base, validação de JWT, CI/deploy.
- **Os padrões são copiados e adaptados:** login e MFA, onboarding, ledger no Valkey, tarifas, webhooks para o cliente, extrato, backoffice.
- **Tudo que fala com o SPEI é novo:** cliente Fintoc, receptor de webhook da Fintoc, máquina de estados do envio, conciliação, CLABE, KYC mexicano e telas.
- **Diferença em relação aos precedentes.** Crypto e Antecipação **não têm ledger próprio**: movimentam dinheiro na conta SaQ do core, em `finance_accountbanktransaction` e no saldo do Valkey. Com banco separado e MXN, o México precisa de **ledger próprio**. Nenhum produto hoje faz isso.

## 2. Estrutura proposta



```
NOVO — repositórios do México

mexico-apps/
  mexico-repository/          copiar o esqueleto do anticipation-repository (alembic, base model, publicação)
  mexico-shared-libraries/    pacotes mexico-fintoc-client (molde: crypto-b2b-client), mexico-permissions, mexico-cache (Lua scripts com prefixo mx:)
  mexico-api-v3/              portal do cliente + API pública + login (/v3/auth/*) + receptor de webhook Fintoc (molde: crypto-api-v3)
  mexico-backoffice-api/      molde: anticipation-backoffice-api, com login de operador próprio
  mexico-backoffice-frontend/ app novo (shell e componentes copiados do backoffice-whitelabel-frontend)
  mexico-portal-frontend/     app novo (base: portal-whitelabel-frontend)
  lambdas/                    SPEI OUT, SPEI IN, webhooks para o cliente, conciliação
  cron-jobs/                  conciliação diária, varredura de tarifa (se for essa a opção)

COMPARTILHADO — repositórios da SaQ consumidos como estão

shared/
  saq-shared-libraries/  pacotes saq-base-api (+ locale es-MX), saq-base-repository e saq-utilities (só auth, aws, http, exceptions, cache, sqs e tracing)
  saq-ci/                workflows docker-kustomize-deploy, chalice-deploy, publish-codeartifact

apps-v3
  Kubernetes-manifest/  pasta mexico-apps/, namespace apps-mexico, config-commons-mexico
  saq-infra/            RDS, ECR, filas e segredos do México declarados aqui
```

MOLDE — copiados como código, nunca importados

```
apps-v3/auth-token-api-v3, user-api-v3, saq-backoffice-api        login, MFA, senha, token e CRUD de usuários
crypto-apps/crypto-api-v3, lambda-aws-crypto-order-processing     config, wiring, webhook do provedor, envio com estados
anticipation-apps/anticipation-backoffice-api, anticipation-repository   backoffice, scopes, auditoria, repository
shared/saq-shared-libraries/packages/saq-cache-client            padrão das Lua scripts de saldo
apps-v3/lambdas/lambda-aws-company-notification-event            motor de webhooks para o cliente
apps-frontend/portal-whitelabel-frontend, backoffice-whitelabel-frontend   base dos apps novos
```

O México tem **conta AWS própria** (decidido em 20/10/2026), e mesmo assim `saq-ci`, `Kubernetes-manifest` e `saq-infra` continuam compartilhados. A SaQ já funciona assim: cada cliente whitelabel (adopay, simpay, bnscred, socred, owempay, credimovel) tem uma conta de dev e uma de prod, criadas pelo `saq-infra`, implantadas pelo `saq-ci` e com overlays próprios no `Kubernetes-manifest`. O México entra como mais um par de contas desse modelo. Detalhe na seção 9.

### Repositórios da SaQ usados de forma compartilhada

Os repositórios abaixo não são copiados: o México os consome como estão. Tudo o que não está nesta lista é copiado para `mexico-apps/` ou criado do zero.

| Repositório | O que o México usa | Como consome | Observação |
|---|---|---|---|
| `shared/saq-shared-libraries` | pacote `saq-base-api` | versão fixada no `requirements.txt`, via CodeArtifact | precisa ganhar o locale `es-MX` |
| `shared/saq-shared-libraries` | pacote `saq-base-repository` | idem | sem mudança |
| `shared/saq-shared-libraries` | pacote `saq-utilities`, só os módulos `auth`, `aws`, `http`, `exceptions`, `cache`, `sqs` e tracing | idem | os helpers do Brasil (`CPF/CNPJ`, ISPB, Pix, moeda, fuso) ficam fora por convenção; a lib não separa isso em pacotes |
| `shared/saq-ci` | workflows `docker-kustomize-deploy.yml`, `chalice-deploy.yml`, `publish-codeartifact.yml` | `uses: SaQDigital/saq-ci/...@master` no workflow de cada app | build e imagem na conta shared (ECR e CodeArtifact); o deploy das lambdas usa a conta do bloco em `actions/chalice-config/environments.json`, que ganha os stages do México |
| `apps-v3/Kubernetes-manifest` | repo único de manifests, ArgoCD, pasta `config-commons-*` por produto | pasta `mexico-apps/` dentro do repo, com overlays `dev-mexico` e `prod-mexico` | cada conta tem o seu cluster e o seu ArgoCD (`destination: kubernetes.default.svc`); o overlay escolhe o cluster |
| `apps-v3/saq-infra` | definição de RDS, ECR, segredos e filas | contas novas em `aws-root`; stacks `Pulumi.<stage>.yaml` do México em `network`, `app-eks` e demais | é o mesmo caminho das contas dos clientes whitelabel |

Os demais pacotes de `saq-shared-libraries` (`saq-cache-client`, `saq-permissions`, `saq-report-filters`, `saq-caf-client`, `saq-novavida-client`, `saq-bacen-sta-client`) e o `shared/saq-repository` **não** são consumidos: ou carregam o core, ou são do Brasil. O `saq-cache-client` serve só como molde para copiar o padrão das Lua scripts (seção 4).

### Arquivos de referência

| Para quê | Copiar de |
|---|---|
| Repository (alembic, base model, publicação) | `anticipation-apps/anticipation-repository/`: `migrations/env.py`, `monorepo_setup.py`, `setup.py`, `VERSION`, `Dockerfile` |
| Nomenclatura de tabelas | `anticipation-apps/anticipation-repository/NOMENCLATURA-BANCO.md`. A cópia do crypto é mais antiga |
| Config, startup e wiring do auth (um banco só: `MEXICO_WRITE`) | `crypto-apps/crypto-api-v3/main/config.py`, `main/db.py`, `main/app.py`. Tirar a sessão `CORE_READ` |
| Login web em dois passos (senha → token temporário → TOTP), login API por `client_id`/`client_secret`, emissão do JWT, lockout | `apps-v3/auth-token-api-v3/main/routes/v3/web_token.py`, `routes/v2/auth_token.py`, `helpers/token_helper.py`, `helpers/login_lockout.py`, `helpers/validate_mfa_token.py`, `helpers/verify_password.py` |
| Registro de MFA, troca e reset de senha | `apps-v3/user-api-v3/main/routes/v3/` (`mfa.py`, `change_password.py`, `change_password_temporary.py`, `reset_password.py`) |
| Token e login do operador de backoffice | `apps-v3/saq-backoffice-api/main/helpers/jwt_helper.py`, `helpers/backoffice_user.py`, `helpers/login_lockout.py` |
| CRUD de usuários, roles e policies do backoffice | `apps-v3/saq-backoffice-api/main/routes/v3/` (`backoffice_users.py`, `users_web.py`, `user_roles.py`) |
| Modelos de identidade (usuário, role, policy, vínculo usuário↔conta, OTP, tentativas de login, reset de senha) | `shared/saq-repository/saq_repository/models/core/core_models.py` (`User`, `Role`, `Policy`, `RolePolicy`, `UserRole`, `Backoffice*`, `UserLoginAttempt`, `UserPasswordReset`) e `models/otp` |
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
| Gestão do onboarding e das contas | **B** | O formato das rotas de `saq-backoffice-api` (`onboarding.py`, `list_accounts_combined.py`, `close_account.py`) e a estrutura de API separada do anticipation-backoffice | As consultas, sobre o banco do México. Bloquear a conta desativa a CLABE na Fintoc (`PATCH /v2/account_numbers/{id}` com `status: disabled`). A `Account` só permite editar a `description` e não tem bloqueio por API |
| Configuração de taxas por conta | **B** | A lógica de `CompanyTariff` (VALUE / PERCENT_WITH_MIN / PERCENT_WITH_MIN_AND_MAX, vigência), `company_tariff_calculator.py` e `tax_split_ledger.py`. A taxa é gravada em transação separada, o que já atende a RN-029 | O modelo de tarifa vive no core e tem campos BRL (`usd_to_brl_exchange_rate`, `today_brazil`, PF/PJ pelo tamanho do documento). Recriar no banco do México, com moeda e centavos. Cobrança por `Transfer` real na Fintoc |
| Administração de usuários | **B** | O desenho de usuário, role, policy e vínculo usuário↔conta do core; as rotas de CRUD do `saq-backoffice-api` (`backoffice_users.py`, `users_web.py`, `user_roles.py`); o login de `auth-token-api-v3` e o MFA e senha de `user-api-v3`. Tudo copiado como código, sobre o banco do México | Tabelas `mx_user`, `mx_role`, `mx_policy`, `mx_userrole` e as de backoffice; chave JWT e audiences próprias; catálogo de policies carregado de `mx_policy`. Os dois perfis do PRD (`Cliente:web`, `backoffice:admin`) viram roles com policies no banco do México. Sem whitelabel. Ver seção 6 |
| Criação de webhooks | **B** | O motor de entrega: `apps-v3/lambdas/lambda-aws-company-notification-event` (HMAC, plano de envio, log), além do reenvio (`pix-webhook-api-v3` + `webhook-resend-runner`). O motor não depende do conteúdo do payload | Tabelas de cadastro e tipos de evento próprios (`SPEI_IN_RECEIVED`, `SPEI_OUT_*`). Os enums atuais são de Pix e `WebhooksCompany` aponta para a mirror do core. Faltam DLQ e retentativa melhor: hoje são 3 recebimentos e descarte (RN-055) |
| Extratos consolidados e detalhados | **B** | Os padrões de `saq-backoffice-api` (`statement.py`, `daily_balances_sum.py`), o export assíncrono (`report-request-runner`) e o saldo diário (`calculate-account-balance-cron-job`) | As consultas sobre o ledger do México. Conciliação dos `Movement`s da Fintoc contra o ledger; o modelo mais próximo é `PiAccountStatementEntry` |

## 4. Plataforma (libs e infra)

| Peça | Veredito | Observação |
|---|---|---|
| `saq-base-api` | **A** | Health, CORS, envelope de erro, paginação, métricas. Só traz `en` e `pt-BR`; para `es-MX`, adicionar o locale (mudança pequena na lib) |
| `saq-base-repository` | **A** | Sessões async com vários bancos nomeados, `SessionManager`, `BaseRepository`, lock nomeado do MySQL |
| `saq-utilities` (AWS, HTTP, auth/JWT, exceções, tracing) | **A** | `BaseAPIClient` serve de base para o cliente Fintoc. O pacote `auth` valida JWT, audience, scopes e bitmap sem tocar em banco: recebe chave, catálogo e loader de roles no startup (`initialize`, `set_permission_catalog`, `set_default_get_user_roles_fn`). É o que permite isolar o login (seção 6) |
| `saq-utilities` (helpers BR) | não usar | CPF/CNPJ, ISPB, E2E, chave Pix, "R$", UTC-3 fixo, dias úteis sem feriados do México. O México precisa de validador de CLABE (dígito próprio), RFC/CURP, MXN, America/Mexico_City |
| `saq-permissions` | não usar | Crypto e Antecipação também já saíram dele (seção 6). No México, o pacote `mexico-permissions` em `mexico-shared-libraries`, no formato do `anticipation-permissions`, com o catálogo carregado de `mx_policy` |
| `saq-cache-client` | **B** | Os DTOs de usuário e roles são montados a partir dos modelos do core e as chaves (`user:`, roles por `user_id`) não têm prefixo. Copiar só `get_user_roles`/`set_user_roles` com prefixo `mx:`, ou ler do banco no login enquanto o volume for baixo. Para o saldo, copiar o padrão das Lua scripts com **prefixo de chave próprio**: as funções atuais usam `account:<id>` / `account_mirror:<id>` |
| `saq-repository` (core) | não usar | Nenhuma tabela do core é lida. Usuário, policies e empresa ficam no banco do México; conta, transação e tarifa são Brasil |
| `saq-ci` | **A** | `docker-kustomize-deploy.yml`, `chalice-deploy.yml`, `publish-codeartifact.yml`. Adicionar o tipo de segredo `rds_mexico` |
| Kubernetes-manifest | **A** | Criar `mexico-apps/config-commons-mexico/` no molde de `crypto-apps/config-commons-crypto/`, e os overlays `dev-mexico`/`prod-mexico` e o ArgoCD da conta nova |
| saq-infra | **B** | Duas contas novas na Organization (`aws-root`) e as stacks Pulumi delas (rede, EKS, RDS, filas, segredos). Os repositórios ECR ficam na conta shared |
| `microservice-template` | não usar | Desatualizado e com sabor Pix. Usar o crypto-api-v3 como template |

### Frontends

| Repo | Veredito | Observação |
|---|---|---|
| `backoffice-whitelabel-frontend` (Next 15) | B / **C** | Vira um app novo (`mexico-backoffice-frontend`) com login próprio contra o `mexico-backoffice-api`. Copiar o shell, o layout, o `use-auth` e o padrão de client HTTP do módulo `anticipation/` (`api-anticipation.ts`). As telas do Brasil não servem: CPF/CNPJ e Pix estão espalhados, `format-currency` é fixo em BRL e as strings estão em pt-BR fixo |
| `portal-whitelabel-frontend` (Next 16) | **B** / C | App novo (`mexico-portal-frontend`) a partir dele. Já é i18n (pt/en/zh); falta `es`. Extrato, saldo, webhooks e usuário servem de base, desde que moeda e locale sejam parametrizados. CLABE, SPEI OUT e cobrança são telas novas. O login aponta para `/v3/auth/*` do `mexico-api-v3` |
| `onboarding-whitelabel-frontend` | B / **C** | O shell e o stepper servem. Campos e validação são todos do Brasil (CPF/CNPJ, CEP, sócios) |
| `qas-frontend-libs` | — | Vazio, sem commits |

## 5. Decisões que destravam o desenho

1. **Ledger do México.** Com banco separado, o saldo e as transações não podem ir para `finance_accountbanktransaction` do core. Proposta: tabela de lançamentos própria no banco do México + saldo no Valkey com prefixo próprio. É a parte mais nova do projeto, porque nenhum produto atual tem ledger próprio.
2. **Onde ficam usuário e conta.** Decidido em 19/10/2026: o México é um produto separado, com backoffice próprio, então usuário, login, policies e o vínculo usuário↔conta ficam **todos no banco do México**. O core não é aberto. No core, `UserRole` e `WebhooksCompany` têm FK para `finance_accountbankmirror`; no México, `mx_userrole.account_id` aponta para a conta mexicana e a checagem por conta (`has_policy_by_account_id`) roda sobre ela. Detalhe na seção 6.
3. **Valores.** O core usa `DOUBLE`. No México, guardar em centavos inteiros (ou `DECIMAL`), como a Fintoc, com coluna de moeda.
4. **Idioma.** Espanhol ou português nas telas e nas mensagens decide se `saq-base-api` e o portal ganham `es`.

## 6. Autenticação e permissões isoladas

O que torna o isolamento barato: a validação em `saq-utilities.auth` não depende do banco. Ela recebe três coisas no startup e o resto é genérico:

| Ponto de extensão | O que a SaQ injeta hoje | O que o México injeta |
|---|---|---|
| `initialize(jwt_key)` | `JWT_TOKEN_KEY` (web/API) ou `BACKOFFICE_JWT_TOKEN_KEY` | `MEXICO_JWT_TOKEN_KEY` e `MEXICO_BACKOFFICE_JWT_TOKEN_KEY`, diferentes das chaves da SaQ. A claim `eux` (user e company cifrados) deriva do mesmo segredo, então fica isolada junto |
| `set_permission_catalog({nome: bit})` | `core_policy` ou `core_backofficepolicy` ordenadas por `id` | `mx_policy` e `mx_backofficepolicy` ordenadas por `id`. A ordem do bitmap `pm` segue o `id`, nunca reordenar |
| `set_default_get_user_roles_fn(loader)` | `get_user_roles` do `saq-cache-client`, que lê `core_userrole` | Classe própria que implementa `UserRolesProtocol` (`get_account_ids`, `has_policy_by_account_id`, ...) lendo `mx_userrole`. `is_account_mirror` é sempre falso |

Depois disso, as rotas usam `require_authenticated_audience` e `require_authenticated_audience_with_roles` exatamente como o crypto-api-v3 e o anticipation-backoffice-api.

### Pacote de permissões

Crypto e Antecipação já tiraram as permissões do `saq-permissions`: cada produto tem o seu pacote na própria shared (`crypto-shared-libraries/packages/crypto-permissions` e `anticipation-shared-libraries/packages/anticipation-permissions`). O formato é sempre o mesmo:

- Uma classe de constantes com o nome das policies (`AnticipationPermissions.NFSE_READ = "anticipation:nfse:read"`). Importar a classe não carrega banco.
- `configure_auth_permissions()`, chamada no startup: registra o loader de roles e carrega o catálogo no `saq_utilities.auth` (`set_permission_catalog`).
- Um seed SQL idempotente das policies (`scripts/seed_*_policies.sql`).

Nos dois produtos, o que saiu foi o **código**. O catálogo continua em `core_policy`, lido com `saq-repository` e `saq-cache-client`. O México copia o formato e troca a origem:

| | `anticipation-permissions` | `mexico-permissions` |
|---|---|---|
| Constantes | `AnticipationPermissions` | `MexicoPermissions` (`mexico:<recurso>:<ação>`) |
| Catálogo | `core_policy` via `saq-repository` | `mx_policy` via `mexico-repository` |
| Roles do usuário | `get_user_roles` do `saq-cache-client` (`core_userrole`) | loader próprio lendo `mx_userrole` |
| Seed | `INSERT INTO core_policy` | `INSERT INTO mx_policy` |
| Dependências | `saq-utilities`, `saq-base-repository`, `saq-cache-client`, `saq-repository` | `saq-utilities`, `saq-base-repository`, `mexico-repository` |

O backoffice segue a mesma ideia, com `mx_backofficepolicy` e a chave JWT do backoffice. Os pacotes novos da SaQ pedem `saq-utilities` e `saq-base-repository` >= 1.59.

### Modelo no banco do México

Copiar a forma das tabelas do core, sem whitelabel (é um tenant só):

- `mx_user` (e-mail, senha, `login_type` web/api/backoffice, `is_active`, `flow_change_password`, `company_id` → cliente mexicano)
- `mx_role` (nome, audience), `mx_policy` (nome, tipo, `allowed_audiences`), `mx_rolepolicy`
- `mx_userrole` (user, company, role, `account_id` → conta mexicana, audience)
- `mx_backofficerole`, `mx_backofficepolicy`, `mx_backofficerolepolicy`, `mx_backofficeuserrole`
- `mx_otptotpdevice`, `mx_userloginattempt`, `mx_userpasswordreset`

Os nomes seguem a regra de palavras coladas do `NOMENCLATURA-BANCO.md` e já estão criados no `mexico-repository` (migration `2f1febf5f622`).

Os dois perfis do PRD (`Cliente:web`, `backoffice:admin`) são duas roles com suas policies. A matriz de permissões (PS06, em aberto) define as policies.

### Serviços

- **Login do cliente** em `mexico-api-v3`, rotas `/v3/auth/*`: web em dois passos (senha → token temporário → TOTP) e API por `client_id`/`client_secret`, copiados do `auth-token-api-v3`. Audiences `mx-web` e `mx-api`. Registro de MFA, troca e reset de senha copiados do `user-api-v3`. Extrair para um serviço próprio depois, se precisar, é simples porque a lógica fica em lib.
- **Login do operador** em `mexico-backoffice-api`, com chave e catálogo próprios, copiado do `saq-backoffice-api` (`jwt_helper.py`, `backoffice_user.py`). O CRUD de usuários, roles e policies vem do mesmo repo.
- **Cache**: no começo, ler do banco no login. Se precisar de cache, copiar `get_user_roles`/`set_user_roles` com prefixo `mx:` e invalidar direto do backoffice do México, sem a fila `SQS_UPDATE_CACHE` e a lambda `lambda-aws-update-cache-event`.
- **Lockout e auditoria de login**: copiar `login_lockout.py` com prefixo de chave próprio no Valkey; gravar tentativas em `mx_userloginattempt` direto, sem o evento `SQS_LOG_NOTIFICATION`.

### O que some em relação ao desenho anterior

Sessão `CORE_READ` para login, classe `_Mexico` em `saq-permissions`, whitelabel México, linhas de policy em `core_policy` e o módulo `mexico/` dentro do `backoffice-whitelabel-frontend`.

## 7. Achado à parte

`apps-v3/microservice-template/requirements.txt` tem um token pessoal do GitHub (PAT) gravado nas URLs `git+https`. Avisar o time para revogar.

## 8. Registro de decisões

| Data | Decisão | Quem |
|---|---|---|
| 19/10/2026 | O México é um produto separado: banco, backoffice e login próprios. Usuário, policies e o vínculo usuário↔conta ficam no banco do México | Rodrigo Lima |
| 20/10/2026 | Confirmado: banco separado do SaQ. Da SaQ, entram só as libs base (`saq-base-api`, `saq-base-repository`, `saq-utilities`), `saq-ci`, `Kubernetes-manifest` e `saq-infra` | Rodrigo Barros (TL) |
| 20/10/2026 | Permissões num pacote próprio, `mexico-permissions` em `mexico-shared-libraries`, no formato que Crypto e Antecipação já usam, lendo `mx_policy`. Nada do `saq-permissions` | Rodrigo Barros (TL) |
| 20/10/2026 | Conta AWS própria para o México, no modelo de conta por cliente que a SaQ já usa. `saq-ci`, `Kubernetes-manifest` e `saq-infra` continuam compartilhados | Rodrigo Lima |
| 20/10/2026 | Tabelas com prefixo `mx_` e palavras coladas; PK inteira em usuário, cliente e conta (o JWT trabalha com ids inteiros) | Rodrigo Lima |

## 9. Conta AWS do México

A SaQ já tem uma conta AWS por ambiente e por cliente. A lista abaixo vem de `saq-ci/actions/chalice-config/environments.json`:

| Tier | Stages (conta própria cada) | Região |
|---|---|---|
| develop | `nonprod`, `adopayhml`, `simpayhml`, `socredhml`, `bnscredhml`, `credimovelhml`, `owenpayhml` | `us-east-1`, `sa-east-1` ou `us-east-2` |
| production | `prod`, `adopay`, `simpay`, `bnscred`, `socred`, `owempay` | `sa-east-1` |

ECR e CodeArtifact ficam na conta shared (`167506318540`). As contas de workload só puxam imagens e pacotes de lá.

Para o México entrar no mesmo modelo:

| Onde | O que criar |
|---|---|
| `saq-infra/aws-root` | duas contas na Organization, por exemplo `mexicohml` e `mexico` |
| `saq-infra/identity` | roles `InfraDeployAccess` nas duas contas, com trust na conta shared |
| `saq-infra` (`network`, `app-eks`, `eks-postinstall`, `utilities`) | `Pulumi.mexicohml.yaml` e `Pulumi.mexico.yaml`: VPC, EKS, RDS `MEXICO`, filas, Valkey e segredos. ECR dos apps do México na conta shared |
| `saq-ci/actions/chalice-config/environments.json` | blocos `mexicohml` (develop) e `mexico` (production), com conta, região, VPC e catálogo de segredos. As lambdas do México usam `only_stages` com só esses dois stages, e as lambdas da SaQ não implantam lá |
| `Kubernetes-manifest` | overlays `dev-mexico`/`prod-mexico` nos apps de `mexico-apps/`, `argocd/dev-mexico` e `argocd/prod-mexico`. O `MANIFEST_ENVIRONMENTS_DEV/PROD` dos repos do México aponta só para esses overlays |

Os workflows que o `mexico-repository` já tem (`development.yaml`, `production.yaml`) não mudam: eles chamam o `docker-kustomize-deploy.yml` com o overlay que vier da variável do repo.

Em aberto:

- **Região.** Todas as contas de prod da SaQ estão em `sa-east-1`. A AWS tem região no México (`mx-central-1`). Se houver exigência de dados no país, a conta de prod precisa nascer lá, e é preciso confirmar se os serviços usados (EKS, MemoryDB, RDS) estão disponíveis nela.
- **Nome dos stages.** `mexicohml`/`mexico` segue o padrão dos clientes, mas depende de quem cria as contas.
- **Quem cria as contas.** O `aws-root` roda na conta root da Organization, que é trabalho do time de infra, não do backend.
