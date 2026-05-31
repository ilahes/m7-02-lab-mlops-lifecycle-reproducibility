# ETA Model — MLOps Lifecycle

```mermaid
flowchart TD
    A([Raw Delivery Events\nS3 event logs]) -->|raw delivery events| B[Data Validation]
    B -->|validated dataset vYYYYMMDD\n+ dataset_hash\n🤖 automatic| C[Feature Engineering]
    C -->|feature schema v{N}\n+ feature store snapshot\n🤖 automatic| D[Experimentation\nMLflow Tracking]
    D -->|training run ID\n+ hyperparameter config\n🤖 automatic| E[Model Training]
    E -->|model artifact URI\n+ environment_image_digest\n🤖 automatic| F[Evaluation\nFrozen Eval Set]
    F -->|evaluation report\nMAE ≤ threshold\n🤖 automatic if gates pass| G[[Staging]]
    G -->|approved model version\n+ signed-off evaluation report\n👤 manual: ml-lead| H[[Production]]
    H -->|replaced model version\n🤖 automatic on new promotion| I[[Archived]]
    H -->|deployed version tag\n+ canary config after approval\n🤖 automatic post-approval| J[Serving\n~200 RPS]
    J -->|latency p95 ms\nprediction confidence\n🤖 automatic| K[Monitoring\n& Alerting]
    K -->|drift signal\nSLA breach alert\n🤖 automatic trigger| L{Retrain\nNeeded?}
    L -->|yes — drift or schedule\nnew dataset_hash| B
    L -->|no — stable| K
```

## Artifact legend

| Arrow | Artifact |
|---|---|
| Raw → Validation | `raw delivery events` (S3 prefix + timestamp range) |
| Validation → Features | `validated dataset vYYYYMMDD`, `dataset_hash` (SHA-256) |
| Features → Experiment | `feature schema v{N}`, feature store snapshot URI |
| Experiment → Training | `training run ID` (MLflow), hyperparameter config |
| Training → Evaluation | `model artifact URI` (S3), `environment_image_digest` |
| Evaluation → Staging | `evaluation report` (JSON), passes all metric gates |
| Staging → Production | `approved model version`, sign-off by ml-lead |
| Production → Archived | `replaced model version` tag |
| Production → Serving | `deployed version tag`, canary config (after manual approval) |
| Serving → Monitoring | latency p95, prediction confidence scores |
| Monitoring → Retrain | `drift signal`, SLA breach alert, scheduled cadence |

## Transition annotations

- 🤖 **Automatic**: data validation → feature engineering → training → evaluation → staging (if all gates pass)
- 👤 **Manual approval required**: staging → production (ml-lead + platform-owner sign-off)
- 🤖 **Automatic / policy-based**: production → archived (triggered when a new version enters production or on explicit deprecation)
- 🤖 **Monitoring loopback**: drift signal or SLA breach triggers a new training run with a fresh dataset snapshot
