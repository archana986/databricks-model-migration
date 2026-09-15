# Sage Review Fix: cleanup scoping + flavor-agnostic import/validate

**Raised by:** Rouzbeh Rahimi (Sage asset review).
**Scope:** two issues in the Workspace-to-Workspace / cross-catalog Model Migration bundle.
**Tested:** end to end in the fevm Azure workspace (`adb-7405609619727450`), catalog `akrishn_fe_dsa`, schemas `mm_sage_src` -> `mm_sage_tgt`, on serverless. Every cell output was reviewed; 0 errors. Transcripts in `e2e_evidence/`.

## Issue 1: `08_cleanup_target_models` was an unscoped delete

Before, cleanup deleted every version of any name-matching model (plus aliases, model, runs, experiment) with no check for migration tags, so pointing it at a schema that already held a real model of the same name destroyed that real model.

Fix: a new `require_migration_tag` widget (default `true`). Cleanup now deletes only versions positively confirmed as migration-created (a `migration_source_model` tag on the model version, or a `migration.source_model` tag on the source run); protects and reports untagged versions; only drops an alias that points at a deleted version; and drops the registered model only when nothing real remains and it actually removed migration versions (so a model it did not touch is never deleted in safe mode). `require_migration_tag=false` restores the full wipe as an explicit opt-in.

## Issue 2: sklearn assumption in import and validate

Before, `05_import` loaded and re-logged through `mlflow.sklearn` and only fell back to an artifact copy by matching the **error message text** (fragile: it triggered only when the error string happened to contain "sklearn" and "flavor"/"no module"). `06_validate` ran its inference smoke test with `mlflow.sklearn.load_model` and sized input from `n_features_in_`. Non-sklearn models (custom pyfunc, xgboost, etc.) could fail to import or migrate correctly but falsely report `Inference: FAIL`.

Fix:
- `05_import` now detects the flavor up front from the `MLmodel` file. sklearn models keep the native sklearn re-log; every other flavor is deep-cloned as-is (`log_artifacts` + `register_model`) so it keeps its real flavor. It also stamps the migration marker on the model version (underscore key, since UC version-tag keys cannot contain dots) so cleanup can scope reliably. The UC-grant preservation cell is unchanged.
- `06_validate` loads the smoke test via `mlflow.pyfunc.load_model` (works for every flavor), builds input from the model signature, and reports `SKIP` (not `FAIL`) when there is no signature. The source-vs-target version match is guarded so a metadata version lacking a source id cannot false-match the first target version.

## End-to-end proof (this variant, two-schema chain)

Real chain `03_export -> 04_transfer -> 05_import -> 06_validate`, then scoped cleanup, on synthetic sklearn (2 versions + Champion/Challenger), custom pyfunc, and xgboost models, plus a `protectme` model (real v1 + migration-tagged v2) and a `wipeme` model.

- Export: sk_model 2, pf_model 1, xgb_model 1; grants exported.
- Import: `TOTAL ok=4 fail=0`; flavors reported `sklearn`, `pyfunc`, `xgboost`; aliases restored; grants applied=0 (none set).
- Validate: every model Version count PASS, Metrics/Params/Lineage full, and `Inference: PASS` for sklearn, pyfunc, and xgboost.
- Scoped cleanup on `protectme`, `wipeme` (`require_migration_tag=true`): `protectme` v2 (migration) deleted, v1 + Champion kept, model kept; `wipeme` untouched.
- Independent re-read: `protectme` = `v1[real]` + Champion; `wipeme` = `v1[real]`; migrated models tagged `[migr]` with aliases.

See `e2e_evidence/sage_*.txt` for full per-cell transcripts. A workspace re-test with screenshots is in `docs/WORKSPACE_PROOF.md` (added after the bundle was deployed and run).
