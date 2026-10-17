# Bundle `vendas`

Bundle de exemplo do laboratório. Ele foi gerado pelo template `default-python` e depois reduzido ao que o lab usa: um job com uma task de notebook (serverless) e um schema.

| Arquivo | Conteúdo |
|---|---|
| `databricks.yml` | nome do bundle, variável `catalog` e os targets `dev`, `hml` e `prod` |
| `resources/sample_job.job.yml` | job `sample_job`: recebe `catalog` e `schema` como parâmetros e roda o notebook |
| `resources/vendas.schema.yml` | schema `vendas` no catálogo do ambiente |
| `src/sample_notebook.ipynb` | grava 100 linhas de `samples.nyctaxi.trips` em `trips_amostra` |

## O que você roda

Só o target `dev`. HML e PRD são publicados pelo GitHub Actions ([`.github/workflows/vendas.yml`](../../.github/workflows/vendas.yml)).

```bash
cd bundles/vendas
databricks bundle validate -t dev
databricks bundle deploy -t dev
databricks bundle run -t dev sample_job
```

O deploy em `dev` vai para `/Workspace/Users/<você>/.bundle/vendas/dev`. O job aparece como `[dev <você>] sample_job`, e a tabela é gravada em `dev.dev_<você>_vendas.trips_amostra`.

Para apagar a sua cópia de Dev (job, schema **com as tabelas** e arquivos):

```bash
databricks bundle destroy -t dev
```

## Não faça

- `databricks bundle deploy -t prod` no seu computador. O target `prod` só aceita a branch `main` (`git.branch`), e a regra do time é que só o pipeline publica.
- `databricks bundle validate -t hml` ou `-t prod` no seu computador. Na primeira vez, o `validate` cria a pasta `files` do ambiente com a sua identidade.
- `GRANT` manual no schema de HML ou PRD. Os `grants` do schema estão no `databricks.yml`, e o próximo deploy remove o que não estiver lá.

Visão geral, ambientes e proteções: [README da raiz](../../README.md).
