# Antecipação salarial — fluxo dos endpoints

Referência da `anticipation-api-v3` (branch `merge/payroll-anticipated-solicitation-com-recebimento-v2`, 2026-09-30):

- `GET  /v3/anticipation/accounts/{agência}/{conta}/payroll/available-balance`
- `POST /v3/anticipation/accounts/{agência}/{conta}/payroll/advances[?dryRun=true]` — body `{"amount": "250.00"}`

Os dois compartilham as **etapas 1–3** (auth → conta-espelho → saldo). O POST acrescenta a 4
(montar o contrato) e a 5 (gravar).

## Bancos e tabelas envolvidos

| Banco | Lidas | Escritas |
|---|---|---|
| **Valkey** (cache) | `user:role:{user_id}:web`, `company:{company_id}` | — |
| **SAQ** (core, sessão `CORE_READ`) | `core_userrole`, `finance_accountbankmirror`, `finance_accountbank` (bolsão), `core_company` | — |
| **ANTICIPATION** (sessão `ANTICIPATION_WRITE`) | `payroll_employee`, `payroll_employer`, `parameter_default`, `contract_creditagreement` | `contract_creditagreement`, `contract_loanposition`, `contract_salaryadvance` (só POST sem `dryRun`) |

Arquivos principais (todos em `anticipation-apps/anticipation-api-v3/main/`):

| Etapa | Arquivo |
|---|---|
| 1 | `saq_utilities/auth/auth.py` (`require_authenticated_audience_with_roles`) — lib `saq-utilities` |
| 2 | `helpers/user_scope.py` (`resolve_account_scope_document`) |
| 3 | `helpers/payroll_balance_helper.py` (`calculate_available_balance`) |
| 4 | `helpers/payroll_advance_helper.py` (`build_salary_advance_contract`) |
| 5 | `routes/v3/post_payroll_advance.py` |

---

## Etapa 1 — Autenticação (as duas rotas)

`require_authenticated_audience_with_roles(allowed_audiences=["web"], required_scopes=[...])`

1. **Assinatura**: `jwt.decode(token, JWT_TOKEN_KEY, algorithms=["HS256"], audience=["web"])`.
   Chave inválida, token expirado ou `aud` diferente → **401 Not authenticated**.
2. Claim **`token_type`** deve ser `access` → senão 401.
3. **Scopes**:
   - se o token tem a claim `pm` (bitmap base64url), decodifica com o catálogo carregado no startup
     (`configure_auth_permissions`: `SELECT name FROM core_policy ORDER BY id` → posição do bit = índice);
   - se não tem `pm`, usa a lista `scope` do próprio token (é o caso do token gerado localmente).
   - GET exige `anticipation:payroll:read`; POST exige `anticipation:payroll:request`.
     Faltando → **403 Missing scopes**.
4. **Identidade**: claim `eux` (Fernet com `sha256(JWT_TOKEN_KEY)`) → `user_id` e `company_id`.
5. Devolve `UserAuthenticatedWithRoles` **sem** carregar as roles ainda (lazy — etapa 2.4).

Nenhum acesso a banco até aqui: só a chave JWT e o catálogo em memória.

---

## Etapa 2 — Resolver a conta-espelho e o CPF do titular

`resolve_account_scope_document(user_authenticated, agency, account)` — usa os path params.

### 2.1 Contas-espelho do usuário (core)

```sql
SELECT m.*
FROM finance_accountbankmirror m
JOIN core_userrole ur ON ur.account_mirror_id = m.id
WHERE ur.user_id = :user_id;
-- selectinload: m.company, m.bolsao (finance_accountbank), m.bolsao.company (core_company)
```

`AccountBankMirrorRepository.get_account_by_user`: **todas** as contas-espelho em que o usuário tem
alguma linha em `core_userrole`.

### 2.2 Filtro em memória

`mirror.agency == agência` **e** `mirror.account == conta` **e** `mirror.company_id == company_id do token`.
Nenhuma → **404 Account not found** (mesmo se a conta existir para outro usuário — não vaza existência).

### 2.3 Documento do titular

`mirror.bolsao.company.get_clean_document()`:

```
finance_accountbankmirror.bolsao_id → finance_accountbank.id
finance_accountbank.company_id      → core_company.id
core_company.cnpj                   → sem máscara (o campo aceita CPF ou CNPJ)
```

> ⚠️ É o **dono do bolsão**, não a empresa da conta-espelho. No nonprod, para `0001/9001830`
> (mirror 409, company 767, bolsão 692 da company 766 — whitelabel), o documento resolvido é o CPF
> `08289064594`, e **não** o `52998224725` da company 767. O contrato sai `SAL-08289064594-…`.
> Ponto em aberto com o time.

