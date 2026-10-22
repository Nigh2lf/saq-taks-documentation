# Onboarding México — sandbox e infra

Escrito em 22/10/2026. Registro do que foi feito para rodar o onboarding com abertura de conta na Fintoc, primeiro na máquina local e depois na AWS. Complementa `Pendencias.md` (que foi escrito antes de existir código) e `Reaproveitamento.md` (desenho dos repositórios).

## 1. Onde o código está

Cinco repositórios no GitHub da SaQDigital, criados em 20/10/2026, todos com a branch de trabalho `feature/mexico-onboarding-schema`. No workspace ficam em `mexico-apps/`, espelhando `crypto-apps/`.

| Repositório | O que é |
|---|---|
| `mexico-api-v3` | API do cliente. Na V0 só o onboarding público: `POST /v3/onboarding/` e `GET /v3/onboarding/{id}` com `X-Onboarding-Token` |
| `mexico-backoffice-api` | Login do operador (senha + TOTP) e gestão dos pedidos: listar, detalhe, aprovar, reprovar, reenviar |
| `mexico-repository` | Modelos e migrations do banco `MEXICO` (`mx_*`) |
| `mexico-shared-libraries` | `mexico-fintoc-client` (HTTP da Fintoc) e `mexico-permissions` (policies, roles e seeds) |
| `lambda-aws-mexico-onboarding-approve-event` | Consome a fila do approve, cria `mx_company`, `mx_account`, a `Account` na Fintoc e o `mx_user` web |

O fluxo:

```
mexico-api-v3          POST /v3/onboarding   →  PENDING
mexico-backoffice-api  approve               →  APPROVED + mensagem na fila mexico-onboarding-approve
lambda                 Account na Fintoc (nasce ativa, com CLABE), mx_company, mx_account ACTIVE, mx_user web
                       →  COMPLETED (ou ERROR; o operador reenvia)
```

Não existe passo de "ativação" separado na Fintoc: o `POST /v2/accounts` já devolve a conta `active` com a CLABE. A lambda valida isso antes de gravar.

## 2. Nomes combinados com a infra

Passados ao Bruno (infra) em 22/10/2026. O identificador do projeto é **mexico**, no mesmo padrão de `anticipation` e `crypto`, com uma diferença: o México **não acessa nada da SaQ** (nem o banco do core, nem `credenciais-auth-token`, nem o `apps-configs` compartilhado). Login, policies e usuários ficam no próprio banco `MEXICO`.

| Recurso | Nome |
|---|---|
| Banco | MySQL, schema `MEXICO`, instância separada. Usuário de aplicação e usuário de migrate |
| Secrets | `credenciais-rds-saq-mexico`, `credenciais-rds-saq-mexico-migrate`, `credenciais-partner-mexico`, `apps-configs-mexico` |
| Lambda | role `mexico` (como `lambdas-anticipation`); stages `mexicohml` e `mexico` no `saq-ci` |
| Filas | `mexico-onboarding-approve` com DLQ (`maxReceiveCount` 3, visibilidade maior que 120 s); `mexico-onboarding-completed` quando houver consumidor |
| Kubernetes | `mexico-apps/mexico-api-v3` e `mexico-apps/mexico-backoffice-api` no `Kubernetes-manifest`; ECR com os mesmos nomes |

O sandbox está na conta **nonprod da SaQ** (741254594040, us-east-1). Por isso os nomes levam `saq` no meio, como crypto e antecipação. Se um dia o México for para conta própria, só muda o nome e o ARN no `environments.json`; o código não muda.

### Conteúdo de cada secret

Cada secret é um JSON cujas chaves são exatamente os nomes das variáveis de ambiente, porque o deployment exporta o JSON direto. Tudo em string.

**`credenciais-rds-saq-mexico`** (lambda, as duas APIs, migration)

```json
{ "DB_USER": "...", "DB_PASSWORD": "...", "DB_HOST": "...", "DB_PORT": "3306", "DB_DATABASE": "MEXICO", "DB_USE_SSL": "true" }
```

**`credenciais-partner-mexico`** (lambda; depois a API de conta). Criado em 22/10 com a chave de teste.

