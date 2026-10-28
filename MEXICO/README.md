# Operação México — documentação

Organizada em 28/10/2026 em três pastas. Comece por `entendimento/GuiaInicial.md`.

| Pasta | O que vai aqui |
| --- | --- |
| `entendimento/` | Contexto do projeto, que não muda por tarefa: PRD, escopo da V0, consolidado técnico da Fintoc, decisões de arquitetura, o que reaproveitar da SaQ, pendências gerais, atas e notas de reunião, links úteis. |
| `fluxos/` | Diagramas draw.io dos fluxos e da arquitetura. |
| `tarefas/<Tarefa>/` | Uma pasta por tarefa com o registro do que foi feito, decisões e o que ficou pendente daquela tarefa. |

## entendimento/

| Arquivo | Conteúdo |
| --- | --- |
| `GuiaInicial.md` | Visão para quem está começando (18/10). |
| `V0.md` | Conta de sandbox da Fintoc e escopo da V0 por frente (portal, backoffice, API). |
| `PRD-MEXICO-V0.pdf` | PRD da fase V0 (rascunho de 30/09), regras RN-011 a RN-066. |
| `OperacaoMexico.md` | Consolidado técnico: endpoints, estados e eventos da Fintoc, fluxos propostos de SPEI IN/OUT e tarifa (16/10). |
| `Reaproveitamento.md` | O que copiar da SaQ e o que criar do zero, com o caminho de cada padrão (19/10). |
| `Comparativo-Arquitetura.md` | Onde o projeto vive: decisão de conta AWS própria com libs, CI, manifests e infra compartilhados (19/10, decidido em 20/10). |
| `Pendencias.md` | Pendências gerais para iniciar: bloqueios com a Fintoc, decisões de produto, ordem das semanas (18/10). |
| `Granola meeting - Mexico 24-09-2026.txt` | Ata da reunião de 24/09. |
| `alinhamento_call.txt` | Notas da call de alinhamento técnico (tabela de transação particionada, tudo por fila, `integration-mx`). |
| `documentação.txt` | Links: guia da Fintoc para wallet de usuários finais e o artifact do Claude. |

## fluxos/

| Arquivo | Conteúdo |
| --- | --- |
| `Fluxo-Onboarding-Mexico.drawio` | Fluxo do onboarding: pedido, aprovação no backoffice, fila, lambda, Fintoc. |
| `Arquitetura-Mexico-hml.drawio` | Arquitetura da conta `qasmexhml` como ficou em 28/10 (EKS, RDS, Secrets, SQS, lambda, Amplify, Fintoc). Cópia de `SAQ/arquitetura-mexico.drawio`. |

## tarefas/

| Pasta | Conteúdo |
| --- | --- |
| `Onboarding/` | Abertura de conta: sandbox local, infra na AWS e validação ponta a ponta. |

Próxima tarefa prevista: `SPEI/` (SPEI IN, SPEI OUT e cobrança).
