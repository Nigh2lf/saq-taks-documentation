# Limite de chaves Pix ativas por conta

**Data:** 15/10/2026 · **Branch:** `feature/pix-key-active-limit` em 6 repositórios · **Estado:** 6 PRs mergeados em 27/10 (ver seção PRs) · **Guia do front:** `FRONTEND-LIMITE-CHAVES-PIX.md`
**Diagrama:** `FLUXO-LIMITE-CHAVES-PIX.drawio` (aba 1 arquitetura, aba 2 fluxo de decisão)

## Demanda

Cada conta ter no máximo X chaves Pix ativas, com um parâmetro global e um por conta.

## Contexto encontrado

- Hoje a SaQ não tem nenhum limite, nem global nem por conta. O único teto é o do Banco Central no DICT (5 PF / 20 PJ), aplicado fora da SaQ e só depois da chamada ao provedor.
- Cinco pontos inserem chave ativa em `finance_pixkeyaccountbank`: inclusão, alteração de vinculação e conclusão de reivindicação na dict-api-v3; cadastro pelo operador no saq-backoffice-api; lambda manage-pix-keys-event (fila). A única checagem existente era chave duplicada.
- Parâmetro global = `finance_globalparameters` (lido do Valkey por `get_global_parameter`, editado no backoffice). Parâmetro por conta = `AccountBankRule` (1:1 com espelho ou bolsão, PATCH `/v3/accounts/rules`).
- O padrão "valor da conta se existir, senão global; zero = sem limite" já existia para `max_amount_for_qr_code` / `pix_qrcode_max_amount`.

## Regra implementada

1. Limite da conta = `AccountBankRule.max_active_pix_keys`. Nulo → usa o global `pix_key_max_active_per_account`.
2. Zero em qualquer nível = sem limite (e não conta chaves).
3. Com limite: se `chaves ativas >= limite`, recusa **antes** de chamar o DICT.
4. Resposta: 400 em todos os pontos (padrão do projeto: 400 regra de negócio, 409 só duplicidade). Na fila, warning e mensagem consumida.

## O que mudou em cada repositório

| Repositório | Papel | Alteração |
|---|---|---|
| **SAQ-Backend** (Django, dono do schema) | Cria coluna e semeia o global | `finance/models.py`: campo `max_active_pix_keys` em `AccountBankRule`. Migration `0492` (AddField, após `0491_merge_20261008_1816`) e `0493` (seed de `pix_key_max_active_per_account` = "0", INTEGER, editável). |
| **saq-repository** 3.51.0 (SQLAlchemy) | Espelha a coluna e conta | `AccountBankRule.max_active_pix_keys`; `PixKeyAccountBankRepository.count_active_by_account(account_mirror_id= \| account_id=)`. |
| **saq-shared-libraries** 1.57.0 | Regra em um lugar só | `saq_utilities/helpers/pix_key_limit.py`: decisão pura (`resolve_max_active_pix_keys`, `ensure_pix_key_limit`, `PixKeyLimitExceededError` 400). `saq_cache_client.ensure_account_can_add_pix_key(conta)`: carrega regra, lê global, resolve e só conta se houver limite. Testes: 18 verdes. |
| **dict-api-v3** | Cliente | 1 linha em cada rota: `pix_key_include.py` (espelho), `pix_key_alter.py` (espelho de **destino**), `pix_key_claim_conclude.py` (espelho do claim). Erro sobe como `BusinessError` 400 pelo handler do saq-base-api. |
| **saq-backoffice-api** | Operador | `pix_key.py`: helper dentro de `try/except` → `AppError` 400 `errors.pix_key.active_limit_reached` (i18n pt-BR/en). Campo `max_active_pix_keys` no schema de request e resposta das regras da conta. Corrige bug de 05/2026: auditoria de chave no bolsão usava `account_mirror.account` (None) na criação e na exclusão. Teste `tests/test_pix_key_active_limit.py` (4 casos). |
| **lambda-aws-manage-pix-keys-event** | Fila | `process_pix_key_creation.py`: helper antes do DICT. `app.py`: captura `PixKeyLimitExceededError`, warning, mensagem consumida (sem retry/DLQ). |

