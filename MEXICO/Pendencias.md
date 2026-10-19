# Operação México — Pendências para iniciar

Levantado em 18/10/2026 a partir de `OperacaoMexico.md`, do `PRD-MEXICO-V0.pdf` (rascunho, 30/09/2026) e da reunião de 24/09/2026.

Situação: não existe código da frente México. O consolidado técnico está em `OperacaoMexico.md` (atualizado em 16/10/2026).

## 1. Bloqueios para começar a integrar (Caio Cesar / Fintoc)

- [ ] Credenciais e chaves de teste (`sk_test_...`) da conta SAQ na Fintoc. Sem isso não há sandbox.
- [ ] Confirmar que "subconta" no acordo é uma `Account` por cliente (padrão B), e não só uma CLABE (padrão A). Muda modelo de dados e fluxo de tarifa.
- [ ] Pedido formal, por e-mail, do aumento de 10 para 50 req/s. Decidido em 24/09, sem confirmação da Fintoc.
- [ ] Custo por `Transfer`. Se `Transfer` entre `Account`s da mesma organização tem custo e é imediato no México.
- [ ] Limite de quantidade de `Account`s por Entity (a doc só fala em 1 milhão de CLABEs por organização).
- [ ] Onde os recursos ficam custodiados e em nome de quem (pergunta regulatória).
- [ ] Se o "uso da conta" aceita clientes de apostas, caso o piloto seja uma bet.
- [ ] Como o "operador chinês" opera direto na conta principal sem subconta nem documento.

## 2. Decisões de produto e liderança

- [ ] **Escopo do portal:** só extrato (Matheus Falcão, 24/09) ou com SPEI OUT e cobrança (PRD, 30/09). As fontes se contradizem. Afeta telas, API e tarifa.
- [ ] **Onboarding (PS01):** a Fintoc não faz KYC do cliente final. Definir cadastro próprio da QAS, perfis atendidos (pessoa física, empresa, estrangeiro), documentos obrigatórios, status e quem aprova.
- [ ] **Status das contas (PS02):** estados e comportamento de cada um para recebimento, envio e consulta.
- [ ] **Limites e processamento (PS03):** limites por operação e por período, status, prazos de acompanhamento, cancelamento e devolução.
- [ ] **Tarifa (PS05):** por operação ou varredura periódica. Modelos de cobrança, valores padrão, isenções, vigência e conta sem configuração.
- [ ] **Permissões (PS06):** matriz de perfis e quais ações exigem aprovação de segundo usuário.
- [ ] **Webhooks (PS07):** catálogo de eventos da V0, conteúdo das notificações e regras de retentativa.
- [ ] **Moeda e clientes internacionais (PS08):** confirmar MXN, elegibilidade de estrangeiros e o que a API externa expõe.
- [ ] **Primeiro cliente piloto** (Júnior).
- [ ] **Comprovante SPEI no nome da QAS:** só o padrão C mostra o nome do cliente. Confirmar se é aceitável.
- [ ] **Stack:** serviços novos (como `anticipation-apps`) ou reaproveitar o core da SaQ (usuários, scopes, backoffice, `saq-cache-client`).
- [ ] **"Plano de Desenvolvimento Técnico — México v.0":** citado no PRD, não está na pasta. Verificar se existe.

## 3. Ações combinadas em 24/09 sem registro de conclusão

| Quem | Ação | Status |
|---|---|---|
| Rodrigo Barros | Revisar a doc da Fintoc com o Mateus (tarifas e limites) | [ ] |
| Rodrigo Barros | Entrar no grupo de WhatsApp do México (Munir adiciona) | [ ] |
| Gabriel Barros | Levantar pontos técnicos da doc da Fintoc e enviar ao Caio | [ ] |
| Caio Cesar | Pedir o aumento de req/s, compartilhar credenciais, investigar o operador chinês | [ ] |
| Jose Vieira Junior | Definir o piloto; contratar QA | [ ] |
| Matheus Falcão | Alocar segundo dev para o México | [ ] |
| Munir | Adicionar Rodrigo e Gabriel ao grupo | [ ] |

## 4. Correções no PRD (ainda é rascunho)

- [ ] Faltam RN-015, RN-016, RN-063 e PS04.
- [ ] Numeração das seções pula (4 → 5.1 → 12 → 13).
- [ ] "Login administrativo" aparece duas vezes.
- [ ] Fala em 11 telas do portal, mas só 4 estão descritas.
- [ ] Referência a "sugestões da seção 6" que não existem.
- [ ] Itens marcados como DIV são citados na legenda, mas nenhum aparece no texto.

## 5. O que dá para começar hoje sem depender de ninguém