### 2.4 Roles no Valkey

`await user_authenticated.load_roles()` → `GET user:role:{user_id}:web` (JSON `UserRoleCacheDTO`).
**Não há fallback para o banco**: chave ausente = usuário sem roles.

`has_policy_by_account_id(account_id=mirror.id, is_account_mirror=True)`: procura uma role com
`company_id` do token **e** `account_mirror_id == mirror.id` **e** que contenha o scope da rota.
Falha → **404 Account not found** (404, não 403).

### 2.5 Documento vazio → 404 `No valid document for user scope`

Sai daqui `document` (11 ou 14 dígitos).

---

## Etapa 3 — Calcular o saldo (as duas rotas)

`calculate_available_balance(document)` — roda numa transação `ANTICIPATION_WRITE` aberta pela rota
(também no GET; ali ela fecha sem escrever nada).

| # | Passo | Tabela / fonte | Erro |
|---|---|---|---|
| 3.1 | `today = get_today_brazil()` (UTC−3) | — | — |
| 3.2 | `is_valid_cpf(document)` | — | **400** `Salary anticipation is only available to individuals` |
| 3.3 | Vínculo ativo: `SELECT * FROM payroll_employee WHERE document = :cpf AND status = 'ACTIVE' ORDER BY admission_date DESC, id DESC LIMIT 1` (com mais de um empregador vence a admissão mais recente) | `payroll_employee` | **404** `Employee not found for authenticated user` |
| 3.4 | `gross_salary_amount > 0` | `payroll_employee.gross_salary_amount` | **400** `Employee gross salary is not registered` |
| 3.5 | `SELECT * FROM payroll_employer WHERE id = :employer_id`; precisa existir **e** `is_active = 1` | `payroll_employer` | **400** `Employer is not active` |
| 3.6 | Período de referência (ver abaixo) | `payroll_employer.payment_day`, `payment_day_type` | — |
| 3.7 | `termination_date` nula **ou** posterior ao vencimento do período | `payroll_employee.termination_date` | **400** `Employee termination date is not after the period due date` |
| 3.8 | `SELECT value FROM parameter_default WHERE key = 'payroll_advance_rate'`; precisa existir e ser > 0 | `parameter_default` | 500 `ServiceError` se a linha não existe; **400** `Advance rate … is not configured` se ≤ 0 |
| 3.9 | `monthly_limit = bruto × taxa / 100` (taxa em **%**: `30` = 30%) | — | — |
| 3.10 | Antecipado no período (query abaixo) | `contract_creditagreement` | — |
| 3.11 | `available = max(0, monthly_limit − anticipated)` | — | — |

### Query do antecipado (3.10)

```sql
SELECT COALESCE(SUM(total_amount), 0)
FROM contract_creditagreement
WHERE client_document  = :cpf
  AND credit_product   = 'ANTICIPATION_SALARY'
  AND maturity_date    BETWEEN :period_start AND :period_end
  AND agreement_status IN ('PENDING_RELEASE','RELEASE_IN_PROGRESS','ANTICIPATED','LIQUIDATED','OVERDUE');
```

Tudo que **não** foi `CANCELLED`/`REJECTED` consome limite — inclusive pedidos ainda pendentes de
aprovação no backoffice.

### Período de referência (`resolve_reference_period`)

- `payment_day` nulo → mês-calendário (dia 1 até o último dia do mês de hoje).
- Senão: `due` = data de pagamento **deste mês** — `payment_day` corrido (limitado ao último dia do
  mês) ou o N-ésimo dia útil se `payment_day_type = BUSINESS_DAY`. Se `due < hoje`, usa o mês seguinte.
- `start` = (pagamento do mês anterior) + 1 dia; `end` = `due` (= `maturity_date` do contrato).

Exemplos com hoje = 30/09/2026:

| `payment_day` | pagamento do mês | período |
|---|---|---|
| 3 (nonprod) | 03/09 já passou → 03/10 | 04/09 → 03/10 |
| 5 (local) | 05/09 já passou → 05/10 | 06/09 → 05/10 |
| 30 | 30/09 = hoje (não é `<`) | 31/08 → 30/09 |

**O GET termina aqui** e responde:

```json
{"item": {"gross_salary_amount": 12345.0, "anticipated_amount": 0.0, "available_amount": 3703.5}}
```

---

## Etapa 4 — Montar o contrato (só POST)

`build_salary_advance_contract(document, balance, requested_amount, metadata_info)`

