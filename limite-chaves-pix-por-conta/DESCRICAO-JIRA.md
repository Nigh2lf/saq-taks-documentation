## Limite de chaves Pix ativas por conta

**Objetivo**

Permitir que a SaQ limite a quantidade de chaves Pix ativas por conta, com um parâmetro global e a possibilidade de sobrescrever conta a conta. Hoje não existe nenhum limite interno; o único teto é o do Banco Central no DICT (5 PF / 20 PJ), aplicado fora da SaQ e só depois da chamada ao provedor.

**Regra**

- Limite da conta = `AccountBankRule.max_active_pix_keys`. Nulo usa o parâmetro global `pix_key_max_active_per_account`.
- Zero em qualquer nível = sem limite.
- Com limite, a criação de chave é recusada **antes** de chamar o DICT quando a conta já tem o máximo de chaves ativas.
- Resposta: 400 para cliente e operador (regra de negócio); na fila, a mensagem é consumida com warning, sem DLQ.
- Seed do global em 0: o deploy não muda o comportamento de nenhuma conta. A operação liga depois pelo backoffice (parâmetros globais ou regras da conta).

**Onde a regra é aplicada (5 pontos que criam chave ativa)**

- dict-api-v3: inclusão de chave, alteração de vinculação (conta de destino) e conclusão de reivindicação.
- saq-backoffice-api: cadastro de chave pelo operador. Também expõe `max_active_pix_keys` no PATCH/GET de regras da conta.
- lambda-aws-manage-pix-keys-event: criação pela fila.

**Entregas (ordem de merge)**

1. SAQ-Backend #255: coluna e seed do parâmetro global (migrations 0492/0493).
2. saq-repository #459: modelo e contagem de chaves ativas por conta (3.51.0).
3. saq-shared-libraries #100: regra em um helper único `ensure_account_can_add_pix_key` (1.57.0).
4. dict-api-v3 #31, saq-backoffice-api #545, lambda-aws-manage-pix-keys-event #15: aplicação nos 5 pontos e pins.

Inclui correção de bug antigo no backoffice (05/2026): auditoria de chave vinculada direto ao bolsão quebrava com 500 na criação e na exclusão.

**Decisões a confirmar com P.O.**

- Limite por conta (espelho ou bolsão), não por empresa.
- Reivindicação concluída e alteração de vinculação contam como chave nova na conta.
- Corrida entre dois cadastros simultâneos na mesma conta aceita (sem lock no banco, para não travar a linha de saldo durante a chamada ao DICT).

**Como validar em nonprod**

1. Global em 1: cadastrar 1 chave e tentar a 2ª → 400 "limite atingido".
2. `max_active_pix_keys` = 2 na conta pelo PATCH de regras: a 2ª passa, a 3ª recusa.
3. Global em 0 e conta nula: sem limite.

**Documentação**

Workspace: `__Tarefas Rodrigo/limite-chaves-pix-por-conta/` (resumo da tarefa e diagrama draw.io do fluxo).
