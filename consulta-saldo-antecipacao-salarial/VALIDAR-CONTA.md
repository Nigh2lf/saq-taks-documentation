# Validar o acesso de uma conta à antecipação salarial (e bater os documentos)

Dado um **usuário** (`core_user.id`), a **company do token** e uma **agência/conta**, isto é tudo
que a `anticipation-api-v3` lê antes de responder
`GET/POST /v3/anticipation/accounts/{agência}/{conta}/payroll/...`, na mesma ordem em que ela lê.
O fluxo detalhado e os erros estão em `FLUXO-PAYROLL.md`; aqui é só o roteiro de consulta.

Três fontes, três acessos diferentes:

| Fonte | O que tem | Como acessar |
|---|---|---|
| **SAQ** (core, MySQL) | usuário, roles, policies, conta-espelho, bolsão, company (documento) | `DB_READ_*` do `.env` (local: `127.0.0.1:3307`, `saq`/`saq`) |
| **Valkey** (cache) | `user:role:{user}:web` e `company:{company}` — **a auth só lê daqui** | local: `docker exec saq-local-dev-valkey-1 valkey-cli --tls --cacert /certs/ca.crt ...`; nonprod: `valkey-cli -h $MEMORYDB_HOST --tls -c ...` (VPN) |
| **ANTICIPATION** (MySQL) | folha (`payroll_*`), parâmetros, contratos | `DB_ANTICIPATION_*` do `.env` (local: mesmo MySQL, banco `ANTICIPATION`) |

## Atalho: `local-dev/check_account.py`

Faz todas as consultas abaixo, imprime `[OK]/[FAIL]/[WARN]` por etapa com o HTTP que a API
devolveria, monta a matriz de documentos e sai com código 1 se algo bloqueia.

```bash
cd ~/Rodrigo/SAQ/anticipation-apps/anticipation-api-v3
.venv/bin/python ../../local-dev/check_account.py --user-id 857 --company-id 767 --agency 0001 --account 9001830
# --company-id omitido → usa core_user.company_id (avisa). --today 2026-10-02 simula outra data. --skip-valkey pula o cache.
```

Usa o `.env` da API: apontando `DB_READ_*`, `DB_ANTICIPATION_*` e `MEMORYDB_HOST` para o
nonprod (VPN), valida uma conta real sem gerar token nem subir a API. Validado em 01/10/2026
contra a API local: o script previu `200 {5000, 1499, 1.00}` e a API respondeu isso.

## Dados de entrada

- **`user_id` e `company_id`** vêm da claim `eux` do JWT (cifrada; não dá para ler no jwt.io).
  No banco: `SELECT id, company_id, email, is_active FROM core_user WHERE email = '...';`
  O `company_id` do token é o da sessão de login (pode ser diferente de `core_user.company_id`
  quando o usuário troca de empresa no portal).
- **agência/conta** são os path params; têm que ser de uma **conta-espelho** (`finance_accountbankmirror`),
  não de uma `finance_accountbank`.

## Roteiro SQL

### 1. Usuário, roles e policies no banco (SAQ)

```sql
-- usuário
SELECT id, email, name, is_active, is_deleted, company_id FROM core_user WHERE id = :user_id;

-- roles do usuário: uma linha por (company, conta-espelho, audience)
SELECT ur.id, ur.company_id, ur.audience, ur.role_id, r.name AS role_name,
       ur.account_mirror_id, ur.account_id
FROM core_userrole ur JOIN core_role r ON r.id = ur.role_id
WHERE ur.user_id = :user_id;

-- policies de cada role (é isto que o Django copia para o Valkey)
SELECT ur.id AS userrole_id, ur.company_id, ur.account_mirror_id, p.name AS policy
FROM core_userrole ur
JOIN core_rolepolicy rp ON rp.role_id = ur.role_id
JOIN core_policy     p  ON p.id = rp.policy_id
WHERE ur.user_id = :user_id
  AND p.name LIKE 'anticipation:payroll:%';          -- GET: :read   POST: :request

-- catálogo (só importa para token com claim `pm`): posição do bit = ordem por id
SELECT id, name, allowed_audiences FROM core_policy WHERE name LIKE 'anticipation:payroll:%';
```

Esperado: `core_user` ativo e não deletado; pelo menos uma `core_userrole` com `audience = 'web'`,
`company_id` = company do token e `account_mirror_id` = a conta-espelho; a role com
`anticipation:payroll:read` (e `:request` para o POST); as duas policies no catálogo.

### 2. Conta-espelho → bolsão → documento do titular (SAQ)

```sql
SELECT m.id AS mirror_id, m.agency, m.account, m.company_id AS mirror_company, m.status AS mirror_status,
       mc.cnpj  AS mirror_company_doc, mc.name AS mirror_company_name,
       b.id  AS bolsao_id, b.document AS bolsao_document, b.is_bolsao, b.status AS bolsao_status,
       bc.id AS bolsao_company, bc.cnpj AS DOCUMENTO_USADO_PELA_API, bc.name AS bolsao_company_name
FROM core_userrole ur
JOIN finance_accountbankmirror m  ON m.id = ur.account_mirror_id
LEFT JOIN core_company         mc ON mc.id = m.company_id
LEFT JOIN finance_accountbank  b  ON b.id = m.bolsao_id
LEFT JOIN core_company         bc ON bc.id = b.company_id
WHERE ur.user_id = :user_id;
```