| # | Passo | Fonte | Erro |
|---|---|---|---|
| 4.1 | `amount > 0` e `amount ≤ available_amount` | balance | **400** `Requested amount must be greater than zero` / `Requested amount exceeds available balance` |
| 4.2 | `SELECT value FROM parameter_default WHERE key = 'payroll_service_fee'`; ≥ 0 | `parameter_default` | 500 se não existe; 400 se < 0 |
| 4.3 | `accrued_days = period.end − today` (aqui `date.today()` do servidor, não `get_today_brazil`); > 0 | — | **400** `Salary payment date must be in the future` |
| 4.4 | `service_amount = amount × (fee / 100 / 30) × accrued_days` — taxa **% ao mês, pro rata dia**. Ex.: 999 × 2%/30 × 5 = 3,33 | — | — |
| 4.5 | `disbursed_amount = amount − service_amount`; > 0 | — | **400** `Service fee is not lower than the requested amount` |
| 4.6 | Instancia os três objetos abaixo (ainda não persistidos) | — | — |

### Objetos criados

**`contract_creditagreement`** (`CreditAgreement`) — o contrato

| Coluna | Valor |
|---|---|
| `id` | uuid4 |
| `agreement_number` | `SAL-{cpf}-{yyyymmddHHMMSS}` |
| `client_document` | CPF do titular |
| `total_amount`, `principal_amount`, `outstanding_balance` | `amount` |
| `service_fee`, `service_amount`, `disbursed_amount` | da etapa 4 |
| `annual/daily/monthly_interest_rate` | 0 |
| `credit_product` | `ANTICIPATION_SALARY` |
| `capitalization_method` | `DAILY` |
| `term_in_days` | `accrued_days` |
| `origination_date`, `disbursement_date` | hoje |
| `maturity_date` | `period.end` |
| `agreement_status` | `PENDING_RELEASE` |
| `metadata_info` | JSON com IP / user-agent da request |

**`contract_loanposition`** (`LoanPosition`) — o item (1 por contrato)

| Coluna | Valor |
|---|---|
| `credit_agreement_id` | id do contrato |
| `nfse_id`, `nfse_number` | `NULL` — é o que diferencia de antecipação de NFSe |
| `payer_document` | **CNPJ do empregador** (`payroll_employer.document`) |
| `payee_document`, `client_document` | CPF |
| `principal`, `principal_outstanding`, `outstanding_balance` | `amount` |
| `service_fee`, `service_amount`, `disbursed_amount`, `accrued_days` | da etapa 4 |
| `maturity_date` | `period.end` |
| `last_accrual_date` | hoje |
| `position_status` | `PENDING_RELEASE` |

**`contract_salaryadvance`** (`SalaryAdvance`) — snapshot da folha usado no cálculo (auditoria)

| Coluna | Valor |
|---|---|
| `loan_position_id` | id do item |
| `employee_id` | `payroll_employee.id` |
| `gross_salary_amount`, `advance_rate`, `monthly_limit` | da etapa 3 |
| `anticipated_amount`, `available_amount` | da etapa 3 (**antes** deste pedido) |
| `reference_period_start`, `salary_payment_date` | `period.start`, `period.end` |

---

## Etapa 5 — Persistir ou não (só POST)

- **`dryRun=true`**: não grava; a transação fecha sem INSERT. Os `agreement_id`/`position.id` da
  resposta são uuids gerados e descartados — não existem no banco.
- **Sem `dryRun`**: `Repository.add(agreement)` + `flush` (o relacionamento `loan_position=[position]`
  insere o `contract_loanposition` em cascata), depois `Repository.add(snapshot)` + `flush`.
  O `begin()` da rota faz o **commit** ao sair do `with`; qualquer exceção → rollback dos três.
- Não há SQS, e-mail nem cashout aqui: o item nasce `PENDING_RELEASE` e a liberação é pelo
  backoffice, fora deste endpoint.

### Resposta

```json
{
  "worked": true,
  "message": "Salary advance registered successfully. Awaiting backoffice approval.",
  "data": {
    "agreement_id": "...", "agreement_number": "SAL-08289064594-20260930193545",
    "agreement_status": "PENDING_RELEASE",
    "position": {"principal": 999.0, "service_fee": 2.0, "service_amount": 3.33,
                 "disbursed_amount": 995.67, "accrued_days": 5,
                 "maturity_date": "2026-10-05", "position_status": "PENDING_RELEASE"},
    "balance": {"gross_salary_amount": 12345.0, "anticipated_amount": 0.0, "available_amount": 3703.5}
  }
}
```

