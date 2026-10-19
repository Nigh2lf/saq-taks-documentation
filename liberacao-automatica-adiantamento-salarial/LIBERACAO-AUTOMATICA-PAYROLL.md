# Antecipação salarial — liberação automática

Regras fechadas com o TL em 06/10/2026 e o passo a passo para implementar. O diagrama correspondente é o `FLUXO-LIBERACAO-AUTOMATICA-PAYROLL.drawio`.

## 1. O que é

Hoje todo adiantamento salarial nasce `PENDING_RELEASE` e depende de um operador aprovar no backoffice; a aprovação só troca o status, nenhum dinheiro é movimentado. A liberação automática faz o pedido já sair pago, sem operador, quando a IP (o banco), o empregador e o funcionário estiverem habilitados. A aprovação manual passa a pagar de verdade, pelo mesmo caminho.

## 2. Regras definidas

### Flags

| Nível | Onde | Quem liga | Default |
|---|---|---|---|
| Global (IP / banco) | `parameter_default.payroll_automatic_release` | seed / DBA, como os outros parâmetros | `0` |
| Empregador | `payroll_employer.is_automatic_release` | backoffice, no CRUD de empregadores | `false` |
| Funcionário | `payroll_employee.is_automatic_release` | backoffice, no CRUD de funcionários | `false` |

- Liberação automática = **global E empregador E funcionário**, avaliadas **na hora do pedido** (`POST .../payroll/advances`).
- Não há validação no cadastro: pode-se ligar o funcionário com o empregador desligado; o efeito é só não liberar automático. Desligar o global para tudo na hora.
- Qualquer flag desligada = **fluxo manual** (`PENDING_RELEASE`), nunca bloqueio do pedido.
- Onboarding e planilha de importação de funcionários **sempre** entram com a flag desligada.
- Ligar as flags de empregador e funcionário usa os scopes de atualização do CRUD (`Employer.UPDATE` / `Employee.UPDATE`); sem scope específico.
- A decisão de cada pedido fica gravada na foto: `contract_salaryadvance.automatic_release` (`true` = nasceu `RELEASE_IN_PROGRESS`; `false` = nasceu `PENDING_RELEASE`).

### Conta de pagamento

- O funcionário escolhe a conta no pedido (agência e conta na URL). Essa conta é **gravada no contrato** nas colunas do PR #57 do repository: `client_account_mirror_id`, `client_account_agency`, `client_account_number`.
- A lambda **paga nessa conta**, não numa conta resolvida pelo CPF.

### Fila e lambda

- Fila nova `anticipation-salary-advance-release` + DLQ, separada da fila da nota. Motivo (TL): manutenção e evitar concorrência no futuro.
- Lambda nova `lambda-aws-anticipation-salary-advance-release`, no molde chalice da `lambda-aws-anticipation-credit-release`.
- Mensagem: `{"type": "ANTICIPATION_SALARY", "contract_id": "<id do contrato>"}`.
- Publicação sempre **depois do commit** (`BackgroundTasks`), para a lambda encontrar o contrato gravado.
- Produtores: `anticipation-api-v3` (pedido automático) e `anticipation-backoffice-api` (aprovação manual e reprocesso).

### Um único caminho para pagar

- Automático: contrato e item nascem `RELEASE_IN_PROGRESS` e a API publica na fila.
- Manual: backoffice muda `PENDING_RELEASE → RELEASE_IN_PROGRESS` e publica na fila.
- **Só a lambda marca `ANTICIPATED`.** O backoffice deixa de marcar `ANTICIPATED` direto.
- A lambda só age se o contrato está em `RELEASE_IN_PROGRESS` (idempotência na reentrega).

### Lançamentos (iguais aos da nota, sem intermediários)

| Movimento | De | Para | Valor |
|---|---|---|---|
| `ANTICIPATION_CLIENT_DEBIT` / `ANTICIPATION_CLIENT_CREDIT` | conta de liquidação (`ANTICIPATION_LIQUIDATION_*`) | conta do funcionário gravada no contrato | `principal` |
| `ANTICIPATION_FEE_DEBIT` / `ANTICIPATION_FEE_CREDIT` | conta do funcionário gravada no contrato | conta de recebimento (`ANTICIPATION_RECEIVEMENT_*`) | `service_amount` |