Pins subidos nos 3 consumidores: `saq-repository==3.51.0`, `saq-*==1.57.0` (ainda não publicados).
A lambda `update-cache-event` entrou e saiu da feature (campo no cache sem leitor); a branch lá ficou vazia.

## Review (skill code-review, nível alto) e como ficou

| # | Achado | Decisão |
|---|---|---|
| 1 | Alteração de vinculação (`pix_key_alter.py`) inseria chave sem o check | Check na conta de destino, antes do DICT. O destino é sempre outra conta (mesmo bolsão já dá 400). |
| 2 | Auditoria de chave no bolsão quebrava com `account_mirror=None` (bug antigo, 05/2026) | Corrigido na criação e na exclusão, mesmo commit, com comentário. |
| 3 | Corrida entre `COUNT` e insert | Trava `FOR UPDATE` na linha da conta **descartada**: a linha carrega o saldo (`amount`) e ficaria presa durante a chamada ao DICT, concorrendo com o runner de saldo. Corrida aceita (pior caso: uma chave além do limite). |
| 4 | Exceção de negócio ia para a DLQ | `app.py` captura, loga warning e consome. |
| 5 | 400 no cliente × 409 no backoffice | 400 nos dois (padrão do projeto). |
| 6 | Regra copiada em 5 lugares | Helper único `ensure_account_can_add_pix_key`; cada ponto virou uma linha. |
| 7 | Campo no cache sem leitor | Removido (DTO, restore, lambda update-cache). Cache só carrega regra lida no caminho quente. |
| 8 | Ramo morto de bolsão na reivindicação | Sumiu com o helper (passa só o espelho do claim). |
| 9 | `COUNT` + `HGET` mesmo sem limite | Helper resolve o limite primeiro e retorna sem contar. |

## Decisões assumidas (confirmar com TL / P.O.)

- Limite por conta espelho ou bolsão, conforme onde a chave está vinculada. Não por empresa.
- Nulo na conta usa o global; zero em qualquer nível = sem limite.
- Reivindicação concluída e alteração de vinculação contam como chave nova na conta.
- Seed do global em zero: o deploy não muda o comportamento de ninguém; a operação liga depois.
- Corrida contagem × insert aceita (sem lock no banco).

## Como testar

- Shared: `cd shared/saq-shared-libraries && PYTHONPATH=packages/saq-utilities:packages/saq-cache-client:../saq-repository <venv>/bin/python -m pytest packages/saq-utilities/tests/test_pix_key_limit.py packages/saq-cache-client/tests/test_ensure_account_can_add_pix_key.py` (usado o venv da anticipation-backoffice-api).
- Backoffice: `tests/test_pix_key_active_limit.py` escrito, **não roda local** (falta `qas_ted_cashin`, só no CodeArtifact). Rodar no CI.
- dict-api-v3 e lambda: sem suíte de rotas; só compilação.
- Ponta a ponta em nonprod: subir o global para 1, cadastrar 1 chave, tentar a 2ª (400); definir `max_active_pix_keys=2` na conta pelo PATCH de regras e repetir.

## PRs

| Ordem | Repositório | PR |
|---|---|---|
| 1 | SAQ-Backend | https://github.com/SaQDigital/SAQ-Backend/pull/255 |
| 2 | saq-repository | https://github.com/SaQDigital/saq-repository/pull/459 |
| 3 | saq-shared-libraries | https://github.com/SaQDigital/saq-shared-libraries/pull/100 |
| 4 | dict-api-v3 | https://github.com/SaQDigital/dict-api-v3/pull/31 |
| 4 | saq-backoffice-api | https://github.com/SaQDigital/saq-backoffice-api/pull/545 |
| 4 | lambda-aws-manage-pix-keys-event | https://github.com/SaQDigital/lambda-aws-manage-pix-keys-event/pull/15 |

## Ordem de merge e deploy

1. SAQ-Backend (migrations 0492/0493) — a coluna precisa existir antes de qualquer API subir com o repository 3.51.0.
2. saq-repository → publicar 3.51.0.
3. saq-shared-libraries → publicar 1.57.0 (cache-client exige repository ≥ 3.48).
4. dict-api-v3, saq-backoffice-api, lambda manage-pix-keys-event (pins só instalam depois de 2 e 3; alternativa: pinar `.devN` da branch).

Atenção: se outra migration `0492` entrar na main antes, renumerar as duas.