A API pega **todas** as linhas e filtra em memória: `agency = :agência AND account = :conta AND
mirror_company = company do token`. Nenhuma linha → `404 Account not found`. O documento usado no
cálculo é **`bc.cnpj` (company do bolsão)**, sem máscara; se for inválido → `404 No valid document`.

### 3. Valkey (o que a auth realmente lê)

```bash
# local
docker exec saq-local-dev-valkey-1 valkey-cli --tls --cacert /certs/ca.crt GET user:role:857:web
docker exec saq-local-dev-valkey-1 valkey-cli --tls --cacert /certs/ca.crt GET company:767
# nonprod (VPN)
valkey-cli -h $MEMORYDB_HOST --tls -c GET user:role:857:web
```

`user:role:{user}:web` é um JSON `{"id", "audience", "roles": [{"company_id", "account_mirror_id",
"account_id", "policies": [...]}]}`. Passa se existe **uma** role com `company_id` = company do
token **e** `account_mirror_id` = mirror da etapa 2 **e** `policies` contendo o scope da rota.
Chave ausente = usuário sem roles → `404 Account not found` (não há fallback para o banco).
`company:{id}` não é usado pela rota de payroll, mas é por outras rotas e pelo HMAC.

Se o banco (etapa 1) e o cache divergem, o cache está desatualizado: no nonprod é o Django que
regrava ao salvar o usuário/role; no local, é o `make_token.py` que semeia. **O Valkey local não
persiste** (`--save ""`): a cada `docker compose up` as chaves somem — rode o `make_token.py` de novo.

### 4. Folha, parâmetros e contratos (ANTICIPATION) — para o documento da etapa 2

```sql
-- vínculo: a API usa o ACTIVE com admissão mais recente
SELECT e.id, e.status, e.admission_date, e.termination_date, e.gross_salary_amount, e.net_salary_amount,
       r.id AS employer_id, r.document AS employer_cnpj, r.name, r.is_active, r.payment_day, r.payment_day_type
FROM payroll_employee e JOIN payroll_employer r ON r.id = e.employer_id
WHERE e.document = :cpf
ORDER BY e.admission_date DESC, e.id DESC;

-- parâmetros
SELECT * FROM parameter_default WHERE `key` IN ('payroll_advance_rate', 'payroll_service_fee');

-- contratos salariais do CPF (todos) — veja quais vencem dentro do período
SELECT agreement_number, agreement_status, maturity_date, total_amount, created_at
FROM contract_creditagreement
WHERE client_document = :cpf AND credit_product = 'ANTICIPATION_SALARY'
ORDER BY created_at DESC;

-- o que consome limite no período (mesma query da API)
SELECT COALESCE(SUM(total_amount), 0) AS anticipated
FROM contract_creditagreement
WHERE client_document  = :cpf
  AND credit_product   = 'ANTICIPATION_SALARY'
  AND maturity_date BETWEEN :period_start AND :period_end
  AND agreement_status IN ('PENDING_RELEASE','RELEASE_IN_PROGRESS','ANTICIPATED','LIQUIDATED','OVERDUE');
```

Período (`resolve_reference_period`): `period_end` = dia de pagamento deste mês (`payment_day`,
corrido ou N-ésimo dia útil); se já passou, o do mês que vem; `period_start` = pagamento do mês
anterior + 1 dia. Sem `payment_day`: mês-calendário. O script calcula isso com a mesma função da API.

Esperado: `status = 'ACTIVE'`, `gross_salary_amount > 0`, `is_active = 1`, `termination_date` nula
ou maior que `period_end`, `payroll_advance_rate > 0`, e para o POST `payroll_service_fee ≥ 0` e
`period_end > hoje`. Disponível = `bruto × taxa/100 − anticipated`, nunca negativo.

### 5. Bate-documentos

Tudo abaixo deveria ser o **mesmo CPF**:

| Origem | Coluna | Se for diferente |
|---|---|---|
| company da conta-espelho | `core_company.cnpj` via `finance_accountbankmirror.company_id` | conta de **whitelabel**: a API calcula para o dono do bolsão, não para o titular da conta (ponto em aberto nº 1 do `FLUXO-PAYROLL.md`) |
| bolsão | `finance_accountbank.document` | informativo; a API não usa |
| company do bolsão | `core_company.cnpj` via `finance_accountbank.company_id` | **é o documento que a API usa** |
| Valkey | `company:{id}.cnpj` | cache desatualizado em relação ao core |
| folha | `payroll_employee.document` | sem vínculo para o documento usado → `404 Employee not found` |
| contratos | `contract_creditagreement.client_document` | contratos de outro CPF não entram na soma |

Exemplo do nonprod: `0001/9001830` (mirror 409, company 767, CPF 52998224725) tem bolsão 692 da
company 766 (CPF 08289064594) → a API usa 08289064594 e o contrato sai `SAL-08289064594-…`.

## Erros, por etapa

| Falhou em | HTTP |
|---|---|
| JWT (chave, `exp`, `aud`, `token_type`) | 401 |
| scope ausente no token / bitmap `pm` | 403 Missing scopes |
| etapa 2 (userrole × mirror × company) **ou** etapa 3 (Valkey sem policy no mirror) | 404 Account not found |
| documento do bolsão inválido | 404 No valid document for user scope |
| documento é CNPJ | 400 only available to individuals |
| sem `payroll_employee` ACTIVE | 404 Employee not found for authenticated user |
| salário, empregador inativo, desligamento, taxa | 400 (mensagens no `FLUXO-PAYROLL.md`) |
| `parameter_default` sem a chave | 500 |
