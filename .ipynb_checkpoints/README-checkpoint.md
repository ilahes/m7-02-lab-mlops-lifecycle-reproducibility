![logo_ironhack_blue 7](https://user-images.githubusercontent.com/23629340/40541063-a07a0a8a-601a-11e8-91b5-2f13e4e6b441.png)

# Lab | MLOps Lifecycle & Reproducibility

## Overview

You will design the MLOps machinery for a small team without writing a training script. The output is a registry specification, a lifecycle diagram, and an ADR on reproducibility. By the end you'll have artifacts a real team could implement in a sprint.

This is a 90-minute design lab. No Python, no notebooks. You'll write YAML, Markdown, and a diagram.

## Learning Goals

By the end of this lab you should be able to:

- Specify a model registry with explicit promotion gates and approver roles
- Draw the lifecycle from training run to production deployment, naming every artifact
- Defend a reproducibility strategy that another engineer can challenge

## Setup

Fork and clone the repo. No language toolchains. You'll need a Markdown editor and a diagram tool (Mermaid in Markdown is recommended).

## The Scenario

You've joined the ML team at **NorthStar Logistics**. The team has four engineers and ships two models:

1. **ETA model** — predicts arrival time at delivery. Online inference, ~200 RPS, retrained every 2 weeks.
2. **Dispatcher routing model** — picks which courier should accept each pickup. Online inference, ~50 RPS, retrained monthly.

Until now, models have lived in S3 with names like `eta_v2_FINAL.onnx`. There is no registry. The CTO has asked you to fix this.

## Tasks

### Task 1 — Lifecycle diagram

Produce `lifecycle.md` (Mermaid) or `lifecycle.png` showing the end-to-end flow for the ETA model: data → experimentation → training → evaluation → registry → deployment → monitoring → back to data.

Requirements:

- **Name every artifact** on each arrow (dataset hash, run ID, model URI, deployed version, drift signal).
- **Mark the registry stages** (Staging / Production / Archived) as their own boxes.
- **Show the monitoring loopback** — what signal feeds what?
- **Annotate** which transitions are automatic and which require approval.

A good diagram has ~10–15 boxes. If you need 30, you're drawing infrastructure; zoom out to artifacts and contracts.

### Task 2 — Registry specification

Write `model-registry.yaml` defining how the registry behaves. Use this skeleton and fill it in:

```yaml
registry:
  name: northstar-models
  stages:
    - name: staging
      promoted_from: <what triggers entry?>
      promotion_rule: <automatic | manual>
      gates:
        - <metric or check that must pass>
        - <metric or check that must pass>
    - name: production
      promoted_from: staging
      promotion_rule: manual
      gates:
        - <e.g. slice metric thresholds>
        - <e.g. successful canary rollout>
      approvers:
        - <role>
        - <role>
    - name: archived
      promoted_from: <what triggers archival?>

models:
  - name: eta
    owner: <team or role>
    sla_p95_ms: 150
    retention_versions: 10
  - name: dispatcher-routing
    owner: <team or role>
    sla_p95_ms: 80
    retention_versions: 10

lineage:
  required_fields:
    - <e.g. git_sha>
    - <e.g. dataset_hash>
    - <fill in 2-3 more>
```

Be specific. "Tests must pass" is not a gate. "Macro F1 on the frozen evaluation set ≥ 0.82, plus per-city F1 ≥ 0.70 on the bottom 3 cities" is.

### Task 3 — Reproducibility ADR

Write `adr/0001-reproducibility-strategy.md` capturing how the team will guarantee reproducibility across the four layers (environment, data, code, randomness). Use the standard ADR format:

```markdown
# ADR 0001: Reproducibility strategy for NorthStar models

## Context
(Two sentences on the current state — no registry, no dataset versioning, models named with FINAL.)

## Decision
(One paragraph per layer: environment, data, code, randomness — what specifically will be pinned and how.)

## Alternatives rejected
(2–3 bullets — e.g. "no dataset versioning, rely on S3 timestamps" — and why that fails.)

## Consequences
(2–3 bullets — what this commits the team to operationally.)

## Revisit if
(One bullet — what change in scale or compliance forces a rethink.)
```

You **must** name specific tools (or specific lightweight conventions if you choose not to adopt heavy tooling). Generic answers like "we will track experiments somehow" fail the bar.

## Submission

Open a PR to the lab repository with these files at the repo root:

```
lifecycle.md          # or lifecycle.png + source
model-registry.yaml
adr/0001-reproducibility-strategy.md
```

Paste the PR link as your deliverable.

## Quality bar

You will be reviewed on:

- **Does the lifecycle diagram name artifacts**, or is it shapeless boxes labeled "train" and "deploy"?
- **Are the registry gates measurable**, or are they aspirational?
- **Does the ADR commit to specific tools and conventions**, or hedge with "we should consider…"?

Make decisions. That's the whole point.
