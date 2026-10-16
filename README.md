# kairos-ci

Reusable CI pipeline for the **Kairos** worker (MajinTechnologies).

This repository is **public** and holds only the *plumbing* (the reusable
workflow). It contains no agent prompts, no governance and no secrets.

The worker **profile** — the `.opencode/` directory with agents, prompts and
governance — stays **private** in
[`MajinTechnologies/kairos-worker-poc`](https://github.com/MajinTechnologies/kairos-worker-poc)
and is cloned **at runtime** inside the pipeline using the
`KAIROS_WORKER_TOKEN` secret.

## Why two repos

GitHub does not allow a reusable workflow hosted in a **private** repository to
be called from a **public** repository ("workflow was not found"). Splitting
the plumbing (public) from the profile (private) lets public consumer repos use
the worker while keeping the profile private.

## Usage (consumer repo)

Copy the thin caller `.github/workflows/kairos-worker.yml` into the consumer
repo and point the `work` job at this public reusable workflow. Since `v1.3.0`
the caller owns **selection + parallelism + claim release** (the reusable
receives one issue per invocation and never declares `concurrency`):

```yaml
jobs:
  # serial: lists OPEN issues labeled `kairos:ci` and WITHOUT
  # `kairos:in-progress`, applies the KAIROS_MAX_ISSUES cap (default 1,
  # -1 = all) and claims them by ADDING `kairos:in-progress` only
  # (`kairos:ci` is the operator kill-switch: never touched) using only `gh`
  select:
    runs-on: ubuntu-latest
    concurrency:
      group: kairos-select
      cancel-in-progress: false
    outputs:
      issues: ${{ steps.select.outputs.issues }}   # JSON array, e.g. [12,15]
      count: ${{ steps.select.outputs.count }}
    # ...steps: gh issue list/claim (see the template)...

  # parallel: one reusable-workflow job per issue, serialized per issue
  work:
    needs: select
    if: needs.select.outputs.count != '0'   # sentinel [0] avoids empty-matrix error
    strategy:
      matrix:
        issue: ${{ fromJSON(needs.select.outputs.issues) }}
    concurrency:
      group: kairos-issue-${{ matrix.issue }}
      cancel-in-progress: false
    uses: MajinTechnologies/kairos-ci/.github/workflows/kairos-pipeline.yml@v1.3.0
    with:
      mode: chain
      issue_number: ${{ matrix.issue != 0 && format('{0}', matrix.issue) || '' }}
      worker_ref: kairos/v1.3.0   # tag of the PRIVATE profile in kairos-worker-poc
    secrets:
      OPENCODE_AUTH: ${{ secrets.OPENCODE_AUTH }}       # optional
      GROQ_API_KEY: ${{ secrets.GROQ_API_KEY }}         # optional
      KAIROS_WORKER_TOKEN: ${{ secrets.KAIROS_WORKER_TOKEN }}  # required

  # always: releases the claim (removes `kairos:in-progress`); runs even if
  # `work` fails or never starts, so the next cron can retry the issue
  release:
    needs: [select, work]
    if: always() && needs.select.outputs.count != '0'
    strategy:
      matrix:
        issue: ${{ fromJSON(needs.select.outputs.issues) }}
    # ...steps: gh issue edit --remove-label kairos:in-progress...
```

- The full working caller (including the `select`/`release` steps) ships in
  `kairos-worker-poc` at the tag `kairos/v1.3.0`.
- Claim model: the caller **only adds** `kairos:in-progress`. The `kairos:ci`
  label is the **operator's order / kill-switch** and the machine never
  adds/removes it; a **closed** issue is the chain terminator (`state:open`).
- The `release` job (`if: always()`) removes `kairos:in-progress` at the end of
  every run, so a dead `work` job does not block the next cron retry.
- `KAIROS_WORKER_TOKEN` — PAT (org secret) with `contents:read` + `metadata:read`
  on `MajinTechnologies/kairos-worker-poc`. Used to check out the private
  `.opencode/` profile at `worker_ref`.
- `KAIROS_MAX_ISSUES` — optional **repository variable** (read by the caller,
  not by this reusable): `-1` = all eligible issues, `N` = first `N`
  (default `1` when unset — do not register the variable to keep the default).
- Anti-loop: the claim/release label mutations use the Actions `GITHUB_TOKEN`, which
  does not re-trigger workflows; trigger stays exclusive to the `kairos:ci`
  label (never `status:*`).
- To pick up a new worker version, bump both refs (distinct namespaces): the
  `uses:` tag of **this** public repo (`vX.Y.Z`) and the `worker_ref` input
  (the private profile tag `kairos/vX.Y.Z`).

## Versioning

Semantic versioning with annotated tags. Current: `v1.3.0` (mirrors the
reusable workflow from `kairos-worker-poc @ kairos/v1.3.0`; BREAKING: corrected
claim model — the caller only ADDS `kairos:in-progress`, never touches
`kairos:ci`, and releases the claim with an `always()` job).
