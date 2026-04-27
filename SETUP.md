# Setup and Run

Step-by-step guide to deploy and run the UC Model Migration bundles.

## Prerequisites

1. **Databricks CLI** v0.278.0 or higher.

   ```bash
   databricks --version
   ```

2. **Same metastore**: Source and target catalogs must be on the **same Unity Catalog metastore**.

3. **Permissions**: Your user must be able to:
   - Read registered models and experiments in the source catalog.
   - Manage permissions on registered models in the source catalog (read direct grants).
   - Create / use the target catalog, schema, and volume.
   - Run jobs in both workspaces.

4. **Python packages** are installed automatically by the bundle's job environment (`scikit-learn`, `mlflow[databricks]`).

## 1. Authenticate

Run the relevant `databricks auth login` command for each workspace you'll use.

```bash
# Source workspace
databricks auth login -p YOUR_SOURCE_PROFILE --host https://<your-source-workspace>.cloud.databricks.com

# Target workspace (skip if same as source)
databricks auth login -p YOUR_TARGET_PROFILE --host https://<your-target-workspace>.cloud.databricks.com
```

Verify:

```bash
databricks current-user me -p YOUR_SOURCE_PROFILE
databricks current-user me -p YOUR_TARGET_PROFILE
```

OAuth or Azure CLI is preferred. PAT works but is not required.

## 2. Configure your bundles

Each bundle reads two YAML files:

- `databricks.yml` — public defaults (committed to git, uses placeholder values).
- `databricks.local.yml` — your real values (gitignored, you create this).

Copy the examples and edit:

```bash
cp source/databricks.local.yml.example source/databricks.local.yml
cp target/databricks.local.yml.example target/databricks.local.yml
```

Edit each `databricks.local.yml` and set:

| Field | Set to |
|---|---|
| `variables.source_catalog.default` | Your source UC catalog name |
| `variables.target_catalog.default` | Your target UC catalog name |
| `variables.source_schema.default` | Schema in the source catalog where models live |
| `variables.target_schema.default` | Schema in the target catalog where models will be registered |
| `variables.model_names.default` | Comma-separated model short names (e.g. `churn_model,fraud_model`) |
| `variables.export_volume.default` | Volume name for staging exports (default `model_exports`) |
| `variables.import_volume.default` | Volume name for staging imports (target bundle only; default `model_imports`) |
| `variables.experiment_prefix.default` | Prefix for the per-model migration experiment in the target (target bundle only). Default `migration_` (e.g. experiment `migration_churn_model`). Set to `""` to use the bare model short name (e.g. experiment `churn_model`). |
| `targets.dev.workspace.profile` | Your Databricks CLI profile name |
| `targets.dev.workspace.host` | Your workspace URL |

`source_schema` and `target_schema` may be the same name (mirror) or different (rename schema during migration).

## 3. Validate and deploy

From the repo root:

```bash
# Source bundle
cd source
databricks bundle validate -p YOUR_SOURCE_PROFILE
databricks bundle deploy   -p YOUR_SOURCE_PROFILE

# Target bundle
cd ../target
databricks bundle validate -p YOUR_TARGET_PROFILE
databricks bundle deploy   -p YOUR_TARGET_PROFILE
```

`validate` catches missing variables and YAML errors before deploy.

## 4. Run the four jobs in order

```bash
# 1. Clean the source export volume (safe to skip on the very first run)
cd source && databricks bundle run src_model_migration_cleanup -p YOUR_SOURCE_PROFILE

# 2. Export models, metadata, artifacts, and grants to the export volume
cd source && databricks bundle run src_model_export -p YOUR_SOURCE_PROFILE

# 3. Clean the target import volume and remove any prior migrated models
cd ../target && databricks bundle run tgt_model_migration_cleanup -p YOUR_TARGET_PROFILE

# 4. Transfer, import, validate, and reconcile
cd ../target && databricks bundle run tgt_model_migration_register -p YOUR_TARGET_PROFILE
```

**Always run source jobs before target jobs.** Step 2 must finish before step 4.

## 5. Verify on the target

```sql
-- Models registered in the target schema
SHOW MODELS IN <target_catalog>.<target_schema>;

-- Versions of a specific model
SELECT name, version, current_stage, run_id
FROM <target_catalog>.information_schema.model_versions
WHERE model_name = '<model_name>';

-- Migrated grants on a target model
SHOW GRANTS ON MODEL <target_catalog>.<target_schema>.<model_name>;
```

The `reconciliation` task in step 4 also prints a per-model table comparing source and target version counts, alias mappings, and a final `ALL OK` / `MISMATCH` line.

## Compute: serverless vs classic

Each job defaults to **serverless** (`environment_key: migration_env`). To switch a task to a classic cluster, edit the job YAML in `resources/`:

```yaml
# Replace
environment_key: migration_env

# With
job_cluster_key: migration_cluster
```

Then add a `job_clusters` section with `spark_version`, `node_type_id`, and other cluster settings. Notebooks are compatible with both — they use Python variables for config, not `spark.conf.set()`.

## Job reference

| Job | Bundle | Tasks |
|---|---|---|
| `src_model_migration_cleanup` | source | `cleanup_export_volume` |
| `src_model_export` | source | `export` (artifacts + metadata + grants → export volume) |
| `tgt_model_migration_cleanup` | target | `cleanup_volume` → `cleanup_models` |
| `tgt_model_migration_register` | target | `transfer` → `import_register` → `validate` → `reconciliation` |

## Limitations

- **Same metastore only.** Cross-metastore and cross-cloud are not supported.
- **Source models are read-only.** No job touches source registered models, versions, aliases, or experiments.
- **Direct grants only.** Direct UC grants on the source model are migrated. Inherited grants from the parent catalog or schema are not exported — they will inherit naturally from whatever the target schema/catalog grants. Principals that don't exist in the target metastore are skipped with a warning, not a failure.
- **Source experiments are not copied.** The target gets a fresh experiment to house the re-logged versions, named `<experiment_prefix><model_name>` (default prefix `migration_`, configurable; set to `""` for `<model_name>`).
- **Each migrated version gets a new run ID and version number** in the target. The originals are tracked via `migration.source_run_id` and `migration.source_version` tags.
- **Target cleanup is destructive.** `tgt_model_migration_cleanup` deletes all model versions, aliases, registered model entries, the migration experiment, and import volume contents for the configured `model_names`.
- **Volume paths only.** Uses `/Volumes/<catalog>/<schema>/<volume>/` — no DBFS or external paths.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `validate` fails with "variable not set" | Check `databricks.local.yml` exists and overrides every variable, or that defaults in `databricks.yml` are set. |
| `deploy` fails with "host not configured" | Set `targets.dev.workspace.host` and `profile` in `databricks.local.yml`. |
| `import_register` fails with `Model does not have the "sklearn" flavor` | Already handled — the import notebook automatically falls back to artifact-copy for non-sklearn flavors (Feature Store, pyfunc, custom). If you still see this, re-deploy the bundle. |
| `cleanup` reports "skip not found" | Expected on first run — nothing to clean. |
| Target grants don't match source | Ensure the principals on the source model also exist in the target metastore. Cross-metastore principals are skipped with a warning. |
