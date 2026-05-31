# ADR 0001: Reproducibility strategy for NorthStar models

## Context

NorthStar Logistics currently stores trained models in S3 under manually chosen names such as `eta_v2_FINAL.onnx`, with no record of the training code version, dataset snapshot, or environment used to produce them. This makes it impossible to audit a live model's provenance, diagnose a regression, or retrain deterministically after a production incident.

## Decision

**Environment.** Every training job runs inside a Docker image whose `sha256` digest is recorded in the model's lineage metadata (`environment_image_digest`). Images are built from a pinned `requirements.txt` (exact package versions, no ranges) committed to the repo, and pushed to an internal container registry where tags are immutable. The digest — not the tag — is what gets stored, so a rebuild of the same tag cannot silently change the environment.

**Data.** Training datasets are materialized as immutable snapshots in S3 under a path that includes a content-addressable SHA-256 hash (e.g. `s3://northstar-data/snapshots/<dataset_hash>/train.parquet`). The hash is computed over the full file contents at snapshot time using DVC `dvc run` or, if DVC is not adopted, a lightweight convention: a `manifest.json` alongside each snapshot listing each file's SHA-256 and byte count. The `dataset_hash` field is required in every training run's lineage metadata and in the registry; a run that cannot supply it is rejected at the staging gate.

**Code.** All training code lives in Git. The exact `git_sha` of the commit used for a training run is captured by the CI/CD pipeline before the job starts and written into the MLflow run's tags. No training run may be promoted if `git_sha` resolves to a dirty working tree or an uncommitted local change. The MLflow `training_run_id` links back to the full parameter, metric, and artifact log, meaning the code + config state is recoverable from a single identifier.

**Randomness.** A fixed integer `random_seed` is passed explicitly to every RNG call (NumPy, scikit-learn, PyTorch/ONNX export). The seed value is stored in the MLflow run parameters and in the lineage `random_seed` field. Where the training framework supports it (PyTorch with `torch.use_deterministic_algorithms(True)`), full determinism is enabled. Where hardware non-determinism cannot be eliminated (e.g. GPU atomics), the seed still ensures that re-runs on identical hardware produce results within an acceptable tolerance window, which is documented per model.

## Alternatives rejected

- **Rely on S3 object timestamps instead of dataset hashes.** S3 timestamps are mutable (a re-upload with the same key silently overwrites them), are not content-addressable, and do not survive bucket lifecycle policies. A timestamp cannot prove two training runs saw identical data.
- **Keep model versions only as manually named ONNX files** (the current state). Human-chosen names like `eta_v2_FINAL.onnx` carry no provenance: they record neither the code commit, the dataset, nor the environment that produced them. Debugging a regression requires guesswork about which "FINAL" is actually live.
- **Skip random seed recording because models are "close enough."** Even small metric differences caused by non-determinism can flip a gate check, making it appear that a code change improved or degraded the model when in fact randomness was the cause. Without a fixed seed, A/B comparisons between runs are unreliable.

## Consequences

- Every training run must emit seven lineage fields (`git_sha`, `dataset_hash`, `training_run_id`, `feature_schema_version`, `model_artifact_uri`, `environment_image_digest`, `random_seed`) before it can reach staging; this adds a small metadata-capture step to the training pipeline.
- Dataset snapshots must be written to immutable S3 paths and never overwritten; the team commits to a storage budget for retaining at least the last 10 dataset snapshots per model alongside the 10 retained model versions.
- Promotion to production is slightly slower because the 24-hour canary soak and manual approval gate require human action, but rollback to any of the 10 retained versions is a single registry operation with a fully reproducible artifact.

## Revisit if

- Revisit if the number of models grows beyond ~10 or if external regulatory requirements (e.g. EU AI Act conformity assessment) mandate a fully audited feature store and model card per version — at that scale the lightweight S3-hash + MLflow convention should be replaced by a managed platform such as Vertex AI Model Registry or SageMaker Model Registry with integrated feature store.
