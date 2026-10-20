# Operação México — onde o projeto vive: dentro da plataforma SaQ, fora, ou quase fora

Escrito em 19/10/2026 para a discussão de arquitetura. Compara três caminhos para o mesmo escopo (SPEI via Fintoc, portal, API pública e backoffice). Em todos os três o México tem **banco próprio, ledger próprio, autenticação própria e backoffice próprio**: isso já está decidido e está em `Reaproveitamento.md`. O que muda aqui é **quanto da plataforma SaQ (libs, CI, cluster, infra) o México usa**.

## 1. As três opções em uma frase

| | Opção | Em uma frase |
|---|---|---|
| **1** | **Dentro da plataforma** (como está no `Reaproveitamento.md`) | Uma pasta `mexico-apps/` na organização SaQDigital, com a mesma stack, as libs privadas do CodeArtifact, o `saq-ci`, o `Kubernetes-manifest` e o cluster, o Valkey e os segredos da SaQ. Só banco, auth e ledger são do México. |
| **2** | **Tudo fora** | Projeto independente: repositórios, stack, libs, CI, conta AWS e domínio próprios. Nenhuma dependência da SaQ, nem de código nem de infra. Só o conhecimento é reaproveitado. |
| **3** | **Quase tudo fora** | Repositórios, infra e CI próprios, mas consumindo as três libs de plataforma que não enxergam banco (`saq-base-api`, `saq-base-repository`, `saq-utilities`) e copiando os padrões como código. |

Para a opção 3 há uma variante **3b**: em vez de consumir as libs do CodeArtifact, copiar o código delas uma vez para o repositório do México e deixar divergir.

## 2. O que a plataforma SaQ dá hoje, e quem usa em cada opção

Levantado nos repos do crypto e da antecipação, que são os produtos separados mais recentes.

| Peça | O que é | Opção 1 | Opção 2 | Opção 3 |
|---|---|---|---|---|
| `saq-base-api` | FastAPI com health, CORS, envelope de erro, paginação, métricas Prometheus, i18n (só `en` e `pt-BR`) | usa | reescreve | usa (ou copia, 3b) |
| `saq-base-repository` | sessões async com vários bancos nomeados, `SessionManager`, `BaseRepository`, lock nomeado | usa | reescreve | usa (ou copia) |
| `saq-utilities` (auth, AWS, HTTP, exceções, tracing) | validação de JWT com bitmap de permissões, `BaseAPIClient`, SQS, Valkey | usa | reescreve | usa (ou copia) |
| `saq-utilities` (helpers BR) | CPF/CNPJ, ISPB, Pix, "R$", UTC-3 | não usa | — | não usa |
| `saq-cache-client` | saldo no Valkey via Lua, cache de usuário e empresa do core | copia o padrão com prefixo | reescreve | copia o padrão |
| `saq-permissions`, `saq-repository` | catálogo e modelos do core | não usa | — | não usa |
| `saq-ci` | 4 workflows reutilizáveis: build Docker + kustomize, deploy Chalice, publicação no CodeArtifact. O deploy do crypto-api-v3 são 15 linhas de YAML apontando para eles | usa | CI própria | copia os workflows |
| `Kubernetes-manifest` | um repo com todos os manifests; namespaces `apps-v3`, `apps-crypto`, `apps-anticipation`; ArgoCD | namespace `apps-mexico` | cluster próprio | manifests no repo do México, cluster próprio |
| Cluster EKS, ingress, Valkey, Secrets Manager, SQS | infra compartilhada. O crypto entra pelo mesmo host `api.*.saq.digital` com paths próprios e lê os segredos comuns (`credenciais-auth-token`, `apps-configs`) além dos seus | compartilha | próprio | próprio |
| CodeArtifact | repositório privado de pacotes. Hoje não há venv nem acesso às dependências privadas na máquina de dev; os testes rodam só no CI | depende | não depende | depende (ou não, em 3b) |
| Frontends | `portal-whitelabel-frontend` (Next 16, i18n pt/en/zh) e `backoffice-whitelabel-frontend` (Next 15, pt-BR fixo) | apps novos a partir deles | apps novos do zero | apps novos a partir deles |

Em todas as opções o que fala com o SPEI é novo: cliente Fintoc, receptor de webhook, máquina de estados, conciliação, CLABE, KYC, telas.

## 3. Comparação por critério