`balance` na gravação é o saldo **antes** deste pedido (calculado na etapa 3, antes do INSERT); o
`contract_salaryadvance` gravado tem os mesmos números. Só na próxima chamada do GET o
`anticipated_amount` inclui este contrato (via a soma da etapa 3.10, porque `PENDING_RELEASE`
consome limite).

---

## Mapa rápido de erros

| HTTP | Mensagem | Onde falhou |
|---|---|---|
| 401 | `Not authenticated` | JWT: chave, expiração, `aud`, `token_type` |
| 403 | `Missing scopes` | policy ausente no role / bitmap `pm` |
| 404 | `Account not found` | `core_userrole` × `finance_accountbankmirror` (agência/conta/company) **ou** roles no Valkey sem o mirror |
| 404 | `Employee not found for authenticated user` | `payroll_employee` sem `ACTIVE` para o CPF do dono do bolsão |
| 400 | `Salary anticipation is only available to individuals` | documento do bolsão é CNPJ |
| 400 | `Employee gross salary is not registered` | `payroll_employee.gross_salary_amount` |
| 400 | `Employer is not active` | `payroll_employer.is_active` |
| 400 | `Employee termination date is not after the period due date` | `payroll_employee.termination_date` |
| 400 | `Requested amount exceeds available balance` | soma em `contract_creditagreement` no período |
| 400 | `Salary payment date must be in the future` | `period.end` ≤ hoje |
| 400 | `Service fee is not lower than the requested amount` | tarifa ≥ valor |
| 405 | `Method Not Allowed` | path antigo (`/payroll/advances` sem `/accounts/...`) |
| 500 | `Default parameter '…' not configured` | `parameter_default` sem `payroll_advance_rate` / `payroll_service_fee` |

## Como conferir no banco

```sql
-- ANTICIPATION
SELECT ca.agreement_number, ca.agreement_status, ca.client_document, ca.maturity_date,
       lp.payer_document, lp.principal, lp.service_amount, lp.disbursed_amount, lp.accrued_days,
       sa.monthly_limit, sa.anticipated_amount, sa.available_amount, sa.reference_period_start
FROM contract_creditagreement ca
JOIN contract_loanposition  lp ON lp.credit_agreement_id = ca.id
JOIN contract_salaryadvance sa ON sa.loan_position_id   = lp.id
WHERE ca.credit_product = 'ANTICIPATION_SALARY'
ORDER BY ca.created_at DESC;

-- Vínculo e empregador do CPF (use o doc_usado_pela_api da query do SAQ abaixo, sem máscara).
-- A API pega a linha ACTIVE com admissão mais recente.
SELECT e.document, e.status, e.admission_date, e.termination_date, e.gross_salary_amount,
       r.document AS cnpj_empregador, r.name, r.is_active, r.payment_day, r.payment_day_type
FROM payroll_employee e
JOIN payroll_employer r ON r.id = e.employer_id
WHERE e.document = '52998224725'
ORDER BY e.admission_date DESC, e.id DESC;

-- Parâmetros
SELECT * FROM parameter_default WHERE `key` LIKE 'payroll%';

-- SAQ (core): documento a partir de agência/conta. A API usa a company do BOLSÃO
-- (doc_usado_pela_api); se doc_company_mirror for diferente, é conta de whitelabel (ponto em aberto 1).
SELECT m.id AS mirror_id, m.agency, m.account,
       m.company_id AS mirror_company, mc.cnpj AS doc_company_mirror,
       b.id AS bolsao_id, b.document AS doc_bolsao,
       bc.id AS bolsao_company, bc.cnpj AS doc_usado_pela_api, bc.name
FROM finance_accountbankmirror m
LEFT JOIN core_company        mc ON mc.id = m.company_id
LEFT JOIN finance_accountbank b  ON b.id  = m.bolsao_id
LEFT JOIN core_company        bc ON bc.id = b.company_id
WHERE m.agency = '0001' AND m.account = '9001830';
```

## Pontos em aberto

1. **CPF do dono do bolsão** (etapa 2.3): em conta de whitelabel resolve a pessoa errada. A conta
   0001/9001830 do nonprod pertence à company 767 (CPF 52998224725, salário 2.500) mas o cálculo sai
   para o CPF 08289064594 (company 766, salário 12.345).
2. **`date.today()` × `get_today_brazil()`**: o GET/etapa 3 usa horário de Brasília; a etapa 4.3 usa a
   data do servidor (UTC no pod). Entre 21h e 0h (Brasília) o `accrued_days` do POST pode sair 1 dia
   menor que o implícito no período do GET.
3. `payroll_service_fee` está `2` no nonprod e `3` no `scripts/parameters.sql` — alinhar o valor oficial.