```json
{ "FINTOC_SECRET_KEY": "sk_test_...", "FINTOC_ENTITY_ID": "ent_..." }
```

**`apps-configs-mexico`** (backoffice, api-v3, lambda). Criado em 22/10 com valores provisórios; a URL da fila e o host do backoffice entram quando existirem.

```json
{
  "MEXICO_BACKOFFICE_JWT_TOKEN_KEY": "<segredo longo, diferente das chaves da SaQ>",
  "SQS_MEXICO_ONBOARDING_APPROVE": "https://sqs.us-east-1.amazonaws.com/<conta>/mexico-onboarding-approve",
  "API_BASE_URL_BACKOFFICE": "https://<host do backoffice>",
  "OAUTH2_TOKEN_URL": "/v3/auth",
  "ENVIRONMENT": "develop",
  "ENABLE_TRACE_SQL": "false",
  "BACKOFFICE_JWT_TOKEN_VALIDATE": "60",
  "BACKOFFICE_MFA_TOKEN_VALIDATE": "2",
  "LOGIN_MAX_FAILED_ATTEMPTS": "5",
  "LOGIN_LOCKOUT_SECONDS": "900",
  "LOGIN_FAIL_WINDOW_SECONDS": "900"
}
```

A chave JWT ficou no `apps-configs-mexico`, e não num `credenciais-auth-token-mexico`, para não criar um quinto secret. `SQS_ONBOARDING_COMPLETED` fica de fora até existir consumidor. Na lambda, `SQS_MEXICO_ONBOARDING_APPROVE` vem do config do chalice, não do secret.

**`credenciais-rds-saq-mexico-migrate`**: só quando existir o cron job de migration no cluster. A primeira migration pode ser rodada da máquina, pela VPN, se o usuário de aplicação tiver DDL.

## 3. Fintoc sandbox

A chave de teste (`sk_test_`) da conta da QAS tem **uma só Entity**, em modo test e operacional: `ent_3D0BUOgkUNzjaryQXjeVKGEeVCK`. É esse o `FINTOC_ENTITY_ID`. Foi encontrada com `GET /v2/entities` usando a própria chave.

Para onboarding não precisa de JWS nem de webhook. Só a chave. CLABEs do ambiente de teste começam com `999`.

## 4. Ambiente local

Fica em `mexico-apps/`, que é só uma pasta do workspace, não um repositório. Os quatro READMEs dos serviços apontam para o `docker-compose.yml` daqui; ainda falta decidir em qual repositório ele entra.

| Item | Onde |
|---|---|
| `docker-compose.yml` | MySQL 8.0 na porta 3308 (`mexico`/`mexico`, banco `MEXICO`, utf8mb4) e LocalStack na 4566 com as filas `mexico-onboarding-approve`, sua DLQ e `mexico-onboarding-completed`, criadas por `docker/localstack/init-queues.sh` |
| `venv/` | Python 3.10. As libs SaQ (`saq-base-repository`, `saq-utilities`, `saq-base-api`, versão 1.48.0) instaladas em modo editável a partir do clone `shared/saq-shared-libraries`, porque a máquina não tem acesso ao CodeArtifact. Os pacotes do México também em editável |
| `postman/mexico-onboarding.postman_collection.json` | 11 requisições na ordem do fluxo, variáveis na própria collection; os scripts guardam os tokens e o id do pedido |

Como subir, na ordem:

```bash
cd mexico-apps && docker compose up -d
cd mexico-repository && python -m alembic -c alembic.ini upgrade head      # DB_* do .env.example
mysql -h 127.0.0.1 -P 3308 -u mexico -pmexico MEXICO < ../mexico-shared-libraries/packages/mexico-permissions/scripts/seed_mx_backoffice_policies.sql
#  depois seed_mx_policies.sql e seed_mx_roles.sql
cd ../mexico-backoffice-api && python -m scripts.create_operator --email ana@qas.mx --name "Ana"   # pede a senha, imprime o otpauth
uvicorn main.app:app --port 8000          # backoffice
uvicorn main.app:app --port 8001          # api-v3, na pasta dela
```

