# Roteiro da demo ao vivo

*Por [Thomaz Antonio Rossito Neto](../README.md#autor), CI&T.*

**Duração:** 20 a 25 minutos, depois da Parte 1 do deck (ou no lugar da seção "Laboratório", slides 36 a 42).
**Público:** o time todo. Quem configura vai querer ver o YAML; quem só usa vai querer ver o caminho do PR até PRD.
**Mensagem central:** em Dev, cada um na sua pasta; em HML e PRD, só o pipeline, e nada depende de uma pessoa.

---

## Antes da reunião (15 minutos)

**Estado do ambiente:**

- [ ] `main` local igual à do GitHub (`git switch main && git pull`).
- [ ] **Nenhum deploy de prod esperando aprovação** em Actions. Um run parado na aprovação segura a fila de `vendas-prod`, e a demo trava.
- [ ] Nenhum PR aberto além dos de demo.
- [ ] `databricks auth login` válido no seu computador (Dev).

**Branch da trava de delete, preparada antes** (o PR fica aberto até a hora da demo):

```bash
git switch main
git pull
git switch -c demo/remover-job
git rm bundles/vendas/resources/sample_job.job.yml
git commit -m "demo: remover o sample_job"
git push -u origin demo/remover-job
```

Abra o PR e confira que o `pr-check` ficou **vermelho**. Não faça merge.

**Abas abertas, nesta ordem:**

1. Repositório no GitHub (Code)
2. GitHub → Actions
3. GitHub → Settings → Environments → `prod`
4. Databricks → Jobs & Pipelines
5. Databricks → Catalog (expandir `dev`, `hml` e `prod`)
6. Databricks → Workspace → `Homologacao/.bundle/vendas` e `Production/.bundle/vendas`
7. Terminal em `~/databricks/dabs-cicd-lab`
8. Editor com `bundles/vendas/databricks.yml` aberto

---

## Roteiro

### 1. A pergunta (1 min)

**Mostrar:** o `databricks.yml` no editor, nos três targets.

**Falar:** "Quando eu faço deploy, ele vai para a minha pasta. Isso está certo em Dev. A pergunta é como isso funciona com 20 pessoas e três ambientes. A resposta está nesses três blocos: `dev` é de cada um; `hml` e `prod` têm pasta do ambiente, `run_as` com service principal e permissões que não citam ninguém do time."

### 2. Dev: cada um na sua pasta (3 min)

**Fazer:**

```bash
git switch -c demo/tag-time
cd bundles/vendas
```

No `resources/sample_job.job.yml`, logo abaixo de `name: sample_job`, adicionar:

```yaml
      tags:
        time: vendas
```

Depois, ainda em `bundles/vendas`:

```bash
databricks bundle deploy -t dev
databricks bundle run -t dev sample_job
```

**Mostrar:**

- Jobs & Pipelines: `[dev <você>] sample_job`, rodando como você.
- Workspace: `/Users/<você>/.bundle/vendas/dev`.
- Catalog: `dev.dev_<você>_vendas.trips_amostra`.

**Falar:** "Se outra pessoa rodar o mesmo comando, ela ganha a própria cópia: outro prefixo, outra pasta e outro schema. Ninguém pisa em ninguém."

### 3. PR e plan (4 min)

**Fazer:**

```bash
cd ../..
git commit -am "feat: tag time=vendas no job"
git push -u origin demo/tag-time
```

Abrir o PR e esperar o `pr-check` (cerca de 10 s). Abrir o step `databricks bundle plan -t hml`.

**Mostrar:** `update jobs.sample_job` e `Plan: 0 to add, 1 to change, 0 to delete`.

**Falar:**

- "O plan é a parte mais importante do review. Ele diz o que vai acontecer em HML antes de acontecer."
- "Ele mostra recursos, não código: mudança de notebook se revisa em *Files changed*."
- "O botão de merge só libera com o `pr-check` verde. Ninguém faz push direto na `main`, nem eu, que sou admin."

### 4. Merge e HML (3 min)

**Fazer:** merge do PR. Ir para Actions.

**Mostrar:**

- `deploy-hml` rodando, com deploy e run do job.
- Jobs: `[hml] sample_job`, *Run as* `sp-hml-run`, com a tag `time: vendas`.
- Workspace: pastas em `Homologacao/.bundle/vendas` com dono `sp-lab-deploy`.

**Falar:** "Quem publicou foi o service principal de deploy. Quem executou foi o de execução. Se eu sair do time amanhã, nada aqui para."

### 5. Aprovação e PRD (3 min)

**Fazer:** no run, clicar em **Review deployments** → `prod` → **Approve and deploy**.

**Mostrar:**

- Settings → Environments → `prod`: required reviewer e a regra de branch `main`.
- O step `bundle plan -t prod` e o deploy.
- Jobs: `[prd] sample_job`, *Run as* `sp-prd-run`, com o agendamento pausado.

**Falar:**

- "PRD só aceita deploy da `main` e com aprovação. São duas travas: uma no bundle (`git.branch`) e outra no GitHub (o environment)."
- "O CI só publica em PRD, não executa. Quem roda PRD é o agendamento, que no lab está pausado."

### 6. Drift: alguém mexe na UI (2 min)

**Mostrar:** abrir `[prd] sample_job` e tentar editar. Aparece a faixa *Connected to Declarative Automation Bundles* e o editor travado.

**Falar:** "Para mexer, alguém precisa desconectar o job de propósito. E mesmo assim o próximo `plan` mostra a diferença no PR, e o próximo deploy desfaz. O Git é a fonte da verdade." Se houver tempo, refaça o teste 15.1 do [passo a passo](lab-passo-a-passo.md#151-drift-editar-um-job-pela-ui).

### 7. Um PR que apaga um job (3 min)

**Mostrar:** o PR `demo/remover-job`, preparado antes:

- o check vermelho, com a mensagem `O plano apaga ou recria: resources.jobs.sample_job`;
- o botão de merge bloqueado.

**Falar:**

- "O CLI pede confirmação para apagar schema, pipeline e volume, mas não para job. Sem esta trava, esse PR passaria verde e o job sumiria de PRD."
- "Se a remoção for de propósito, a pessoa coloca o label `permite-delete`. Fica registrado no PR quem liberou."

**Opcional:** colocar o label ao vivo e mostrar o check ficando verde.

### 8. Branch errada (1 min)

**Fazer:** no terminal, ainda na branch `demo/tag-time` (ou em qualquer branch que não seja a `main`):

```bash
cd bundles/vendas
databricks bundle plan -t prod
```

**Mostrar:** `Error: not on the right Git branch: expected … main`.

### 9. Fechamento (2 min)

**Mostrar:** o slide "A infraestrutura volta com o deploy. Os dados, não." e a lista de [fundação](fundacao.md).

**Falar:**

- "O bundle cuida do job, do schema e das permissões dele."
- "Catálogo, service principal, secret e GitHub são fundação: ficam com a plataforma."
- "E dado não é código: um `destroy` apaga as tabelas, e só o job traz de volta."

---

## Se algo der errado ao vivo

| Situação | O que fazer |
|---|---|
| Actions demora para começar | Mostrar um run anterior completo (Actions → filtrar por `main`) enquanto espera |
| `deploy-prod` não pede aprovação | Conferir se há outro run de prod parado esperando aprovação; aprovar ou cancelar esse antes |
| Check obrigatório preso em *Waiting for status* | Conferir se alguém recolocou `paths` no `pull_request` do workflow. Com o filtro, PRs fora do bundle não disparam o check |
| `bundle deploy -t dev` pede login | `databricks auth login --host <host>` |
| Sem internet ou workspace fora do ar | Seguir o [passo a passo](lab-passo-a-passo.md) e os slides da seção "Laboratório" do deck (36 a 42) |

---

## Perguntas que costumam aparecer

| Pergunta | Resposta curta |
|---|---|
| "Posso rodar `deploy -t hml` no meu computador para testar?" | Não. HML é publicado só pelo CI. Para testar, use Dev. Até o `validate` de HML cria pasta com a sua identidade. |
| "Por que o secret do SP fica no GitHub? Não era para ser sem senha?" | No time real, sim: OIDC, sem secret guardado. A Free Edition não tem console de conta para criar a federation policy, por isso o lab usa OAuth M2M com secret. |
| "E se o service principal perder o acesso?" | O deploy falha no CI, à vista de todo mundo, e não depende de alguém lembrar a própria senha. A correção é na fundação, não no bundle. |
| "Quem aprova PRD?" | Os *required reviewers* do environment `prod`. No time, um grupo, e não uma pessoa. |
| "Dois merges ao mesmo tempo dão problema?" | Não. O `concurrency` coloca o segundo deploy na fila. Uma aprovação de prod esquecida segura a fila, então aprove ou rejeite logo. |
| "E se eu precisar de um hotfix?" | Ele passa pelo mesmo PR, só que pequeno e com revisão rápida. |
| "Se eu der `destroy`, perco o quê?" | Job (o ID muda e o histórico some), schema com as tabelas e arquivos. Os dados só voltam quando o job roda de novo. |
| "Como fica com vários bundles?" | Uma pasta por bundle em `bundles/`, um workflow por bundle com filtro de `paths` no `push`, e `CODEOWNERS` por pasta. |

---

## Depois da demo

```bash
git switch main
git pull
git branch -D demo/remover-job demo/tag-time
```

No GitHub:

- feche o PR `demo/remover-job` **sem merge**;
- apague as duas branches;
- tire o label `permite-delete` do PR, se foi colocado.

A tag `time: vendas` pode ficar no job. Se quiser tirar, faça isso por PR, como qualquer outra mudança.
