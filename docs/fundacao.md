# Fundação: o que existe fora do bundle

*Por [Thomaz Antonio Rossito Neto](../README.md#autor), CI&T.*

O bundle `vendas` cria e mantém só a carga de trabalho:

- o job;
- o schema e os grants do schema;
- os arquivos em `root_path`.

Tudo o que está nesta página foi feito à mão no laboratório. Nem `bundle deploy` nem `bundle destroy` mexem nisso. Se o ambiente for desmontado, é esta lista que precisa ser refeita, **na ordem abaixo**, antes do primeiro deploy pelo CI.

Num time real, esta camada vai para Terraform ou scripts versionados, num repositório de plataforma. Aqui ela está documentada para ser refeita à mão.

Nos comandos, `<host>` é a URL do workspace, `<app-id>` é o *application ID* de um service principal, `<id-sp>` é o ID numérico dele no workspace e `<você>` é o seu usuário.

---

## Ordem de criação

| # | Item | Onde | Depende de |
|---|---|---|---|
| 1 | Catálogos `dev`, `hml`, `prod` | Databricks | — |
| 2 | Service principals | Databricks | — |
| 3 | Entitlement `workspace-access` nos SPs de deploy | Databricks | 2 |
| 4 | Permissão **Use** do SP de deploy no SP de run | Databricks | 2 |
| 5 | Grants nos catálogos | Databricks | 1, 2 |
| 6 | Secrets OAuth dos SPs de deploy | Databricks | 2, 3 |
| 7 | Repositório, environments, variables e secrets | GitHub | 6 |
| 8 | Ruleset da `main` e label `permite-delete` | GitHub | 7 (o check `pr-check` precisa ter rodado uma vez) |

---

## 1. Catálogos

**Pela UI:** Catalog → **Create catalog** → nome `dev`, `hml` ou `prod` → **Default Storage**.

Na Free Edition, `databricks catalogs create` falhou pedindo o Default Storage (*"Metastore storage root URL does not exist"*). Pela UI funcionou.

## 2. Service principals

| Nome | Papel | Ambiente |
|---|---|---|
| `sp-lab-deploy` | deploy (CI) | HML |
| `sp-hml-run` | execução dos jobs | HML |
| `sp-prd-deploy` | deploy (CI) | PRD |
| `sp-prd-run` | execução dos jobs | PRD |

**Pela UI:** Settings → Identity and access → Service principals → **Add service principal**.

**Pela CLI**, como admin do workspace:

```bash
databricks service-principals create --display-name sp-hml-run
```

Se os SPs forem recriados, os *application IDs* mudam. Atualize `run_as` e `permissions` em `bundles/vendas/databricks.yml` e a variable `DATABRICKS_CLIENT_ID` dos environments.

## 3. Entitlement nos SPs de deploy

Nos SPs `sp-lab-deploy` e `sp-prd-deploy`: marcar **Workspace access** nos entitlements.

Sem isso, o login M2M funciona, mas qualquer chamada responde *"API is disabled for users without workspace-access"*.

## 4. Permissão Use (servicePrincipal.user)

O SP de deploy precisa poder "usar" o SP de execução, senão o deploy com `run_as` falha com *"Cannot bind the service principal provided in 'run_as'"*.

| No SP | Dar **Use** para |
|---|---|
| `sp-hml-run` | `sp-lab-deploy` |
| `sp-prd-run` | `sp-prd-deploy` |

**Pela UI:** Service principals → `<sp de run>` → **Permissions** → **Grant access** → `<sp de deploy>` → **Use**.

Erro que já aconteceu no lab: dar o **Use** ao próprio SP de run. A permissão tem que ir para o SP que **faz o deploy**.

## 5. Grants nos catálogos

Estado conferido com `SHOW GRANTS ON CATALOG` em 17/10/2026. Nos comandos SQL, um service principal é identificado pelo *application ID*, entre crases.

```sql
-- HML
GRANT USE CATALOG, CREATE SCHEMA
  ON CATALOG hml TO `<app-id do sp-lab-deploy>`;
GRANT USE CATALOG, USE SCHEMA, SELECT, MODIFY, CREATE TABLE
  ON CATALOG hml TO `<app-id do sp-hml-run>`;

-- PRD
GRANT USE CATALOG, CREATE SCHEMA
  ON CATALOG prod TO `<app-id do sp-prd-deploy>`;
GRANT USE CATALOG, USE SCHEMA, SELECT, MODIFY, CREATE TABLE
  ON CATALOG prod TO `<app-id do sp-prd-run>`;

-- Leitura para pessoas (no lab, um e-mail; no time, um grupo)
GRANT USE SCHEMA, SELECT ON CATALOG hml  TO `<você>`;
GRANT USE SCHEMA, SELECT ON CATALOG prod TO `<você>`;
```

| Catálogo | Principal | Privilégios |
|---|---|---|
| `dev` | `account users` | `BROWSE` (padrão) |
| `hml` | `sp-lab-deploy` | `USE CATALOG`, `CREATE SCHEMA` |
| `hml` | `sp-hml-run` | `USE CATALOG`, `USE SCHEMA`, `SELECT`, `MODIFY`, `CREATE TABLE` |
| `hml` | `<você>` | `USE SCHEMA`, `SELECT` |
| `hml` | `account users` | `BROWSE` (padrão) |
| `prod` | `sp-prd-deploy` | `USE CATALOG`, `CREATE SCHEMA` |
| `prod` | `sp-prd-run` | `USE CATALOG`, `USE SCHEMA`, `SELECT`, `MODIFY`, `CREATE TABLE` |
| `prod` | `<você>` | `USE SCHEMA`, `SELECT` |
| `prod` | `account users` | `BROWSE` (padrão) |

Para conferir:

```sql
SHOW GRANTS ON CATALOG dev;
SHOW GRANTS ON CATALOG hml;
SHOW GRANTS ON CATALOG prod;
```

Notas:

- Em `dev` não há grant extra. Quem cria o catálogo é o dono dele, e o schema de Dev é criado pela própria pessoa.
- A leitura para pessoas fica **no catálogo** de propósito. Os `grants` do schema estão no bundle, e o bundle remove do schema qualquer grant que não esteja no YAML.
- Os privilégios do SP de execução no catálogo já cobrem a escrita em qualquer schema dele. Por isso, no lab, os `grants` do schema no bundle são redundantes. No desenho mais restrito, o catálogo dá só `USE CATALOG` ao SP de execução, e o bundle concede o resto schema a schema.
- No time, a leitura vai para grupos de conta, e em PRD com critério: quem faz deploy não precisa ler dado de produção.

## 6. Secrets OAuth

**Pela UI:** Service principals → `sp-lab-deploy` → **Secrets** → **Generate secret**. O mesmo vale para o `sp-prd-deploy`.

**Pela CLI:**

```bash
databricks service-principal-secrets-proxy create <id-sp>
```

O secret aparece uma vez só. Ele vai direto para o environment do GitHub (passo 7) e não fica salvo em arquivo, no chat nem em print.

Para testar antes de colocar no GitHub (zsh):

```bash
read -s "SECRET?Cole o secret do SP: "
echo
DATABRICKS_HOST=<host> DATABRICKS_AUTH_TYPE=oauth-m2m \
DATABRICKS_CLIENT_ID=<app-id> DATABRICKS_CLIENT_SECRET="$SECRET" \
databricks current-user me
unset SECRET
```

## 7. GitHub: repositório e environments

**Repositório:** no GitHub Free, só em repositório **público** funcionam os required reviewers e as regras de branch dos environments. Num plano pago, use repositório privado.

**Environments** (Settings → Environments):

| Environment | Required reviewers | Deployment branches and tags |
|---|---|---|
| `hml` | — | *No restriction*, porque o `pr-check` usa este environment a partir de branches de PR |
| `prod` | você (no time: o grupo que aprova PRD) | *Selected branches and tags* → `main` |

**Variables e secrets:** os nomes são os mesmos nos dois environments, o valor muda.

| Nome | Tipo | `hml` | `prod` |
|---|---|---|---|
| `DATABRICKS_HOST` | variable | `<host>` | `<host>` |
| `DATABRICKS_CLIENT_ID` | variable | app-id do `sp-lab-deploy` | app-id do `sp-prd-deploy` |
| `DATABRICKS_CLIENT_SECRET` | secret | secret do `sp-lab-deploy` | secret do `sp-prd-deploy` |

**Pela CLI do GitHub.** O `gh secret set` sem `--body` pede o valor no terminal, sem deixar o secret no histórico do shell:

```bash
gh variable set DATABRICKS_HOST      --env hml --body "<host>"
gh variable set DATABRICKS_CLIENT_ID --env hml --body "<app-id do sp-lab-deploy>"
gh secret   set DATABRICKS_CLIENT_SECRET --env hml

gh variable set DATABRICKS_HOST      --env prod --body "<host>"
gh variable set DATABRICKS_CLIENT_ID --env prod --body "<app-id do sp-prd-deploy>"
gh secret   set DATABRICKS_CLIENT_SECRET --env prod
```

Os environments e os reviewers são criados pela UI.

## 8. Ruleset da `main` e label

**Ruleset** (Settings → Rules → Rulesets → **New branch ruleset**):

| Campo | Valor |
|---|---|
| Ruleset name | `main-protegida` |
| Enforcement status | Active |
| Bypass list | vazia |
| Target branches | Include default branch |
| Restrict deletions | sim |
| Block force pushes | sim |
| Require a pull request before merging | sim, com 0 aprovações no lab (no time: 1 ou mais, com `CODEOWNERS`) |
| Require status checks to pass | `pr-check`, com a fonte **GitHub Actions** |

O check `pr-check` só aparece na busca do ruleset depois de ter rodado pelo menos uma vez num PR.

**Label** que libera remoções no PR:

```bash
gh label create permite-delete --description "Libera delete/recreate no plan do PR" --color D93F0B
```

---

## Checklist de verificação

Depois de refazer a fundação, antes de confiar na esteira:

- [ ] `SHOW GRANTS` dos três catálogos bate com a tabela do item 5.
- [ ] Teste M2M de cada SP de deploy (`databricks current-user me`) responde com o nome do SP.
- [ ] Em cada SP de run, a aba **Permissions** mostra o SP de deploy com **Use**.
- [ ] `databricks bundle deploy -t dev` e `bundle run -t dev sample_job` funcionam no seu computador.
- [ ] Um PR qualquer roda o `pr-check` e mostra o `plan` de HML.
- [ ] Depois do merge, `[hml] sample_job` aparece com *Run as* `sp-hml-run`, e as pastas em `/Workspace/Homologacao/.bundle/vendas` têm o `sp-lab-deploy` como dono.
- [ ] A aprovação de prod publica `[prd] sample_job` com *Run as* `sp-prd-run`, e as pastas em `/Workspace/Production/.bundle/vendas` têm o `sp-prd-deploy` como dono.
- [ ] Um push direto na `main` é recusado.

**Não rode `bundle validate` ou `bundle plan` de HML/PRD no seu computador** durante a recriação. Na primeira vez, o `validate` cria a pasta `files` do ambiente com a sua identidade.

---

## Ponto não testado

No lab, as pastas de primeiro nível `/Workspace/Homologacao` e `/Workspace/Production` foram criadas pelo `validate` local do passo 9, com o usuário humano. **Não foi testado** se um SP de deploy, sem ser admin, consegue criar uma pasta direto em `/Workspace` no primeiro deploy de um ambiente novo.

Se o primeiro deploy falhar ao criar o `root_path`:

1. crie `/Workspace/Homologacao` e `/Workspace/Production` como admin;
2. dê **Can manage** nelas ao SP de deploy do ambiente;
3. rode o deploy de novo pelo CI.