- Líquido na conta do funcionário = `principal − service_amount` (= `disbursed_amount`).
- Mesmas contas e mesmos tipos de transação da nota; nenhum tipo novo no catálogo.
- Cada lançamento é buscado antes de criar (padrão `_get_transaction` da lambda da nota), para reprocesso não pagar em dobro.

### Falha

- Qualquer erro no pagamento: lambda marca contrato e item em **`ERROR`** (status do PR #53 do `anticipation-repository`, já na branch). A lambda não tenta de novo sozinha.
- `ERROR` **continua consumindo o saldo do ciclo** e aparece em `pending_amount`, para o funcionário não pedir de novo enquanto o operador decide.
- O operador vê o contrato no backoffice e decide:
  - **reprocessar**: `ERROR → RELEASE_IN_PROGRESS` + publica na fila; a lambda retoma do lançamento que faltou;
  - **cancelar**: `ERROR → CANCELLED`, valor volta ao saldo. **Só é permitido se o contrato não tem lançamento no razão** (`finance_accountbanktransaction.transaction_id` = id do contrato). O razão não marca "executado": a lambda grava os lançamentos antes de executá-los, então qualquer linha é dinheiro em movimento ou a caminho; a rota responde 409 e só resta reprocessar.
- DLQ fica só como rede de segurança, para quando a lambda nem consegue gravar o `ERROR`.

### Fora de escopo por enquanto

- Notificação ao funcionário (sucesso ou falha).
- Rotina de split e baixa (`LIQUIDATED`) quando a empresa paga o salário; a baixa continua manual pelo backoffice.
- Estorno de lançamentos parciais (por isso o cancelamento é restrito).

## 3. Estado atual (06/10/2026)

| Repositório | Branch do adiantamento salarial | PR | Branch da liberação automática |
|---|---|---|---|
| `anticipation-repository` | `feature/payroll-anticipated-solicitation` | #62 (conflita com a `master`) | `feature/payroll-automatic-release`: contém a anterior + `ERROR` (#53 via merge) |
| `anticipation-api-v3` | `feature/payroll-anticipated-solicitation` | #59 (conflita com a `master`) | `feature/payroll-automatic-release`: contém a anterior + commit `0909e20` (flag global única, fila da nota) |
| `anticipation-backoffice-api` | `feature/payroll-anticipated-solicitation` | #47 (mesclável) | `feature/payroll-automatic-release`: **saiu da `master`**, não contém o adiantamento salarial; commit `29b4225` faz a rota de notas aceitar contrato salarial |

Os commits de 05/10 foram feitos antes das decisões acima e precisam de rework. Os PRs #62 (api-v3), #61 (repository) e #48 (backoffice) abertos a partir deles foram fechados.

PRs de terceiros que viram pré-requisito: **#57** (repository) e **#54** (api-v3), que gravam a conta do cliente no contrato, e a branch `feature/anticipation-credit-release-account-v1` da lambda da nota, de onde vem `_resolve_client_account`.

## 3b. Andamento (06/10/2026, fim do dia)

Passos 2, 3 e 4 feitos no working tree e **em stage, sem commit**, para revisão:

| Repositório | Branch | Estado |
|---|---|---|
| `anticipation-repository` | `feature/payroll-automatic-release` | merge do #57 parado antes do commit (`VERSION` resolvido em 1.28.0); colunas e migration `payroll_0005_automatic_release` em stage. Heads do Alembic: main (`4f96a9a26c26`), `9099c5aaa5e4` (#57) e folha |
| `anticipation-api-v3` | `feature/payroll-automatic-release` | merge do #54 parado antes do commit (conflitos resolvidos: `user_scope.py` ficou com os dois resolvedores, `requirements.txt` manteve os pins `.dev`); passo 4 completo em stage. Suíte: 277 passaram; 3 falhas em `test_get_payment_received` vêm do venv apontar para o repository local, que ainda não tem o `recebimentoV2` da `master` |
| `anticipation-backoffice-api` | `feature/payroll-automatic-release` | branch recriada a partir de `feature/payroll-anticipated-solicitation` (o remoto ainda tem o commit antigo `29b4225`; precisa de push forçado); passo 3 completo em stage. Suíte: 378 passaram, rodada com um worktree temporário do repository que juntava a branch de merge do nonprod + `ERROR` + as colunas novas |
| `lambda-aws-anticipation-salary-advance-release` | `master` (sem commits) | repositório criado e clonado em `anticipation-apps/lambdas/`; passo 5 escrito e **em stage**: molde da `credit-release` (workflows, `initialization.py`, `.gitignore` etc.), `app.py`, `chalicelib/helpers/salary_advance_transaction.py`, `test.py`, README e 9 testes unitários (passam com o venv da api-v3; a lambda não tem venv próprio). Ordem register → execute → confirm → `ANTICIPATED`, falha → `ERROR`. Pins de `requirements.txt` (saq-* 1.47.1, `anticipation-repository` 1.28.0) a confirmar no publish. Fila, DLQ, roles, chave no `apps-configs` e environments/secrets do repositório ainda pendentes (passo 1) |

**Code review (06/10, `/code-review high` nos quatro repositórios)** — correções aplicadas, todas em stage:

- api-v3: fila declarada no `Settings` (falta no boot, não no `BackgroundTask`); `is_automatic_release_enabled` também cai no manual se o valor da chave global não for booleano; `resolve_account_scope_document` virou atalho de `resolve_user_scope_account`, para o `GET` de saldo e o `POST` de adiantamento aplicarem o mesmo gate de conta.
- backoffice: fila no `Settings`; checagem do razão feita **com o contrato travado** e só para contrato em `ERROR` (as sessões empilham por contextvar, então a sessão do core abre dentro da de escrita); `RELEASE_IN_PROGRESS → RELEASE_IN_PROGRESS` passou a ser aceito (republicar contrato preso quando a publicação falhou depois do commit); ramo 422 inalcançável removido; testes diretos do formato da mensagem e do filtro do razão.
- lambda: **ordem voltou à da nota** (registrar → executar → `ANTICIPATED` → confirmar) — o cache só é idempotente enquanto a chave pendente existe e o confirm a apaga, então `ERROR` só é gravado para falha antes do `ANTICIPATED`; falha no confirm sobe para a DLQ sem `ERROR` (reprocessar pagaria de novo). `_update_contract` com lock e só a partir de `RELEASE_IN_PROGRESS`; conta do funcionário conferida contra agência/número gravados; tarifa zero não gera lançamento; flag de trace SQL corrigida; secret manager sem API depreciada e sem sobrescrever os `DB_*` do SAQ; `.env.example` com as variáveis reais. Recomendação de infra registrada no README: fila **FIFO** com `MessageGroupId = contract_id` contra entrega duplicada.
- repository: migration do #57 (`9099c5aaa5e4`) encadeada em `4f96a9a26c26` (ERROR) para a linha principal ter uma head só; ao levar #53 e #57 para a `master`, manter esse encadeamento.

Achados que **não** são deste trabalho (código do #54/#47, reportar aos autores): a rota nova `POST /accounts/{agência}/{conta}/contracts` do #54 está protegida só por `anticipation:nfse:read` (perfil de leitura consegue contratar); `post_contract_account_scoped.py` duplica ~80 linhas de `post_contract.py`; `resolve_legacy_user_scope` duplica `resolve_user_scope_document`; a lambda da nota em `master` ainda paga pelo CPF (a branch `feature/anticipation-credit-release-account-v1` corrige); o `development.yaml` do backoffice ainda tem o gatilho da branch de merge do nonprod (já no checklist do #47). Sugestão do revisor não aplicada: bump de `VERSION` do repository (1.28.0 já tem builds `.dev` publicados); decidir junto com a publicação.

**Conformidade com `saq-ai-workspace/.claude/rules/` (06/10):** infra, saq-ci, Python, testes e nomenclatura conferidos; workflows da lambda idênticos ao canônico. Desvios a decidir: (1) `migrations.md` — a `payroll_0005` foi escrita à mão (precedente da folha), conferir com `alembic revision --autogenerate`; (2) `git-flow.md` — repo novo desenvolve na `develop` com PR para a `master`; o primeiro push da lambda foi na `master` e disparou o workflow de produção (conferir em Actions); (3) `saq-ci` — branch local sem upstream, primeiro push com `git push -u origin <branch>`. Os 2 testes que falham no `saq-ci` (`test_crypto_apps_use_crypto_role_and_secret`, `test_only_stages_limits_jdpi_validate_account`) são pré-existentes.

Pontos para a revisão:

- **Dependência do backoffice**: o #47 usa os modelos da importação de funcionários (`EmployeeImport`, da branch `feature/payroll-employee-import` do repository), que não estão em `feature/payroll-automatic-release` do repository. O pin `1.28.0.dev73` vinha de uma branch de merge. Decidir de qual branch publicar o `.devN` que as duas APIs vão pinar (ou trazer `feature/payroll-employee-import` para a branch de liberação automática).
- `api-v3/main/helpers/user_scope.py` ficou com dois resolvedores por conta: `resolve_user_scope_account` (do #54, usado agora pelo `POST` do adiantamento) e `resolve_account_scope_document` (usado pelo `GET` de saldo). Unificar depois que o #54 entrar.
- Backoffice: a lambda de importação cria `Employee` sem a flag; o `server_default` da migration cobre, mas vale um teste quando a lambda for atualizada.

## 3c. Estado real (07/10/2026, manhã)

Na tarde de 06/10, depois da sessão, tudo o que estava em stage foi commitado e enviado: `feature/payroll-automatic-release` nos três repositórios (merges do #57/#54 e do #53 incluídos; backoffice recriado a partir de `feature/payroll-anticipated-solicitation`), lambda na `develop` (4 commits), `saq-infra` com a fila FIFO nos stacks `adopay` e `nonprod`, e `saq-ci` com o json da lambda na branch `feature/lambda-aws-anticipation-salary-advance-release`. `anticipation-repository` publicou `1.28.0.dev78` a partir da branch.

Pendências encontradas em 07/10:

- **Deploy da lambda falhou na `develop`** (run 37525978949): o Chalice monta `<app_name>-<stage>-<handler>` e `lambda-aws-anticipation-salary-advance-release-nonprod-handle-sqs` tem 65 caracteres (limite da AWS: 64). Correção no working tree: handler renomeado de `handle-sqs` para `sqs` em `app.py` (nonprod fica com 58, adopay com 57). Os runs anteriores na `master` falharam antes disso, em "Render Chalice config" (antes do ajuste da `ci_ref`).
- **Nenhum PR aberto** para `feature/payroll-automatic-release` (repository, backoffice, api-v3), nem para `saq-infra` e `saq-ci`.
- **Pins desalinhados**: lambda em `1.28.0.dev78` (da branch), api-v3 ainda em `1.28.0.dev66`, backoffice em `1.28.0.dev73` (da branch de merge do nonprod). O backoffice (`routes/employees_import.py`) importa `EmployeeImport`, que não existe na branch de liberação automática do repository — continua a decisão de trazer `feature/payroll-employee-import` ou publicar de uma branch de merge.
- `origin/master` do repository avançou 9 commits (recebimentoV2 #56 mergeado, VERSION voltou para 1.26.0) e não está na branch; #62, #58 e o #59 da api-v3 constam como CONFLICTING no GitHub.
- Alembic na branch: heads `9099c5aaa5e4` (principal) e `payroll_0005_automatic_release` (folha), como planejado.


**Decisão de 07/10 (tarde): fila comum, sem FIFO.** Ajustes no working tree, sem commit: `saq-ci` (json sem `.fifo`), lambda (`.env.example`, README, lock por contrato `salary_advance:release:{id}` via `client.lock` do redis-py, token por dono, não bloqueante, TTL 90s; colisão com contrato ainda em `RELEASE_IN_PROGRESS` levanta `ReleaseInProgressError`, com contrato já concluído é no-op; falha ao soltar é só log; 21 testes). Code review `high` rodado em 07/10 e achados aplicados; recomendação não aplicada: levar o lock para um helper compartilhado e usar também na `credit-release`, que tem a mesma janela), `anticipation-api-v3` e `anticipation-backoffice-api` (envio sem `message_group_id`, testes e `.env.example`). `saq-infra`: edição do usuário no editor (no disco ainda FIFO às 16h).

## 3d. Conflitos de `feature/payroll-automatic-release` com a `master` (07/10/2026)

Simulação feita em worktrees temporários (`git merge --no-commit origin/master`), sem tocar nas branches. Resultado por repositório:

| Repositório | `master` à frente | Conflitos | Resolução |
|---|---|---|---|
| `anticipation-backoffice-api` | 0 commits | **nenhum** | PR #50 está `MERGEABLE`; o `BLOCKED` do GitHub é só review/checks pendentes |
| `anticipation-api-v3` | 1 commit (`bbf0d69`, disparo manual nos workflows) | `.github/workflows/development.yaml` | ficar com a `master`: `branches: ["develop"]` + `workflow_dispatch:` (tirar `feature/payroll-available-balance` do gatilho, que já estava no checklist) |
| `anticipation-repository` | 9 commits (recebimento V2 #56, split pending, `VERSION` 1.26.0) | `VERSION`, `models/__init__.py`, `repositories/__init__.py` | ver abaixo |

### `anticipation-repository` — os três conflitos

Todos são "os dois lados acrescentaram no mesmo ponto"; a resolução é manter os dois lados:

1. **`VERSION`**: branch `1.28.0` × master `1.26.0` (o commit `69af02f` de 05/10 baixou de 1.27.0 para 1.26.0 e a tag `v1.26.0` existe). Ficar com **`1.28.0`**: é a versão que os pins `.devN` das APIs e da lambda já usam, e 1.27.0 nunca virou tag.
2. **`anticipation_repository/models/__init__.py`** (linha ~95, lista `__all__`): manter `Employer`, `Employee`, `PaymentDayTypeEnum`, `EmployeeStatusEnum` (branch) **e** `PaymentSplitPendingModel`, `PaymentSplitPendingStatusEnum`, `PaymentSplitPendingLegKindEnum` (master).
3. **`anticipation_repository/repositories/__init__.py`** (import na linha ~14 e `__all__` na ~34): manter `SalaryAdvanceRepository` (branch) **e** `PaymentSplitPendingRepository` (master), nos dois lugares.

Fora dos conflitos, o merge traz 10 arquivos do recebimento V2 sem intervenção (736 linhas).

### O que o merge **não** acusa: terceira head do Alembic

`master` e branch partem da mesma revisão `128e63828684` e cada uma seguiu por um caminho:

```
128e63828684 ─┬─ f2d0beda912f ─ a9b1c2d3e4f5          (master: agência/conta no recebimento, payment_splitpending)
              └─ 4f96a9a26c26 ─ 9099c5aaa5e4          (branch: ERROR/RELEASE_IN_PROGRESS no enum, client_account_* no contrato)
c1a4e7b30f95 ─ a7c3e1b40d92 ─ e9b4d2a17c56 ─ payroll_0004 ─ payroll_0005   (head `payroll`, independente, ok)
```

Depois do merge a linha principal fica com **duas heads** (`a9b1c2d3e4f5` e `9099c5aaa5e4`) além da `payroll`: `alembic upgrade head` falha com "Multiple head revisions" e `alembic upgrade heads` passa a ser obrigatório. A regra do workspace é uma head só na linha principal. Correção ao resolver o merge: em `4f96a9a26c26_add_error_and_release_in_progress_to_.py` trocar `down_revision = '128e63828684'` por **`'a9b1c2d3e4f5'`** (o ERROR passa a vir depois do split pending; `9099c5aaa5e4` continua depois dele). Conferir com `alembic heads`: esperado `9099c5aaa5e4` e `payroll_0005_automatic_release`.

**Antes de re-encadear, conferir o banco de dev** (a VPN caiu durante a sessão e a consulta não rodou):

```sql
SELECT * FROM alembic_version;
SELECT TABLE_NAME, COLUMN_NAME FROM INFORMATION_SCHEMA.COLUMNS
 WHERE TABLE_SCHEMA='ANTICIPATION'
   AND ((TABLE_NAME='payment_splitpending' AND COLUMN_NAME='id')
     OR (TABLE_NAME='contract_creditagreement' AND COLUMN_NAME LIKE 'client_account%')
     OR (TABLE_NAME='payroll_employer' AND COLUMN_NAME='is_automatic_release'));
```

Se `9099c5aaa5e4` já estiver em `alembic_version` **sem** `a9b1c2d3e4f5` ter sido aplicada (sem a tabela `payment_splitpending`), o re-encadeamento faria o Alembic considerar `a9b1c2d3e4f5` como já aplicada e pular a tabela: nesse caso aplicar `a9b1c2d3e4f5` à mão antes (`alembic upgrade a9b1c2d3e4f5` ainda com as duas heads) e só então re-encadear. Se o banco estiver em `a9b1c2d3e4f5` + `d4e8b2c17f39` (estado visto em 29/09), o re-encadeamento é seguro e o `upgrade heads` aplica `4f96a9a26c26` → `9099c5aaa5e4` e a folha.

### Dependência cruzada que continua

O backoffice (#50) importa `EmployeeImport`, que está em `feature/payroll-employee-import` (#59 do repository) e **não** em `feature/payroll-automatic-release`. O merge entre essas duas branches do repository só conflita em `VERSION` (testado em 07/10 num worktree). Ordem sugerida no repository: resolver os conflitos acima com a `master` → trazer `feature/payroll-employee-import` → publicar um `.devN` que as duas APIs e a lambda pinam → PR para a `master` com `VERSION 1.28.0` e tag `v1.28.0` no merge.


**PRs abertos em 07/10/2026:** `saq-infra` #82 (fila + roles), `saq-ci` #19 (json da lambda; descrição escrita hoje), `anticipation-repository` #64 (inclui #62/#53/#57; CONFLICTING com a master, ver 3d), `anticipation-backoffice-api` #50 (inclui #47/#43; MERGEABLE), `anticipation-api-v3` #64 (inclui #59/#54; conflita só no `development.yaml`). Lambda segue na `develop` sem PR (deploy em nonprod depende da fila).

## 4. Passo a passo

A ordem segue a dependência entre os repositórios. Os passos 2 a 4 não dependem da infra; só a lambda e o teste ponta a ponta dependem.

### Passo 1 — Infra (pedir primeiro: é o que tem mais espera)

Pedido à equipe de infra, em nonprod e adopay:

1. SQS comum `anticipation-salary-advance-release` + DLQ via `saq-infra` (`app-eks:sqs` no `Pulumi.<tenant>.yaml`, `fifoQueue: false`, `maxReceiveCount: 3`, `visibilityTimeoutSeconds: 360`), molde da `credit-release`. **Decisão de 07/10 (usuário): sem FIFO**, para ficar no padrão das demais filas de dinheiro da plataforma; a deduplicação FIFO não estava ativa de fato (id de deduplicação era uuid por envio) e a entrega concorrente do mesmo contrato passou a ser tratada por lock por contrato no Valkey, na lambda. PR **#82** do `saq-infra` aberto em 07/10 ainda com a versão FIFO; os YAML foram ajustados pelo usuário no working tree e o corpo do PR precisa ser atualizado junto com o push.
2. Roles: `lambdas-anticipation` com consumo na fila; `eks-microservicos-anticipation` com publicação — ARNs `anticipation-salary-advance-release*` adicionados nas duas, no mesmo diff.
2b. Registro da lambda no `saq-ci`: `lambdas/config/lambda-aws-anticipation-salary-advance-release.json` (role `anticipation`, secrets `rds_app_user`/`rds_anticipation`/`apps_configs`, variável da fila), **preparado** na branch local `feature/lambda-aws-anticipation-salary-advance-release` do `saq-ci`. O estado de deploy fica em `lambdas/deployed/<app>/<stage>.json` no `saq-ci`; primeiro deploy não precisa de estado prévio.
3. ~~Repositório `SaQDigital/lambda-aws-anticipation-salary-advance-release`~~ — **criado em 06/10/2026** (https://github.com/SaQDigital/lambda-aws-anticipation-salary-advance-release), clonado em `anticipation-apps/lambdas/`, ainda vazio. Faltam os environments e secrets iguais aos da `lambda-aws-anticipation-credit-release` (`develop` e produção), que o workflow `chalice-deploy` do `saq-ci` usa.
4. Chave `SQS_ANTICIPATION_SALARY_ADVANCE_RELEASE` no `apps-configs`.

Pronto quando: fila visível no console, environments/secrets do repositório configurados, chave preenchida.

### Passo 2 — `anticipation-repository`

Branch: `feature/payroll-automatic-release` (já contém o adiantamento salarial e o `ERROR`).

1. Trazer o #57 (`feature/anticipation-credit-release-account-v1`) para a branch: colunas `client_account_*` em `contract_creditagreement`.
2. `Employer.is_automatic_release` e `Employee.is_automatic_release` em `models/payroll/payroll_models.py`: `Boolean`, `nullable=False`, `default=False`, `server_default="0"`, comentário em português como as demais colunas.
3. `SalaryAdvance.automatic_release` em `models/contract/contract_models.py`: mesmo formato.
4. Migration na head da folha (revisa `payroll_0004_salary_advance`): `add_column` nas três tabelas, com `server_default` para preencher as linhas existentes; `downgrade` remove.
5. Atualizar `NOMENCLATURA-BANCO.md` se ele listar as colunas da folha.
6. `VERSION`: manter 1.28.0 enquanto o #62 não entra; conferir ao resolver o conflito.
7. Publicar `1.28.0.devN` no CodeArtifact para as APIs pinarem.

Pronto quando: `alembic upgrade heads` roda num banco com a folha já criada e as colunas aparecem com `0`.

### Passo 3 — `anticipation-backoffice-api`

Branch: **recriar** `feature/payroll-automatic-release` a partir de `feature/payroll-anticipated-solicitation` (a atual saiu da `master`). Do commit `29b4225`, aproveitar as traduções de `ERROR` e o `nfse_id` opcional no schema; **descartar** a mudança em `PATCH /v3/credit-agreements/{id}/status` (rota de notas não aceita contrato salarial).

1. CRUD de empregadores e funcionários (`routes/employers.py`, `routes/employees.py`, schemas): expor `is_automatic_release` na leitura e na atualização. Na criação e na importação por planilha, não aceitar o campo: entra sempre `false`.
2. `PATCH /v3/salary-advances/{id}/status` (`routes/update_salary_advance_status.py`):
   - `PENDING_RELEASE → RELEASE_IN_PROGRESS` (aprovação) e `ERROR → RELEASE_IN_PROGRESS` (reprocesso): atualiza contrato e posições e publica na fila nova **depois do commit** (`BackgroundTasks`, mesmo padrão de `update_agreement_status.py`).
   - `PENDING_RELEASE → CANCELLED`: só status.
   - `ERROR → CANCELLED`: antes, consultar os lançamentos do contrato (`finance_accountbanktransaction` via `saq-repository`); se houver algum executado, 409 e só reprocesso.
   - Remover a transição direta para `ANTICIPATED`. Manter `ANTICIPATED/OVERDUE → LIQUIDATED` (baixa manual) até a rotina de split existir.
3. Listagem `GET /v3/salary-advances`: `ERROR` precisa ser filtrável por `agreement_status`, para o operador achar o que falhou; detalhe expõe `automatic_release` e a conta de pagamento.
4. Env `SQS_ANTICIPATION_SALARY_ADVANCE_RELEASE` (`.env.example`, `tests/conftest.py`, README).
5. Testes: transições novas, publicação só após o commit, nenhuma publicação no cancelamento, 409 no cancelamento com lançamento executado, campo novo no CRUD, importação ignorando o campo.
6. `requirements.txt`: pinar o `anticipation-repository` publicado no passo 2.

Pronto quando: `pytest tests/` verde e um contrato em `ERROR` pode ser reprocessado ou cancelado pela rota.

### Passo 4 — `anticipation-api-v3`

Branch: `feature/payroll-automatic-release` (já contém o adiantamento salarial). Rework do commit `0909e20`: a estrutura fica (parâmetro `automatic_release` no helper, `BackgroundTasks` após o commit, `ERROR` consumindo limite); muda a resolução da flag e a fila.

1. `payroll_advance_helper.is_automatic_release_enabled` passa a receber `employer` e `employee` e devolver `global and employer.is_automatic_release and employee.is_automatic_release`. Sem a chave global no banco, cai no manual (já é assim).
2. Renomear o parâmetro global para `payroll_automatic_release` (`AUTOMATIC_RELEASE_PARAMETER` e o `INSERT` em `scripts/parameters.sql`).
3. `calculate_available_balance` já carrega `employer` e `employee`; expor as flags em `PayrollAvailableBalance`.
4. `build_salary_advance_contract` grava `automatic_release` na foto e `client_account_mirror_id`, `client_account_agency`, `client_account_number` no contrato com a conta espelho da URL (já resolvida em `resolve_account_scope_document`). Depende do #54 estar na branch.
5. Fila: `send_anticipation_credit_release_message` volta a ser só da nota; criar `send_salary_advance_release_message(contract_id)` publicando em `SQS_ANTICIPATION_SALARY_ADVANCE_RELEASE` com `type = ANTICIPATION_SALARY`. Trocar a env em `.env.example`, `tests/conftest.py` e README.
6. Testes: as oito combinações das três flags (só `1/1/1` libera), chave global ausente, publicação após a resposta, `dryRun` não publica, `ERROR` em `pending_amount`, conta e decisão gravadas.
7. `requirements.txt`: pinar o `anticipation-repository` do passo 2.

Pronto quando: `pytest` verde e, em nonprod com as três flags ligadas, o pedido grava `RELEASE_IN_PROGRESS` com a conta e a mensagem aparece na fila.

### Passo 5 — `lambda-aws-anticipation-salary-advance-release`

Repositório novo (passo 1), copiando a estrutura da `lambda-aws-anticipation-credit-release` (`app.py`, `chalicelib/`, `.chalice/`, workflows `development.yaml` e `production.yaml`, `test.py`).

1. `app.py`: `on_sqs_message` em `SQS_ANTICIPATION_SALARY_ADVANCE_RELEASE`, `batch_size=1`; validar `type == ANTICIPATION_SALARY` e `contract_id`.
2. Carregar contrato (`credit_product == ANTICIPATION_SALARY`, `agreement_status == RELEASE_IN_PROGRESS`; qualquer outro status: log e sai sem erro), posição única e snapshot.
3. Resolver contas: liquidação e recebimento pelas envs `ANTICIPATION_LIQUIDATION_*` / `ANTICIPATION_RECEIVEMENT_*`; conta do funcionário pelas colunas `client_account_*` do contrato (`_resolve_client_account`, da branch `feature/anticipation-credit-release-account-v1` da lambda da nota). Contrato sem conta gravada: `ERROR`.
4. Registrar, executar e confirmar os quatro lançamentos (tabela da seção 2), reaproveitando `_get_transaction` / `_create_transaction` / `_execute_transaction` / `_confirm_transaction`. Sem intermediários, sem `NfseRepository`.
5. Sucesso: contrato e posição `ANTICIPATED`. Exceção: contrato e posição `ERROR`, log com a causa, mensagem consumida (não devolver à fila; o reprocesso é pelo backoffice). Se nem o `ERROR` puder ser gravado, deixar a exceção subir para a DLQ.
6. `.chalice/deployed/<stage>.json` vazio commitado antes do primeiro deploy (ver README da lambda da nota).
7. README nos moldes da `credit-release`: variáveis, pré-requisitos por ambiente, deploy, `test.py nonprod ANTICIPATION_SALARY <contract_id>`.

Pronto quando: em nonprod, um contrato `RELEASE_IN_PROGRESS` vira `ANTICIPATED` com os quatro lançamentos no razão, e um contrato forçado a falhar vira `ERROR`.

### Passo 6 — Ponta a ponta em nonprod

1. Seed: `payroll_automatic_release` em `parameter_default`, variáveis novas no `apps-configs`, migration aplicada.
2. Flags `0/0/0`: pedido nasce `PENDING_RELEASE`; aprovar no backoffice → fila → lambda → `ANTICIPATED` na conta escolhida.
3. Flags `1/1/1`: pedido nasce `RELEASE_IN_PROGRESS` → fila → lambda → `ANTICIPATED`, sem operador.
4. Flags `1/1/0` e `1/0/1`: pedido nasce `PENDING_RELEASE`.
5. Falha forçada (ex.: conta de liquidação sem saldo): `ERROR`; reprocessar pelo backoffice após corrigir → `ANTICIPATED`; cancelar outro sem lançamento → `CANCELLED` e saldo de volta; tentar cancelar um com lançamento executado → 409.
6. Reentrega da mesma mensagem: nenhum lançamento duplicado.

### Ordem dos PRs

1. `anticipation-repository`: `feature/payroll-automatic-release → feature/payroll-anticipated-solicitation` (ou direto para a `master` depois que o #62 e o #57 entrarem).
2. `anticipation-backoffice-api`: idem, base `feature/payroll-anticipated-solicitation` (ou `master` depois do #47).
3. `anticipation-api-v3`: idem, base `feature/payroll-anticipated-solicitation` (ou `master` depois do #59 e do #54).
4. Lambda: `develop` (nonprod) e depois `master` (adopay).

Trocar os pins `.devN` pelas versões finais do `anticipation-repository` antes de cada merge na `master`.
