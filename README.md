# Coldline Task 3.5 — CI reliability gate

This checkpoint wires one deterministic CI gate to a real reliability behavior and proves it
actually gates something. `.github/workflows/task.yml` already declares a `reliability-gate`
job alongside the supplied `verify` job; it checks out, installs, and starts the stack exactly
like `verify` does, but its reliability-check step between startup and cleanup is a placeholder
that runs and always succeeds without checking anything. Replacing that one step with a real command, and proving — against your own
running stack — that it actually rejects a known regression, is this Task's work. This is the
first Task in this Sprint where `.github/workflows/task.yml` itself is student-editable.

[![Open in GitHub Codespaces](https://github.com/codespaces/badge.svg)](https://codespaces.new/tripleten-com/ai-system-engineering-curriculum-sprint-3-task-3-5/tree/main)

## Start the system

Prerequisites are Python 3.12 and Docker with Compose v2. The supplied bootstrap supports macOS
arm64/x86-64, Windows x86-64, and Linux x86-64/aarch64, and installs pinned uv 0.11.8 under
`.tools/bin`. If your computer cannot run the stack locally, use the Codespaces button above.

On macOS and most Linux distributions the interpreter is `python3`; substitute it wherever these
commands say `python`.

```shell
python infra/scripts/bootstrap.py
./.tools/bin/uv sync --frozen
./.tools/bin/uv run --frozen poe preflight
./.tools/bin/uv run --frozen poe start
./.tools/bin/uv run --frozen poe ready
```

PowerShell and POSIX wrappers are available under `infra/scripts/`. After uv is on `PATH`, the
shorter `uv run --frozen poe <task>` form works.

| Service | Local URL | Purpose |
|---|---|---|
| API | `http://localhost:8000` | Submit exception workflows and retrieval queries; `/version` names the build that answers |
| Grafana | `http://localhost:3000` | Use the focused diagnostics dashboard |
| Prometheus | `http://localhost:9090` | Query bounded metrics and inspect the deployed alert rule |
| Alertmanager | `http://localhost:9093` | Inspect firing and resolved alerts |
| Jaeger | `http://localhost:16686` | Inspect local traces |
| LocalStack S3/SQS | `http://localhost:4566` | Inspect the emulated object-storage and queue endpoint |

Each of these ports can be overridden by setting the matching `COLDLINE_API_HOST_PORT`,
`COLDLINE_GRAFANA_HOST_PORT`, `COLDLINE_PROMETHEUS_HOST_PORT`, `COLDLINE_ALERTMANAGER_HOST_PORT`,
`COLDLINE_JAEGER_HOST_PORT`, or `COLDLINE_LOCALSTACK_HOST_PORT` environment variable in your shell
environment or a local `.env` file (copy `.env.example`) if a default collides with something
already running on your machine. Keep the override in place for every `poe` command.

This Task runs as its own Compose project, `coldline-task-3-5`. If an earlier Task's stack is
still running, run `poe stop` in that Task's repository first; otherwise `poe start` here fails
because the published ports are already taken.

PostgreSQL, Redis, worker metrics, and OTLP remain inside the Compose network. Codespaces uses the
same `compose.yaml` and keeps every forwarded port private. Redis keeps running in this Task only
for an earlier checkpoint's own contract test; no composition root reads it anymore.

## Command path

For this Task, run the supplied commands in this order:

```text
poe start
poe gate-contract
poe verify
```

| Command | Use |
|---|---|
| `poe gate-contract` | Run the automated static-wiring and live-rejection checks against your wired `reliability-gate` job |
| `poe queue-contract` | Run Task 3.3's own automated dead-letter redrive check |
| `poe slo-contract` | Run Task 3.4's own automated alert-bound, firing, and resolution checks |
| `poe contract` | Check interfaces, boundaries, submissions, and repository structure |
| `poe smoke` | Check the initialized running platform |
| `poe e2e` | Run the external API-to-worker workflow |
| `poe verify` | Run the public student verification path |
| `poe student-tests` | Run the supplied tests under `tests/student/`; this Task permits no additions there |
| `poe restart` | Restart the existing API and worker containers **without rebuilding** |
| `poe stop` | Remove containers and the network, keeping named volumes |
| `poe reset` | Remove containers, the network, and local named volumes |

For Task 3.5, `poe verify` starts the stack, ingests the supplied corpus, runs the smoke tests,
the end-to-end exception workflow, the queue-contract, SLO-contract, and gate-contract checks,
and the answer-sheet checks. `poe gate-contract` itself starts and stops nothing extra: it runs
your wired command against the stack `poe verify` already started, temporarily reverts and
restores the one setting that command protects, and leaves your working tree unchanged.

## Folder map

```text
repository root/
├── docs/                Student guidance, public contracts, and fidelity notes
│   ├── contracts/       Machine-readable public contracts
│   ├── fidelity/        Local-runtime boundary notes
│   ├── architecture/    Supplied vector engine technical profiles, in prose
│   ├── retrieval/       Supplied retrieval pipeline reference
│   └── student/         This Task's contract
├── config/              Retrieval configuration, settled and supplied from Sprint 2
├── infra/               Local setup and runtime configuration
│   ├── containers/      The API and worker Dockerfiles, with the build identity arguments
│   ├── observability/   Prometheus, Alertmanager, and Grafana configuration
│   ├── release/         The supplied Task 3.1 release manifest, unchanged
│   ├── corpus/          Supplied synthetic corpus, query set, and designated investigation
│   ├── judge/            Supplied cached judge evidence and its provenance record
│   ├── profiles/        Supplied engine and emulator profiles, and their provenance record
│   └── postgres/        Database initialization and the migration baseline stamp
├── loadtest/            Supplied traffic profile and provider-latency harness
├── migrations/          Alembic environment, revision template, and revisions
├── src/
│   ├── api/             HTTP application code, the retrieval and document paths, composition
│   ├── worker/          Background application code, including the dead-letter depth poller
│   ├── domain/          Shared domain code, contracts, the failure taxonomy, service and repository contracts
│   ├── ports/           Application interfaces
│   └── adapters/        Technology-specific implementations, including the supplied SQS/DLQ queue adapter
└── tests/
    ├── unit/            Isolated behavior checks
    ├── benchmark/       Supplied evaluation harness, metrics, and adoption policy
    ├── contract/        Interface, retrieval, and repository checks
    ├── diagnostics/     Supplied stage inspector
    ├── doubles/         Supplied deterministic test doubles
    ├── failure/         Supplied failure-exercise scripts — run them, do not edit them
    ├── student/         Supplied student tests; no additions in this Task
    ├── smoke/           Running-platform checks
    └── e2e/             Supplied workflow tools and checks
```

## Overview

Use the Task 5 lesson (Task 3.5 in this repository) to decide what to do. This README covers
local setup and repository orientation.

1. `README.md` — local setup, commands, and permitted changes.
2. [`docs/student/task-3-5-contract.md`](docs/student/task-3-5-contract.md) — the one step, the
   two permitted commands, what each check verifies, and the permitted paths.
3. `.github/workflows/task.yml` — the `reliability-gate` job whose one placeholder step you
   replace.
4. `tests/contract/reliability_gate.py` — the automated check `poe gate-contract` runs; read its
   docstrings to see exactly what it proves.

The application source lives in five flat packages:

| Package | Responsibility |
|---|---|
| `api` | HTTP delivery, API use cases, the retrieval workflow, versioned routes, configuration, and composition |
| `worker` | Background processing, retries, the dead-letter depth poller, configuration, and composition |
| `domain` | Provider-neutral contracts, state rules, identity, redaction, embedding, chunking, fusion, access constraints, failure classification, service and repository contracts |
| `ports` | Exactly five visible application interfaces |
| `adapters` | PostgreSQL, pgvector retrieval, LocalStack SQS/DLQ, S3-compatible object storage, deterministic model, the resilient model-provider wrapper, logs, traces |

`src/api/bootstrap.py` and `src/worker/bootstrap.py` compose each process from its settings and
adapters. Process settings live in `src/api/config.py` and `src/worker/config.py`.

## The five ports

Find the available interfaces in `src/ports/`. A port describes an application capability; an
adapter provides it using a concrete technology.

| Port | General responsibility |
|---|---|
| `ModelProvider` | Call an AI model service |
| `Retriever` | Look up relevant context or documents |
| `ObjectStore` | Store large binary objects or files |
| `JobQueue` | Publish and consume background work |
| `SecretProvider` | Read API keys and credentials |

LocalStack SQS, with a bound dead-letter queue, still carries `JobQueue`, unchanged from Task 3.3.
The dead-letter depth poller reads the dead-letter queue's own attribute directly, alongside
`JobQueue` rather than through it; see [JobQueue fidelity](docs/fidelity/JobQueue.md).

## Test levels

| Level | Requires Compose | Main question |
|---|---:|---|
| Unit | No | Does one responsibility behave correctly, including failures? |
| Contract | Some | Do interfaces, schemas, paths, and dependency rules stay compatible? |
| Smoke | Yes | Did the complete local platform initialize and become observable? |
| E2E | Yes | Can an external client complete the supplied workflow? |

Contract checks marked `runtime` need the running stack. `poe contract` skips them; `poe verify`,
`poe runtime-contract`, `poe queue-contract`, `poe slo-contract`, and `poe gate-contract` run them.

## Submission checks

Run `poe verify` locally before opening your student pull request. Public GitHub CI repeats
the student checks. This Task records `answers: {}`: your wired CI job and its automated
static-wiring and live-rejection checks are the evidence, so there is no separate protected
answer check. Follow the Task lesson's instructor-review and progression policy.

## Task boundary

Task 3.5 asks you to replace the `reliability-gate` job's placeholder step in
`.github/workflows/task.yml` with exactly one real command — `poe queue-contract` or
`poe slo-contract` — and run `poe gate-contract` until it proves that command actually depends
on the setting it is supposed to protect.

These paths are student-editable:

- `.github/workflows/task.yml`
- `submission.yaml`

Keep the `verify` job, the `reliability-gate` job's supplied steps (`Check out the
repository`, `Set up Python 3.12`, `Install pinned tooling`, `Pull and build the pinned
images` and `Start the pinned runtime` before the one step you replace, then `Show the runtime
logs on failure` and `Stop the runtime` after it), Task 3.3's own settled `compose.yaml`, Task 3.4's own settled `infra/observability/alerts.yml`, and
every test file exactly as supplied; the public checks compare them. Everything else in this
repository is supplied.

### Student walkthrough

See **Task 5: CI reliability gate** in your course platform for the full walkthrough. In
outline: read `docs/student/task-3-5-contract.md`, replace the `reliability-gate` job's
placeholder step in `.github/workflows/task.yml` with exactly `poe queue-contract` or
`poe slo-contract`, run `poe gate-contract` until both its static and live checks pass, run
`poe verify`, and open your pull request.

## Operational limits

This local system does not authenticate users, terminate TLS, or manage production secrets.
The Compose PostgreSQL password and the LocalStack access keys are local-only non-secret
credentials. Never place real credentials, personal data, or production records in this
repository.

Alertmanager here is configured with a "default" receiver that has no notification integration:
alerts are queryable through its own API but never sent anywhere real. Never add a webhook, email,
Slack, or paid integration; Sprints 1-4 are emulator-only and never call a hosted endpoint.

LocalStack's SQS emulation is a local reliability primitive, not a managed-service durability,
IAM, availability, or cost claim. See [JobQueue fidelity](docs/fidelity/JobQueue.md) for its exact
boundary.

Named volumes preserve local PostgreSQL, Redis, Prometheus, Alertmanager, Grafana, and Jaeger state
across `poe stop`. LocalStack object and queue contents are deliberately not persisted; the
initializer re-uploads the supplied corpus artifacts and re-provisions the queue on every start.
The `poe reset` command deletes the named volumes. This topology makes no backup, replication,
high-availability, disaster-recovery, capacity, latency-SLO, or availability claim beyond the one
alert Task 3.4 configures and the one CI gate this Task wires to it.

See [JobQueue fidelity](docs/fidelity/JobQueue.md),
[ModelProvider fidelity](docs/fidelity/ModelProvider.md),
[ObjectStore fidelity](docs/fidelity/ObjectStore.md), and
[Retriever fidelity](docs/fidelity/Retriever.md) for the active adapter boundaries. The
[local runtime evidence](docs/fidelity/local-runtime.md) records the current measurement and its
qualification limits.
