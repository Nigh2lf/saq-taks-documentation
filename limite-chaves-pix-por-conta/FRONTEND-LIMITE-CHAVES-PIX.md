# Limite de chaves Pix ativas por conta — guia para o frontend do backoffice

**Data:** 27/10/2026 · **API:** `saq-backoffice-api` (PRs mergeados em 27/10) · **Contato:** Rodrigo Lima

## O que é

Cada conta passa a ter um teto de chaves Pix **ativas**. O teto é resolvido assim:

1. Se a conta tem um limite próprio (`max_active_pix_keys` na regra da conta), vale ele.
2. Se a conta não tem (`null`), vale o parâmetro global `pix_key_max_active_per_account`.
3. **Zero**, em qualquer um dos dois níveis, significa **sem limite**.

O backend recusa a criação de uma chave quando a conta já tem `ativas >= limite`. A recusa acontece em todos os caminhos (app do cliente, backoffice e fila), sempre com HTTP 400. O front não precisa contar chaves nem validar o limite: só configurar os dois valores e tratar o erro 400.

O parâmetro global nasce com `"0"` (sem limite). Enquanto a operação não mudar isso, nada é bloqueado.

Referência do Banco Central (tetos do DICT, aplicados fora da SaQ): 5 chaves para pessoa física, 20 para pessoa jurídica. Se a operação quiser replicar isso, hoje o caminho é global = 5 e limite por conta = 20 nas contas PJ. Não existe parâmetro separado por tipo de pessoa.

Todas as rotas abaixo exigem o JWT do backoffice (`Authorization: Bearer`).

---

## 1. Parâmetro global

Prefixo: `/v3/parameters`. Só o usuário da IP (`whitelabel_default` no JWT) pode usar essas rotas; usuário BaaS recebe 403.

| Operação | Método e rota | Scope |
|---|---|---|
| Listar parâmetros (paginação por cursor) | `GET /v3/parameters/` | `global-parameter:read` |
| Consultar um parâmetro | `GET /v3/parameters/pix_key_max_active_per_account` | `global-parameter:read` |
| Alterar o valor | `PUT /v3/parameters/pix_key_max_active_per_account` | `global-parameter:update` |

### Consulta

`GET /v3/parameters/pix_key_max_active_per_account`

```json
{
  "key": "pix_key_max_active_per_account",
  "value": "0",
  "description": "Máximo de chaves Pix ativas por conta sem limite próprio. 0 = sem limite.",
  "readonly": false,
  "value_type": "INTEGER",
  "allowed_values": null,
  "created_at": "2026-10-27T18:30:00",
  "updated_at": "2026-10-27T18:30:00"
}
```

### Alteração

`PUT /v3/parameters/pix_key_max_active_per_account`

```json
{ "value": "5" }
```

- `value` é **string**, mesmo sendo inteiro. Tem que casar com `-?\d+`; o backend devolve 400 `errors.global_parameter.invalid_integer` se não for inteiro.
- Pode mandar só `value`, só `description`, ou os dois. Corpo vazio devolve 400 `errors.global_parameter.nothing_to_update`. Qualquer outro campo é rejeitado (schema fechado).
- `"0"` desliga o limite global.
- A resposta é o mesmo objeto da consulta, já com o valor novo. O cache é atualizado na hora; não há atraso para o limite passar a valer.

Erros possíveis: 404 `errors.global_parameter.not_found`, 400 `errors.global_parameter.readonly` (não é o caso deste parâmetro, que é editável), 403 para usuário BaaS ou sem o scope.

Sugestão de tela: campo numérico inteiro, mínimo 0, com a dica "0 = sem limite", convertendo para string no envio.

---

## 2. Limite por conta

Prefixo: `/v3/accounts`. O campo entra na regra da conta (`AccountBankRule`), ao lado das regras já existentes (`block_cashout`, `max_amount_for_qr_code` etc.). Vale tanto para conta bolsão (`is_mirror=false`) quanto para conta espelho (`is_mirror=true`): o limite é da conta onde a chave fica vinculada.

