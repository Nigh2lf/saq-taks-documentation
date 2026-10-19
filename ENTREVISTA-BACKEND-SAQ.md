# Entrevista backend — plataforma SaQ

Roteiro para entrevistar candidato backend que vai atuar na plataforma SaQ (APIs v3, lambdas, runners, libs compartilhadas). Baseado no que o código exige de fato; exemplos tirados da tarefa de limite de chaves Pix por conta (10/2026).

Escala sugerida por critério: 0 não conhece · 1 ouviu falar · 2 já usou · 3 explica trade-offs com exemplo próprio.

## 1. Capacidades essenciais

| # | Capacidade | O que o projeto exige | Como aparece no código |
|---|---|---|---|
| 1 | Python assíncrono | `async/await` em todo o código v3; sessão e transação explícitas; o que acontece quando uma exceção sai de dentro de `begin()` | FastAPI, SQLAlchemy 2.x async com `asyncmy`, `awaitable_attrs`, `SessionManager` / `get_session().begin()` |
| 2 | SQLAlchemy + MySQL | Modelos mapeados, repositórios, `select`/`func.count`, `FOR UPDATE`, isolamento, gap locks; custo de travar uma linha que carrega saldo durante chamada externa | `PixKeyAccountBankRepository.count_active_by_account`, `AccountBankMirror.amount` |
| 3 | Django só para schema | SAQ-Backend é dono do banco: `AddField`, `RunPython` com `apps.get_model`, reverso, numeração e dependências, duas heads e merge migration | migrations `0492` / `0493` |
| 4 | Pydantic v2 e contratos | Schemas de request/resposta, `exclude_unset` × `exclude_none`, `Decimal`; códigos HTTP com critério (400 regra de negócio, 404, 409 duplicidade, 422 validação) | `UpdateAccountBankRuleRequestSchema`, `AppError` i18n |
| 5 | Mensageria AWS | Lambda com Chalice consumindo SQS: `batch_size`, visibility timeout, `maxReceiveCount`, DLQ, idempotência por `messageId`; falha transitória (retry) × rejeição de negócio (consumir e logar) | `lambda-aws-manage-pix-keys-event/app.py` |
| 6 | Cache Valkey/Redis | Parâmetros globais, DTOs de conta em cache, Lua para saldo, locks com TTL e token por dono; o que vai para o cache e o que fica no banco | `get_global_parameter`, `AccountBaseCacheDTO`, `debit_account` |
| 7 | Pacotes internos e versões | Monorepo de libs publicado no CodeArtifact, pins em `requirements.txt`, ordem de merge/publicação entre repos dependentes, canal `.devN`, ciclo de import e import tardio | `saq-repository` 3.51.0 → `saq-shared-libraries` 1.57.0 → consumidores |
| 8 | Testes com mocks async | `pytest`, `pytest-asyncio`, `AsyncMock`, `patch` no namespace do módulo da rota; testar o contrato sem banco e saber o que o mock esconde | `tests/test_pix_key_active_limit.py` |
| 9 | Domínio Pix / DICT | Tipos de chave, inclusão, exclusão, alteração de vinculação, portabilidade e reivindicação, limites BACEN (5 PF / 20 PJ), provedores PSP (JDPI, ONZ); nada existe até estar no DICT | `saq-proxy-client`, rotas `pix_key_*` da dict-api-v3 |
| 10 | Domínio financeiro básico | Conta espelho × bolsão, saldo e bloqueado, razão com débito/crédito, auditoria das operações do operador | `AccountBankMirror`, `finance_accountbanktransaction`, `set_audit_log_context` |

## 2. Desejáveis

- Kubernetes com ArgoCD/EKS, GitHub Actions, Secrets Manager: investigar pod em CrashLoop por variável ausente no secret.
- i18n de erros com chaves e locales (`errors.*`, pt-BR / en).
- OpenTelemetry e logs estruturados.
- Alembic (usado na antecipação), com a mesma dor de heads múltiplas.
- Leitura de legado espalhado em dezenas de repos (APIs, lambdas, runners, crons, libs): descobrir onde a regra já existe antes de escrever.

## 3. Postura que o projeto cobra

- Procura o precedente antes de inventar (o padrão "conta se tiver, senão global, zero é sem limite" já existia para o QR Code).
- Checa a regra antes de chamar serviço externo, para não ter que desfazer.
- Documenta decisões e premissas no PR e pergunta ao P.O. o que não está definido.
- Lê review com calma e sabe discordar com argumento (ex.: trava de banco descartada por contenção com o saldo).

## 4. Perguntas técnicas (com resposta esperada)

1. **Corrida.** "Uma rota conta chaves ativas e insere uma nova. Dois pedidos chegam ao mesmo tempo. O que acontece e como resolveria?"
   Espera: identifica a corrida; cita `FOR UPDATE` e o custo de travar a linha de saldo durante a chamada ao DICT; alternativas (lock em Redis, travar outra linha, aceitar a corrida) com prós e contras.
2. **Fila.** "Uma lambda que consome SQS levanta exceção por regra de negócio. O que acontece com a mensagem?"
   Espera: reentrega até `maxReceiveCount`, DLQ; regra de negócio determinística deve ser consumida com log, retry só para falha transitória.
3. **Ordem de deploy.** "Precisa adicionar uma coluna numa tabela que três serviços leem por uma lib compartilhada. Qual a ordem?"
   Espera: migration no dono do schema primeiro; lib com modelo novo; publicar; consumidores com pin novo; o que quebra se inverter.
4. **Cache.** "Quando um campo de regra da conta deve ir para o cache?"
   Espera: só se alguma API lê no caminho quente; senão fica no banco; custo de manter DTO e restore em sincronia.
5. **Regra de negócio.** "Uma chave é alterada da conta A para a conta B do mesmo cliente. Qual conta é checada contra o limite?"
   Espera: a de destino; percebe que a origem perde uma; pergunta se o limite é por conta ou por cliente.
6. **HTTP.** "Limite atingido: 400 ou 409?"
   Espera: depende do padrão do projeto; argumenta 409 para conflito com recurso existente (duplicidade) e 400 para regra de negócio; olha o que a API já faz antes de decidir.
7. **Transação.** "Exceção sai de dentro de `async with session.begin()`. O que o banco vê?"
   Espera: rollback automático; nada escrito; mas o que foi feito em serviço externo antes não volta.
8. **Migrations.** "Depois de um `git pull`, sua migration 0484 colidiu com outra 0484. O que faz?"
   Espera: renumerar e apontar `dependencies` para a head atual; o número é convenção, a dependência é o que manda; evitar merge migration desnecessária.

## 5. Exercício prático (30 min, com código na tela)

Mostrar `pix_key_include.py` da dict-api-v3 e pedir: "Onde você colocaria uma checagem de limite de chaves por conta, e por quê nesse ponto?"
Observar: lê o fluxo antes de responder; identifica o insert e a chamada ao DICT; coloca a checagem antes dos dois; pergunta onde fica o limite (conta × global); pensa nos outros pontos que inserem chave (alteração, reivindicação, backoffice, fila).

## 6. Sinais de alerta

- Resolve a corrida com `FOR UPDATE` sem perguntar o que mais usa a linha.
- Trata toda exceção da lambda do mesmo jeito.
- Sobe pins sem saber se a versão existe.
- Não pergunta ao P.O.; assume regra de negócio em silêncio.
