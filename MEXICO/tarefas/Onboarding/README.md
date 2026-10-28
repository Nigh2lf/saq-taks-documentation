# Tarefa: Onboarding (abertura de conta)

Status em 28/10/2026: **concluída em homologação** na conta AWS `qasmexhml` (707337761900).

## O que ficou no ar

- `mexico-api-v3` (ALB público `api.qasmexhml.qastech.com.br`): `POST /v3/onboarding/` e consulta de status.
- `mexico-backoffice-api` (ALB interno, VPN `qasmexhml`): login em dois passos (senha + TOTP), listagem, aprovação, reprovação e reenvio dos pedidos, e criação de pedido pelo operador (via API pública, no padrão do SaQ).
- `mexico-migrate-cron-job`: migrations do `mexico-repository`.
- Lambda `lambda-aws-mexico-onboarding-approve-event-qasmexhml-handle-sqs` na fila `approve-onboarding` (DLQ, `maxReceiveCount` 3).
- `mexico-backoffice-frontend` no Amplify: `backoffice.qasmexhml.qastech.com.br`, branch `develop`.
- Banco `QASMEXHML` no RDS `qasmexhml-db`, com seeds de policies e perfis (`backoffice:admin`, `Cliente:web`).

## Validação ponta a ponta (28/10)

Pedido "Rodrigo LTDA" (RFC `QXC010716RK5`): criado → aprovado no backoffice → fila → lambda → conta na Fintoc sandbox com CLABE `999969789305622038` → `COMPLETED` em 10 s → usuário web criado com perfil `Cliente:web`. Fila e DLQ zeradas; lambda com 1 invocação, 0 erros.

## Arquivos desta pasta

| Arquivo | Conteúdo |
| --- | --- |
| `Sandbox-Onboarding.md` | Registro do sandbox local e da infra na AWS (22/10), com o passo a passo e os ids de teste da Fintoc. |

## Pendências ligadas a esta tarefa

- `GET /v3/me` na `mexico-backoffice-api` para o perfil do operador (hoje o front usa o e-mail do login).
- `mexico-onboarding-frontend` no Amplify (produto `onboarding-mexico`).
- Ligar o autoSync do ArgoCD em `dev-qasmexhml`.
- Login web, definição de senha e API key do cliente no `mexico-api-v3` (pré-requisito do SPEI).