| Critério | Opção 1 (dentro) | Opção 2 (tudo fora) | Opção 3 (quase tudo fora) |
|---|---|---|---|
| **Tempo até o primeiro deploy** | o menor. Pipeline, manifests, segredos e observabilidade já existem; o crypto-api-v3 serve de template funcional | o maior. Antes de uma rota de negócio existir, é preciso montar base de API, sessão de banco, auth, CI, cluster, observabilidade e segredos | intermediário. As libs e o padrão de CI vêm prontos; a infra é montada do zero |
| **Código reaproveitado** | libs como estão; padrões por cópia | nada | libs como estão; padrões por cópia |
| **Acoplamento de versão** | as libs são fixadas por versão no `requirements.txt` (o crypto está em `1.48.0`). Um bump da SaQ não quebra o México sozinho, mas correções de segurança nas libs exigem acompanhar | nenhum | igual à opção 1; em 3b, nenhum (e nenhuma correção chega de graça) |
| **Raio de impacto de incidente** | compartilha cluster, Valkey, ingress e Secrets Manager. Uma falha nesses componentes derruba os dois produtos. Colisão de chave no Valkey é risco real sem prefixo | isolado | isolado em runtime; só o código das libs é comum |
| **Isolamento de segredos e identidade** | chaves JWT próprias e banco próprio já isolam o login. Mas o `SecretProviderClass` do crypto monta também segredos comuns da SaQ; o México precisaria de um só com os seus | total | total |
| **Operação (deploy, on-call, monitoramento)** | o mesmo time de infra, as mesmas ferramentas, o mesmo ArgoCD | um segundo ambiente para operar, com tudo duplicado | um segundo ambiente para operar, mas com as mesmas ferramentas e convenções |
| **Experiência de dev** | precisa de acesso ao CodeArtifact para rodar local, o que hoje não existe na máquina | livre | igual à opção 1 (ou livre, em 3b) |
| **Regras do Brasil vazando** | o risco maior. As libs e os templates têm BRL, UTC-3, CPF/CNPJ e Pix espalhados; é preciso disciplina para não importar | nulo | menor que na 1, porque só três libs entram, e elas não têm regra de negócio |
| **Idioma** | `saq-base-api` precisa ganhar `es-MX` (mudança pequena, mas na lib da SaQ) | i18n do zero no idioma certo | igual à 1, ou local em 3b |
| **Residência de dados e requisito regulatório mexicano** | dados em conta e região da SaQ. Se houver exigência de dados no México ou de entidade separada, essa opção não atende | atende qualquer exigência | atende, porque a infra é própria |
| **Custo de infra** | marginal: um RDS, filas e pods a mais | cheio: conta, cluster, cache, observabilidade | cheio, igual à 2 |
| **Separação societária futura** (QAS México como empresa à parte, venda, auditoria separada) | difícil de desmembrar depois | já nasce separado | fácil de desmembrar: basta trocar as três libs por cópia |

## 4. Riscos próprios de cada opção

**Opção 1.** O risco é o acoplamento silencioso. Hoje nenhum produto da SaQ tem ledger próprio; o México será o primeiro, e vai tentar reaproveitar funções de saldo e de cache feitas para `finance_accountbanktransaction` e para `account:<id>`. Cada reaproveitamento desses é um ponto onde o Brasil entra pela porta dos fundos. O segundo risco é operacional: incidente no cluster ou no Valkey afeta os dois países, e o México passa a disputar janela de deploy e atenção de on-call com o Pix.

**Opção 2.** O risco é reescrever plataforma em vez de produto. O que a SaQ levou anos para estabilizar (sessão de banco, auth com bitmap, envelope de erro, deploy com kustomize e ArgoCD) seria refeito antes da primeira transferência SPEI. Com um backend só, o cronograma do PRD não fecha. E a segunda base de código não teria quem a mantivesse além do próprio time do México.

**Opção 3.** O risco é ficar no meio sem decidir. Se a infra é própria mas as libs vêm do CodeArtifact, o México depende de um recurso da SaQ (o repositório de pacotes) sem ganhar o resto da plataforma. A variante 3b resolve isso, ao custo de não receber mais correções das libs. Também é a opção que mais depende de infra: alguém precisa criar conta, cluster, cache, observabilidade e segredos antes do primeiro deploy, e esse trabalho não é do backend.

## 5. O que decide entre elas

As perguntas abaixo são de negócio e infra, não de código. A resposta a elas escolhe a opção.

1. **Há exigência de dados no México ou de entidade jurídica separada?** Sim elimina a opção 1.
2. **Quem opera o México em produção?** Se é o mesmo time de infra da SaQ, as opções 1 e 3 custam pouco a mais; a 2 custa um ambiente inteiro.
3. **O México pode virar empresa ou produto vendido separadamente?** Sim favorece a 3 (ou a 3b).
4. **O time de backend do México é o mesmo da SaQ?** Sim favorece manter stack e libs (1 ou 3). Time diferente, com outra stack, favorece a 2.
5. **Qual o prazo do primeiro SPEI em produção?** Quanto menor, mais a opção 1 pesa.
6. **A SaQ aceita abrir o `saq-base-api` para `es-MX` e manter as libs para dois consumidores?** Se não, 3b.

## 6. Recomendação

**Opção 1 como ponto de partida, com quatro fronteiras emprestadas da opção 3**, que custam pouco e evitam o acoplamento que importa:

- namespace `apps-mexico` com `SecretProviderClass` próprio, sem montar os segredos comuns da SaQ;
- Valkey próprio ou, no mínimo, prefixo `mx:` em todas as chaves;
- host próprio no ingress (por exemplo `api.mx.*`), em vez de paths dentro de `api.*.saq.digital`;
- nenhuma importação de `saq-cache-client`, `saq-permissions`, `saq-repository` nem dos helpers BR. Só as três libs de plataforma.

Com isso o México entrega mais rápido, opera com as ferramentas que o time já conhece e continua desmembrável: se as perguntas 1 ou 3 da seção 5 responderem "sim" mais tarde, migrar para a opção 3 é mover manifests e infra, sem reescrever código, porque as três libs seguem iguais.

A opção 2 só se justifica se o time ou a stack forem outros. A opção 3 desde o início só se justifica se a residência de dados ou a separação societária já estiverem decididas.
