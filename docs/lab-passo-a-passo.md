# Laboratório passo a passo

Registro de tudo o que foi feito para montar este laboratório, na ordem em que aconteceu, entre 14 e 17 de outubro de 2026. Cada passo traz o objetivo, o que foi feito no GitHub, no Databricks e no terminal, como conferir o resultado e os erros que apareceram no caminho.

**Ambiente usado:**

- Databricks Free Edition (um workspace, só serverless);
- GitHub Free, com repositório público;
- Databricks CLI v1.21.0 num MacBook (zsh).

Nos comandos, `<host>` é a URL do workspace (está em `bundles/vendas/databricks.yml`) e `<você>` é o seu usuário no Databricks.

---

## Sumário

| Fase | Passos |
|---|---|
| [Preparação](#fase-1-preparação) | 1 a 6: repositório, CLI, service principals, catálogos e secrets |
| [O bundle](#fase-2-o-bundle) | 7 a 9: `bundle init`, Dev e validação de HML/PRD |
| [A esteira](#fase-3-a-esteira) | 10 a 14: environments, workflow, primeiro deploy, ruleset e manutenção do CI |
| [Testes de proteção](#passo-15-testes-de-proteção) | 15.1 a 15.5: drift, remoção, branch errada, concorrência e dados por ambiente |
| [Pendente](#passo-16-limpeza-pendente) | 16: limpeza |
| [Resumo de erros](#resumo-dos-erros-e-correções) | todos os erros e correções numa tabela |

---

## Fase 1: preparação

### Passo 1: pasta do projeto

```bash
mkdir -p ~/databricks/dabs-cicd-lab
cd ~/databricks/dabs-cicd-lab
git init -b main
```

### Passo 2: repositório no GitHub

**No GitHub:** criar o repositório `dabs-cicd-lab`, **público**. No GitHub Free, os *required reviewers* e as regras de branch dos environments, que usamos no passo 10, só funcionam em repositório público.

**No Mac:**

```bash
git config user.name "<seu nome>"
git config user.email "<id>+<usuario>@users.noreply.github.com"
git remote add origin git@github.com:<usuario>/dabs-cicd-lab.git
git push -u origin main
```

| Problema | Causa | Correção |
|---|---|---|
| `Author identity unknown` no primeiro commit | git sem nome e e-mail configurados nesse computador | `git config user.name/user.email` **local** no repositório, com o e-mail *noreply* do GitHub, já que o repositório é público |
| `403 … denied to <outra conta>` no push por HTTPS | o Keychain do macOS guardava a credencial de outra conta do GitHub | remote por SSH, com a chave que já existia. Teste: `ssh -T git@github.com` responde `Hi <usuario>` |

Observação: o GitHub não aceita mais senha no git por HTTPS, mas HTTPS com token continua funcionando. A troca para SSH foi por praticidade.

### Passo 3: Databricks CLI

```bash
brew upgrade databricks/tap/databricks
databricks --version                          # Databricks CLI v1.21.0
databricks auth login --host <host>           # login como pessoa, só para Dev
```

A versão que estava instalada era a v1.16.1. A v1.21.0 é de 14/10/2026.

### Passo 4: service principals

Quatro service principals, dois por ambiente, criados por um admin do workspace (**Settings → Identity and access → Service principals**):

| Service principal | Papel | Ambiente |
|---|---|---|
| `sp-lab-deploy` | publica o bundle (identidade do CI) | HML |
| `sp-hml-run` | executa os jobs | HML |
| `sp-prd-deploy` | publica o bundle (identidade do CI) | PRD |
| `sp-prd-run` | executa os jobs | PRD |

Os *application IDs* estão em `databricks.yml`, com o nome de cada SP em comentário.

Ainda neste passo, cada SP de deploy precisa da permissão **Use** (o papel `servicePrincipal.user`) no SP de execução do seu ambiente. Ela é configurada em **Service principals → `<sp de run>` → Permissions**. Sem isso, o deploy com `run_as` falha (ver passo 12).

| Problema | Correção |
|---|---|
| A tentativa de renomear o SP de deploy de HML pela API SCIM não teve efeito | O nome `sp-lab-deploy` foi mantido |

### Passo 5: catálogos

Três catálogos, criados **pela UI** (Catalog → **Create catalog**), com **Default Storage**: `dev`, `hml` e `prod`.

| Problema | Causa | Correção |
|---|---|---|
| `databricks catalogs create` falhou com *"Metastore storage root URL does not exist … Default Storage"* | na Free Edition o catálogo precisa usar o Default Storage, e pela CLI esse caminho não funcionou | criar pela UI |

Depois, os grants no nível do catálogo (conferidos com `SHOW GRANTS ON CATALOG` em 17/10/2026):

| Catálogo | Principal | Privilégios |
|---|---|---|
| `hml` | `sp-lab-deploy` | `USE CATALOG`, `CREATE SCHEMA` |
| `hml` | `sp-hml-run` | `USE CATALOG`, `USE SCHEMA`, `SELECT`, `MODIFY`, `CREATE TABLE` |
| `prod` | `sp-prd-deploy` | `USE CATALOG`, `CREATE SCHEMA` |
| `prod` | `sp-prd-run` | `USE CATALOG`, `USE SCHEMA`, `SELECT`, `MODIFY`, `CREATE TABLE` |
| `dev`, `hml`, `prod` | `account users` | `BROWSE` (já vem assim) |

O SP de deploy só cria schemas. O SP de execução lê e grava em qualquer schema do catálogo do seu ambiente, porque privilégio dado no catálogo vale para tudo o que está dentro dele. Em `dev` não há grant extra: você é o dono do catálogo. Os comandos estão em [`fundacao.md`](fundacao.md).

### Passo 6: secrets OAuth dos SPs de deploy

Em **Service principals → `sp-lab-deploy` → Secrets → Generate secret**. O mesmo vale para o `sp-prd-deploy`.

O secret aparece uma vez só: ele vai direto para o GitHub no passo 11 e não fica salvo em arquivo nenhum.

Teste de login M2M no Mac, sem gravar o secret em lugar nenhum:

```bash
read -s "SECRET?Cole o secret do SP: "
echo
DATABRICKS_HOST=<host> \
DATABRICKS_AUTH_TYPE=oauth-m2m \
DATABRICKS_CLIENT_ID=<application-id do SP> \
DATABRICKS_CLIENT_SECRET="$SECRET" \
databricks current-user me
unset SECRET
```

O `read -s "VAR?texto"` é a sintaxe do zsh. No bash, o equivalente é `read -s -p "texto" SECRET`.

| Problema | Causa | Correção |
|---|---|---|
| `API is disabled for users without … workspace-access` | o SP não tinha o entitlement de acesso ao workspace | marcar **Workspace access** nos entitlements do SP de deploy |

---

## Fase 2: o bundle

### Passo 7: `bundle init` e ajustes

```bash
mkdir -p bundles && cd bundles
databricks bundle init default-python      # nome do projeto: vendas
```

Os ajustes foram feitos em três commits:

1. **Schema como recurso do bundle:**
   - `resources/vendas.schema.yml` com `catalog_name: ${var.catalog}`;
   - no job, `schema` passou a ser `default: ${resources.schemas.vendas.name}`.

   A referência só é resolvida no deploy: o `validate` mostra o texto literal.
2. **Targets `hml` e `prod`:** `mode: production` e `root_path` na pasta do ambiente, mais:
   - `presets.name_prefix` (`[hml] `, `[prd] `);
   - `presets.trigger_pause_status: PAUSED`;
   - `run_as` com o SP de execução;
   - `permissions` para o SP de deploy (`CAN_MANAGE`), o SP de run (`CAN_VIEW`) e o grupo `users` (`CAN_VIEW`).

   O `prod` ganhou ainda `git.branch: main`.
3. **Notebook sem dependência editável** (ver passo 8).

O resultado está em [`bundles/vendas/databricks.yml`](../bundles/vendas/databricks.yml).

### Passo 8: Dev

```bash
cd bundles/vendas
databricks bundle deploy -t dev
databricks bundle run -t dev sample_job
```

**Para conferir:**

- o job `[dev <você>] sample_job` roda como você;
- o schema `dev.dev_<você>_vendas` existe, porque em `mode: development` o nome dos recursos ganha o prefixo `dev_<você>_`;
- os arquivos estão em `/Workspace/Users/<você>/.bundle/vendas/dev`.

| Problema | Causa | Correção |
|---|---|---|
| Run falhou com *"Library installation failed … Obtaining file:///Workspace/…/files"* | o notebook do template declara a dependência `--editable ..` no environment, para instalar o próprio projeto Python, e isso falhou no serverless | trocar as dependências do environment do notebook por `[]` |

### Passo 9: validar HML e PRD

```bash
databricks bundle validate -t hml
databricks bundle validate -t prod
```

Os dois passaram, com um aviso *"Recommendation: permissions section should explicitly include the current deployment identity"*. O aviso aparece porque quem rodou foi uma pessoa, que não está nas `permissions`. **Não adicione o seu usuário:** no CI, quem roda é o SP de deploy, que já está listado, e o aviso some.

> **Descoberto depois (passo 12):** o `validate` não é só leitura. Se a pasta `files` do `root_path` ainda não existe, ele a cria com a identidade de quem rodou ([código do CLI v1.21.0](https://github.com/databricks/cli/blob/v1.21.0/libs/sync/path.go)). Por isso as pastas `files` de HML e PRD ficaram com o seu usuário como dono. **Regra:** `validate` e `plan` de HML e PRD só no CI.

---

## Fase 3: a esteira

### Passo 10: environments no GitHub

**Settings → Environments:**

| Environment | Configuração |
|---|---|
| `hml` | sem revisores, sem restrição de branch. O `pr-check` usa este environment a partir de branches de PR |
| `prod` | **Required reviewers:** você. **Deployment branches and tags:** *Selected branches and tags* com a regra `main` |

### Passo 11: variables e secrets por environment

Os nomes são os mesmos nos dois environments; o valor muda:

| Nome | Tipo | `hml` | `prod` |
|---|---|---|---|
| `DATABRICKS_HOST` | variable | URL do workspace | URL do workspace |
| `DATABRICKS_CLIENT_ID` | variable | application ID do `sp-lab-deploy` | application ID do `sp-prd-deploy` |
| `DATABRICKS_CLIENT_SECRET` | **secret** | secret do `sp-lab-deploy` | secret do `sp-prd-deploy` |

Com isso, o job que usa `environment: hml` nunca enxerga a credencial de prod.

### Passo 12: workflow e primeiro deploy

O workflow [`.github/workflows/vendas.yml`](../.github/workflows/vendas.yml) entrou pelo PR #1 e tem três jobs:

| Job | Quando roda | O que faz |
|---|---|---|
| `pr-check` | só em `pull_request` | `validate` e `plan` de HML |
| `deploy-hml` | só em `push` na `main` | `deploy` em HML e `run` do job |
| `deploy-prod` | depois do `deploy-hml` | `plan` e `deploy` em PRD, após a aprovação |

O caminho foi PR → `pr-check` verde → merge → `deploy-hml` → aprovação → `deploy-prod`.

| Problema | Causa | Correção |
|---|---|---|
| `deploy-prod` falhou: *"Cannot bind the service principal provided in 'run_as' (sp-prd-run) … must have 'servicePrincipal.user'"* | a permissão **Use** no `sp-prd-run` tinha sido dada ao próprio `sp-prd-run`, e não ao `sp-prd-deploy` | em **Service principals → `sp-prd-run` → Permissions**, dar **Use** ao `sp-prd-deploy`. Depois, **Re-run failed jobs** e aprovar de novo |
| As pastas `files` de HML e PRD tinham o seu usuário como dono, e foram criadas horas antes do primeiro deploy pelo CI | o `validate` do passo 9 criou essas pastas | apagar a pasta `files` dos dois ambientes e rodar **Re-run all jobs**. Os SPs recriaram as pastas |

**Para conferir:**

- `[hml] sample_job` e `[prd] sample_job` aparecem com *Run as* = SP de execução e o agendamento pausado;
- os schemas `hml.hml_vendas` e `prod.prd_vendas` existem;
- as pastas `artifacts`, `files` e `state` têm o SP de deploy do ambiente como dono.

O deploy parcial que falhou já tinha criado o `prd_vendas`, e o re-run convergiu sem duplicar nada.

**Quando o job de PRD roda:** o workflow só publica em PRD, não executa. Além disso, o target `prod` tem `trigger_pause_status: PAUSED`. No lab, o job de PRD só roda com **Run now**. Num time real, a forma de ligá-lo é tirar o `PAUSED` e trocar o `periodic` por um `schedule` com `quartz_cron_expression` e `timezone_id`, tudo por PR.

### Passo 13: proteger a `main`

**Settings → Rules → Rulesets → New branch ruleset:**

| Campo | Valor |
|---|---|
| Ruleset name | `main-protegida` |
| Enforcement status | **Active** |
| Bypass list | vazia, para valer também para admins |
| Target branches | **Include default branch** |
| Restrict deletions / Block force pushes | marcados |
| Require a pull request before merging | marcado, **Required approvals = 0** (um único usuário no lab) |
| Require status checks to pass | `pr-check`, com a fonte fixada em **GitHub Actions** |

**Teste:** um commit vazio e `git push` direto na `main` foram recusados com `GH013`, listando as duas regras. Depois, `git reset HEAD~1` para descartar o commit.

### Passo 14: manutenção do workflow (PR #2)

| Mudança | Motivo |
|---|---|
| `actions/checkout@3d3c42e…  # v7.0.1` | roda em Node 24 (acaba o aviso de Node 20) e fica fixado por SHA |
| `persist-credentials: false` | o token do GitHub não fica salvo no runner |
| `databricks/setup-cli@2bf4520…  # v1.21.0` | fixado por SHA. É uma action *composite* (sem Node) |
| `ubuntu-latest` → `ubuntu-24.04` | o `latest` ia mudar de imagem; com o label explícito, a migração é decisão nossa |
| `paths` removido do `pull_request` | com o filtro de pasta, um PR que não mexe no bundle não dispara o `pr-check`, e o check obrigatório fica preso em *"Waiting for status to be reported"* |

**Resultado:** `Plan: 0 to add, 0 to change, 0 to delete, 3 unchanged` no prod. O workflow mudou, o bundle não, e o deploy repetido não alterou nada.

---

## Passo 15: testes de proteção

### 15.1 Drift: editar um job pela UI

1. Abrir o `[hml] sample_job`. A UI mostra **Connected to Declarative Automation Bundles** e não deixa editar. O CLI publica todo job com `edit_mode: UI_LOCKED` ([código](https://github.com/databricks/cli/blob/v1.21.0/bundle/deploy/metadata/annotate_jobs.go)).
2. **Disconnect from source** (só quem tem `CAN_MANAGE` consegue) e renomear o job.
3. Abrir um PR com commit vazio. O `plan` do `pr-check` mostrou `update jobs.sample_job` e `1 to change`. O Git não mudou, mas o workspace sim.
4. Fechar o PR sem merge e rodar **Re-run all jobs** do último run da `main`. O nome voltou e o job foi reconectado ao bundle.

**Três camadas:** a trava na UI, o `plan` denunciando a diferença no PR e o deploy desfazendo a mudança.

O `plan` em texto diz **que** o recurso muda, não **o que** muda. Para ver campo a campo: `databricks bundle plan -t hml -o json`.

### 15.2 Remoção: um PR que apaga um job

1. PR com `git rm bundles/vendas/resources/sample_job.job.yml`. O `plan` mostrou `delete jobs.sample_job` e `1 to delete`, e mesmo assim **o check ficou verde** e o merge ficou liberado.
2. **Descoberta no código do CLI:** o `deploy` sem `--auto-approve` só pede confirmação para apagar ou recriar recursos que guardam dados: schemas, pipelines, volumes, dashboards, Lakebase, vector search e Genie. **Jobs não estão na lista** ([código](https://github.com/databricks/cli/blob/v1.21.0/bundle/phases/deploy.go)). Um job removido do YAML é apagado em PRD sem pergunta.
3. **Correção (PR #6):** step no `pr-check` que lê `plan -o json` e falha se houver `delete` ou `recreate`. Para liberar uma remoção intencional, coloca-se o label **`permite-delete`** no PR. O workflow passou a reagir a `labeled` e `unlabeled`, e o step usa `set -o pipefail` para não passar em silêncio se o `plan` falhar.
4. **Teste:** sem o label, o check falhou com `O plano apaga ou recria: resources.jobs.sample_job` e o merge ficou bloqueado. Com o label, o check ficou verde. O PR foi fechado sem merge.

O label funciona como **registro de intenção**, não como controle de acesso: qualquer pessoa com permissão *triage* ou maior pode colocá-lo. Para exigir um aprovador específico, use `CODEOWNERS` em `resources/`.

### 15.3 Branch errada

1. **CLI:** numa branch `teste/branch-errada`, `databricks bundle plan -t prod` falhou com `not on the right Git branch: expected … main, actual: teste/branch-errada` antes de calcular qualquer plano. A checagem roda no `plan` e no `deploy` e só é ignorada com `--force` ([código](https://github.com/databricks/cli/blob/v1.21.0/bundle/config/mutator/validate_git_details.go)).
2. **Limite:** se não houver branch atual (*detached HEAD*, como num checkout de PR no CI), a checagem é pulada. Por isso a outra metade da proteção está no GitHub: o environment `prod` só aceita a `main` (passo 10).

### 15.4 Concorrência

Dois PRs (A e B) com mudanças só em comentários, mergeados em sequência. A observação mais útil veio no PRD:

- o `deploy-prod` do A, **parado só esperando aprovação**, já ocupava o grupo `vendas-prod`;
- o run do B mostrou *"A job in this workflow is waiting for deploy-prod (vendas #13) to complete before running"*;
- o botão **Review deployments** do B só apareceu depois que o A terminou.

**Ordem que o GitHub segue:** primeiro a fila de concorrência, depois as regras do environment (aprovação) e por fim a execução. As aprovações de prod chegam uma de cada vez e em ordem.

**Consequências:**

- uma aprovação esquecida trava os deploys seguintes;
- com o padrão `queue: single`, só um run fica pendente por grupo, e um terceiro cancela o pendente ([doc](https://docs.github.com/en/actions/concepts/workflows-and-actions/concurrency)). Para bundles isso é aceitável, porque o deploy mais novo publica o estado completo da `main`.

Se o time quiser todos os deploys em ordem, existe `queue: max` ([doc](https://docs.github.com/en/actions/how-tos/write-workflows/choose-when-workflows-run/control-workflow-concurrency)).

Detalhe do merge: o aviso *"Base branch was modified. Review and try the merge again"* no segundo PR não é bloqueio do ruleset. A página foi aberta antes do primeiro merge; basta recarregar.

### 15.5 Dados por ambiente, grants e destroy

1. **Notebook (PR #9):** passou a gravar `trips_amostra` (100 linhas de `samples.nyctaxi.trips`) no schema do ambiente. Antes ele só lia, e por isso os schemas estavam vazios.
2. **`grants` do schema nos targets `hml` e `prod`:** `USE_SCHEMA, SELECT, MODIFY, CREATE_TABLE` para o SP de execução do ambiente. Dev não precisa, porque lá você é o dono do schema.

   O `SHOW GRANTS` feito depois mostrou que os SPs de execução **já tinham esses privilégios no catálogo** desde o passo 5. No lab, os grants do schema são redundantes: o job gravaria mesmo sem eles. Eles ficam porque deixam a permissão do bundle versionada no código, e passam a ser necessários quando o catálogo der só `USE CATALOG` ao SP de execução, que é o desenho mais restrito.
3. **Resultado:**

   | Ambiente | Tabela | Gravada por |
   |---|---|---|
   | Dev | `dev.dev_<você>_vendas.trips_amostra` | você, pelo `bundle run` no Mac |
   | HML | `hml.hml_vendas.trips_amostra` | `sp-hml-run`, pelo CI depois do merge |
   | PRD | `prod.prd_vendas.trips_amostra` | `sp-prd-run`, com **Run now** manual |

4. **Leitura negada em HML** (*"Requires permission USE_SCHEMA on SCHEMA hml.hml_vendas"*): o dono do schema é o SP de deploy, e os `grants` do bundle **mandam na lista inteira**. Qualquer `GRANT` manual no schema some no próximo deploy ([código](https://github.com/databricks/cli/blob/v1.21.0/bundle/direct/dresources/grants.go)). A leitura para pessoas foi concedida no **catálogo**, que é fundação:

   ```sql
   GRANT USE SCHEMA, SELECT ON CATALOG hml TO `<você>`;
   GRANT USE SCHEMA, SELECT ON CATALOG prod TO `<você>`;
   ```

5. **Destroy e deploy em Dev:**

   ```bash
   databricks bundle destroy -t dev      # lista o que apaga e pede confirmação
   databricks bundle deploy -t dev
   databricks bundle run -t dev sample_job
   ```

   - O `destroy` apagou o job, o schema (com `force`, levando a tabela junto) e a pasta do bundle.
   - O `deploy` recriou o job **sem histórico de runs** e o schema **vazio**.
   - A `trips_amostra` só voltou com o `run`, porque quem cria os dados é o job, não o deploy.

---

## Passo 16: limpeza (pendente)

O ambiente foi mantido para a apresentação ao time. Quando for desmontar:

1. `bundle destroy` de HML e PRD **pelo CI**, com um workflow manual que use as credenciais de cada environment. Pela regra do lab, só os SPs mexem nesses ambientes.
2. `bundle destroy -t dev` no Mac.
3. Revogar os secrets OAuth dos SPs de deploy e apagar os secrets dos environments no GitHub.
4. Se for o fim definitivo, apagar a fundação listada em [`fundacao.md`](fundacao.md).

---

## Resumo dos erros e correções

| # | Sintoma | Causa | Correção |
|---|---|---|---|
| 2 | `Author identity unknown` | git sem identidade nesse computador | `git config` local com e-mail *noreply* |
| 2 | push `403` com outra conta | credencial errada no Keychain | remote por SSH |
| 3 | CLI antigo (v1.16.1) | — | `brew upgrade databricks/tap/databricks` |
| 5 | `catalogs create` falha pedindo Default Storage | limitação da Free Edition | criar o catálogo pela UI |
| 6 | `API is disabled for users without workspace-access` | SP sem entitlement | **Workspace access** no SP |
| 8 | `Library installation failed … file:///Workspace/…/files` | `--editable ..` no notebook do template | dependências `[]` |
| 9→12 | pastas `files` de HML/PRD com dono pessoa | `validate` local cria a pasta | apagar e redeployar pelo CI; `validate` de HML/PRD só no CI |
| 12 | `Cannot bind the service principal provided in 'run_as'` | SP de deploy sem **Use** no SP de run | dar **Use** ao SP de deploy |
| 14 | check obrigatório preso em *Waiting for status* | `paths` no `pull_request` | tirar o `paths` do `pull_request` |
| 15.2 | delete de job passa com check verde | jobs fora da lista de confirmação do CLI | trava no `pr-check` + label `permite-delete` |
| 15.5 | pessoa sem leitura em HML/PRD | grants do schema são do bundle | `GRANT` de leitura no catálogo |
| — | `fatal: only one reference expected` | comentário `#` na mesma linha do comando no zsh | não colar comentários na linha do comando |