| Operação | Método e rota | Scope |
|---|---|---|
| Consultar regras | `GET /v3/accounts/rules/{account_id}?mirror=true\|false` | `account:read` |
| Alterar regras (update parcial) | `PATCH /v3/accounts/rules` | `account:update-rule` |

### Consulta

`GET /v3/accounts/rules/1234?mirror=true`

```json
{
  "worked": true,
  "is_mirror": true,
  "account_id": 1234,
  "rule": {
    "block_cashin": false,
    "block_cashout": false,
    "max_amount_for_qr_code": null,
    "document_required_amount_qrcode": null,
    "max_active_pix_keys": null,
    "qr_code_random_amount_enabled": false,
    "created_at": "2026-05-02T10:00:00",
    "updated_at": "2026-10-27T18:30:00"
  },
  "qr_code_random_amount_feature_enabled": false
}
```

(Os demais campos de `rule` foram omitidos aqui; a resposta real traz todos.)

Leitura de `max_active_pix_keys`:

| Valor | Significado na tela |
|---|---|
| `null` | "Usa o global" (mostrar o valor global ao lado, obtido na rota da seção 1, ajuda o operador) |
| `0` | "Sem limite" nesta conta, mesmo que o global tenha limite |
| `n > 0` | Limite próprio de `n` chaves ativas |

### Alteração

`PATCH /v3/accounts/rules`

```json
{
  "is_mirror": true,
  "account_id": 1234,
  "max_active_pix_keys": 20
}
```

- `is_mirror` e `account_id` são obrigatórios; os campos de regra são todos opcionais (update parcial). Só o que for enviado muda.
- `max_active_pix_keys` é **inteiro** (não string), `>= 0`. Para voltar a usar o global, enviar `null` explicitamente.
- A resposta tem o mesmo formato da consulta, com a regra atualizada.
- Se a conta ainda não tinha regra, o PATCH cria uma.

Erros possíveis: 400 de validação (inteiro negativo, `account_id <= 0`; o handler do projeto usa 400, não 422), 403 se o usuário não tem acesso à conta ou falta o scope, 404 conta inexistente.

Sugestão de tela: na aba de regras da conta, um campo inteiro com três estados: "Usar global (n)", "Sem limite" e "Limite próprio: [ ]". `null` ↔ usar global, `0` ↔ sem limite.

---

## 3. O erro ao cadastrar uma chave pelo backoffice

`POST /v3/pix/keys` (cadastro de chave pelo operador) passa a poder devolver **400** quando a conta atingiu o limite:

```json
{
  "worked": false,
  "detail": "A conta já tem 5 chaves Pix ativas; o limite é 5",
  "message": "A conta já tem 5 chaves Pix ativas; o limite é 5",
  "data": null,
  "code": "errors.pix_key.active_limit_reached"
}
```

- Identificar pelo `code`, não pelo texto. A mensagem vem traduzida pelo header `Accept-Language` (pt-BR e en).
- A verificação acontece **antes** de chamar o DICT; nada foi criado quando esse erro volta.
- O mesmo limite vale no app do cliente (dict-api-v3) na inclusão, na alteração de vinculação e na reivindicação de chave, e na criação pela fila. Nesses caminhos o erro também é 400, mas não passa pelo backoffice.

---

## Resumo rápido

| Quero… | Chamada |
|---|---|
| Ver o limite global | `GET /v3/parameters/pix_key_max_active_per_account` |
| Mudar o limite global | `PUT /v3/parameters/pix_key_max_active_per_account` com `{"value": "5"}` |
| Ver o limite de uma conta | `GET /v3/accounts/rules/{id}?mirror=…` → `rule.max_active_pix_keys` |
| Dar limite próprio à conta | `PATCH /v3/accounts/rules` com `{"is_mirror": …, "account_id": …, "max_active_pix_keys": 20}` |
| Conta sem limite | mesmo PATCH com `0` |
| Conta volta ao global | mesmo PATCH com `null` |
| Tratar recusa no cadastro | 400 com `code = errors.pix_key.active_limit_reached` |