Na lambda, o `.env` leva o banco local e `FINTOC_SECRET_KEY` / `FINTOC_ENTITY_ID`. Há a opção de apontar `SECRET_MANAGER_CREDENCIAIS_PARTNER_MEXICO=credenciais-partner-mexico` e ler o secret da AWS pelo boto3, mas isso exige credenciais AWS válidas na máquina, e em 22/10 nem o perfil `default` nem a trinca temporária do `.env` da raiz valiam. Ficou para depois.

Localmente o approve só publica na fila; nada consome. A lambda roda à mão, por id, e ignora a fila:

```bash
cd lambdas/lambda-aws-mexico-onboarding-approve-event
python test.py <onboarding_requisition_id>
```

## 5. Teste de ponta a ponta (22/10/2026)

Feito com o pedido `2ea3b717-c5de-4c65-92f6-6c21b2a59342`, RFC `ABC900101AB2`, pessoa moral:

1. `POST /v3/onboarding/` na api-v3: 201, `PENDING`.
2. Login do operador no backoffice (senha + TOTP) e `POST .../approve`: `APPROVED`, mensagem na fila do LocalStack.
3. `python test.py <id>`: `POST /v2/accounts` na Fintoc respondeu 201. Resultado `COMPLETED`.

No banco: pedido `COMPLETED` sem `error_reason`, `mx_company` e `mx_account` `ACTIVE`, CLABE `999969560468944131`, `fintoc_account_id` `acc_3L3nx3ygZYyq7KOMxH1WTKRo1i5`, `mx_user` web `r@r.com` sem senha e com `flow_change_password` ligado.

Uma primeira execução sem `FINTOC_SECRET_KEY` parou na etapa 2 com "falha temporária, volta para a fila", deixando `mx_company` e `mx_account` `PENDING`. A segunda execução retomou dali, sem duplicar nada. Confirma o desenho de reprocessamento da lambda.

Regras do RFC que valem no envio (o erro antigo dizia só "requires 12 characters" e foi trocado por uma mensagem com o formato; ajuste em `main/schemas/schemas.py` do `mexico-api-v3`, ainda sem commit):

| Tipo | Formato | Exemplo |
|---|---|---|
| `COMPANY` | 3 letras maiúsculas + AAMMDD + 3 de homoclave (12) | `ABC900101AB1` |
| `INDIVIDUAL` | 4 letras maiúsculas + AAMMDD + 3 de homoclave (13) | `GOMA900101AB1` |

RFC com pedido em andamento devolve o mesmo pedido (200, sem token). Para repetir o teste, use outro RFC.

## 6. O que falta para rodar na AWS

Na ordem em que destrava:

- [ ] Bruno: RDS com o schema `MEXICO` e usuários; fila `mexico-onboarding-approve` + DLQ; role `mexico`; ECR.
- [ ] Preencher `credenciais-rds-saq-mexico` e completar `apps-configs-mexico` (URL da fila, host do backoffice).
- [ ] Rodar migration e seeds no RDS; criar operador.
- [ ] `saq-ci`: `lambdas/config/lambda-aws-mexico-onboarding-approve-event.json` (proposta no README da lambda) e stages `mexicohml`/`mexico` no `actions/chalice-config/environments.json`.
- [ ] Tags `v0.1.0` em `mexico-repository` e `mexico-shared-libraries` para publicar no CodeArtifact (os apps fixam `0.1.0` e hoje não há nenhuma tag).
- [ ] `Kubernetes-manifest`: base e overlay `dev-mexico` de `mexico-api-v3` e `mexico-backoffice-api`.
- [ ] Revisão e merge das cinco branches `feature/mexico-onboarding-schema` (a `master` só tem o commit inicial).
- [ ] Decidir onde o `docker-compose.yml` e a collection do Postman ficam versionados.

Depois da conta ativa, para o cliente usar: rota de definir senha no `mexico-api-v3` e o e-mail com o link (o evento `ONBOARDING_COMPLETED` é o gancho), telas do portal e do backoffice, e a decisão de produto sobre KYC e documentos (na V0 o operador aprova só com RFC, nome, e-mail e telefone).