- [ ] Modelagem de dados e do ledger para o padrão B: mirror 1:1 com `Account` Fintoc (`fintoc_account_id` + CLABE), idempotência por `transfer.id` / `tracking_key`, saldo no Valkey.
- [ ] Máquina de estados do SPEI OUT (`pending` → `succeeded` / `rejected` / `failed`), tolerante a webhooks fora de ordem, com estorno da reserva em falha e acompanhamento por `GET /v2/transfers/{id}`.
- [ ] Cliente HTTP da Fintoc: autenticação por chave, assinatura JWS (`Fintoc-JWS-Signature` com `nonce` e `ts`), `Idempotency-Key`, valores em centavos.
- [ ] Receptor de webhook com validação do HMAC SHA-256 (`Fintoc-Signature`) e deduplicação por `transfer.id`.
- [ ] Lista única de perguntas para a Fintoc (seção 1), para o Caio enviar de uma vez.

## 6. Primeiro passo sugerido

1. Cobrar do Caio as credenciais de teste e a confirmação do padrão B.
2. Fechar com Matheus e Gabriel o escopo do portal (extrato só ou com SPEI OUT).
3. Em paralelo, iniciar o cliente Fintoc e a modelagem (seção 5).

## 7. Divisão de trabalho (time: Rodrigo no backend + 1 dev frontend)

### Backend / integração (Rodrigo)

- [ ] Modelagem: mirror 1:1 com `Account` Fintoc, ledger no Valkey, transações, taxas, ajustes.
- [ ] Cliente Fintoc: autenticação, JWS, `Idempotency-Key`, criação de `Account`, `Transfer`, consulta.
- [ ] Receptor de webhooks da Fintoc: validação HMAC, deduplicação, máquina de estados do SPEI OUT.
- [ ] Autenticação e perfis (`Cliente:web`, `backoffice:admin`), reaproveitando scopes da SaQ se a decisão de stack for essa.
- [ ] API do portal: login, onboarding, saldo, CLABE, cobrança SPEI, SPEI OUT, extrato.
- [ ] API do backoffice: onboarding, contas, usuários, taxas, webhooks do cliente, extratos.
- [ ] API pública para integradores + documentação (OpenAPI).
- [ ] Webhooks da QAS para o cliente, com retentativa e log de entrega.
- [ ] Conciliação: `Movement`s da Fintoc contra o ledger, registro de divergência.

### Frontend

- [ ] Portal do cliente: login, abertura de conta e acompanhamento, saldo e CLABE, cobrança SPEI, SPEI OUT (se entrar no escopo), extrato com filtros e detalhe.
- [ ] Backoffice: as 12 telas da seção 12 do PRD, no padrão visual das IPs atuais.

### O que o frontend precisa de mim para não ficar parado

- [ ] **Contrato de API antes do código.** Publicar um OpenAPI dos endpoints do portal e do backoffice na primeira semana, mesmo que mude depois.
- [ ] **Mock server** a partir do OpenAPI, para o front desenvolver sem depender da Fintoc nem do backend pronto.
- [ ] **Modelo de autenticação definido:** token, refresh, scopes e como o front descobre o perfil do usuário.
- [ ] **Formato padrão de erro e paginação**, igual em todos os endpoints.
- [ ] **Ambiente de dev** com dados de exemplo, assim que o backend tiver o mínimo.
- [ ] **Decisão do escopo do portal** (extrato só ou com SPEI OUT). Sem isso o front não sabe quantas telas fazer.

### Pendências específicas do frontend

- [ ] Definir a stack do front: mesma dos backoffices atuais das IPs ou nova.
- [ ] Protótipos do backoffice não existem. O PRD só lista as telas e diz que o designer segue o padrão das IPs. Confirmar se há designer alocado ou se o front parte das telas atuais.
- [ ] Protótipos do portal existem só para 4 telas (início, SPEI IN, SPEI OUT, extrato). Login, onboarding e detalhe de operação não têm desenho.
- [ ] Idioma do portal e do backoffice: espanhol, português ou ambos. Não está no PRD.
- [ ] Formato de moeda (MXN, centavos), fuso horário do México e formato de CLABE nas telas.

### Ordem sugerida das primeiras semanas

| Semana | Backend (Rodrigo) | Frontend |
|---|---|---|
| 1 | OpenAPI do portal e backoffice; modelo de auth; mock server; modelagem de dados | Setup do projeto; login; layout base; telas de extrato (entra em qualquer escopo) |
| 2 | Cliente Fintoc + receptor de webhook em sandbox (depende das chaves do Caio); auth real | Saldo e CLABE; telas de contas e onboarding do backoffice contra o mock |
| 3 | SPEI IN de ponta a ponta; abertura de conta; API de extrato real | Troca do mock pela API real do extrato; onboarding do cliente |
| 4 | SPEI OUT com máquina de estados e estorno; taxas | SPEI OUT e cobrança (se no escopo); taxas e usuários do backoffice |

A ordem assume que as chaves de teste da Fintoc chegam até a semana 2. Se atrasarem, o backend segue com mock da Fintoc e o front não é afetado.
