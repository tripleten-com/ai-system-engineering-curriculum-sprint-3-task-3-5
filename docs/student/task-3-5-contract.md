# Task 3.5 — CI reliability gate contract

Wire one deterministic CI gate to a real reliability check, then prove — against your own
running stack, before you ever open a pull request — that it actually rejects a regression.
You edit one step in `.github/workflows/task.yml`. You do not touch the `verify` job in that
same file, the deployed dead-letter redrive budget from Task 3.3, or the deployed alert window
from Task 3.4.

## The one CI gate

`.github/workflows/task.yml` already declares a second job, `reliability-gate`, alongside the
supplied `verify` job. It checks out the repository, installs the pinned toolchain, and starts
the stack — exactly like `verify` does — but its reliability-check step between startup and
cleanup is a placeholder:

```yaml
- name: Run the reliability check (placeholder)
  run: echo "reliability gate placeholder — replace this step"
```

This is legal YAML. It runs on every pull request. It always succeeds. It does not check
anything. Replace that one step with a real command — nothing else in the file changes.

## What is assessed, and by whom

| Assessed | By |
|---|---|
| The pull request changes only `.github/workflows/task.yml` and `submission.yaml` | Automated, in this repository |
| The `reliability-gate` job's placeholder step is gone | Automated, by parsing the workflow YAML |
| The step that replaces it runs exactly `poe queue-contract` or `poe slo-contract` — nothing else | Automated, by parsing the workflow YAML |
| That command passes against your current, correctly configured stack | Automated, against the running application |
| That command actually fails when the one setting it is supposed to protect is reverted to its own Task's known starting-broken value | Automated, against the running application, with the setting reverted and restored for the duration of the check only |
| Why you picked the check you picked, and what it would miss | Your instructor, at the Project Defense |

## What is already supplied

| Supplied | Where | Note |
|---|---|---|
| The `verify` job | `.github/workflows/task.yml` | do not edit |
| The `reliability-gate` job's checkout, toolchain, and start/stop steps | `.github/workflows/task.yml` | do not edit |
| Task 3.3's own settled dead-letter redrive policy | `compose.yaml` | unchanged; not this Task's editable surface |
| Task 3.4's own settled alert window | `infra/observability/alerts.yml` | unchanged; not this Task's editable surface |
| The automated checks this Task's own check exercises | `tests/contract/test_runtime_adapters.py`, `tests/contract/test_slo_alert.py` | run them through `poe queue-contract`/`poe slo-contract`; do not edit them |

## The one step

Replace the placeholder `run:` line with exactly one of:

```yaml
run: .tools/bin/uv run --frozen poe queue-contract
```

or

```yaml
run: .tools/bin/uv run --frozen poe slo-contract
```

Either is a correct choice; they check two different, already-deployed reliability behaviors
from earlier Tasks. Pick one. Do not run both, do not run `poe verify` wholesale, and do not run
a command that is real but unrelated to either setting — a command that would stay green even if
the setting it is supposed to protect regressed is not a reliability gate, whatever else it
checks.

Nothing else in the file changes, with one exception: the `# PLACEHOLDER …` instruction comment
above that step is yours to keep, replace with your own one-line rationale, or delete — the
checks parse the workflow as YAML, which discards comments, so this one is invisible to them
either way.

## Prove it before you submit

`poe gate-contract` runs two checks against your wired job:

1. It parses `.github/workflows/task.yml` and confirms the placeholder is gone and the
   replacement step runs exactly one of the two permitted commands.
2. It runs your chosen command against your current, correct configuration — which must pass —
   then temporarily reverts the one setting that command is supposed to protect to this
   checkpoint's own known starting-broken value, runs the same command again — which must now
   fail — and restores the setting before returning. Your working tree is unchanged afterward;
   this check leaves nothing behind.

If your chosen command does not actually depend on the setting it is supposed to protect, step 2
fails: a check that stays green through a real regression is not doing its job, no matter how
real the command it runs otherwise is.

## Commands

```shell
poe gate-contract   # the automated static-wiring and live-rejection checks
poe verify          # the full public student verification path
```

## What the checks verify

| Check | What it looks at |
|---|---|
| `test_reliability_gate_runs_a_real_check` | The `reliability-gate` job exists, triggers on `pull_request`, carries the same export-branch exclusion as `verify`, no longer runs the placeholder, and its replacement step runs exactly `poe queue-contract` or `poe slo-contract` |
| `test_reliability_gate_rejects_the_known_broken_value` | Your wired command passes against the current, correct configuration; reverting the one setting it protects to this checkpoint's own known starting-broken value makes it fail; restoring the setting leaves your working tree unchanged |

## Student-editable paths

- `.github/workflows/task.yml`
- `submission.yaml`

Keep the `verify` job, the `reliability-gate` job's checkout/toolchain/start/stop steps, the
deployed queue and alert settings from Tasks 3.3 and 3.4, and every test file exactly as
supplied. The public checks compare them. Choosing and wiring the one real reliability check —
and proving it actually gates something — is this Task's assignment.
