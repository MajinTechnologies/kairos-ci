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
repo and point the `work` job at this public reusable workflow. Since `v1.2.0`
the caller owns **selection + parallelism** (the reusable receives one issue
per invocation and never declares `concurrency`):

```yaml
jobs:
  # serial: lists open issues labeled `kairos:ci`, applies the
  # KAIROS_MAX_ISSUES cap (default 1, -1 = all) and claims them
  # (`kairos:ci` -> `kairos:in-progress`) using only `gh`
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
    uses: MajinTechnologies/kairos-ci/.github/workflows/kairos-pipeline.yml@v1.2.0
    with:
      mode: chain
      issue_number: ${{ matrix.issue != 0 && format('{0}', matrix.issue) || '' }}
      worker_ref: kairos/v1.2.0   # tag of the PRIVATE profile in kairos-worker-poc
    secrets:
      OPENCODE_AUTH: ${{ secrets.OPENCODE_AUTH }}       # optional
      GROQ_API_KEY: ${{ secrets.GROQ_API_KEY }}         # optional
      KAIROS_WORKER_TOKEN: ${{ secrets.KAIROS_WORKER_TOKEN }}  # required
```

- The full working caller (including the `select` steps) ships in
  `kairos-worker-poc @ kairos/v1.2.0` — copy it from there.
- `KAIROS_WORKER_TOKEN` — PAT (org secret) with `contents:read` + `metadata:read`
  on `MajinTechnologies/kairos-worker-poc`. Used to check out the private
  `.opencode/` profile at `worker_ref`.
- `KAIROS_MAX_ISSUES` — optional **repository variable** (read by the caller,
  not by this reusable): `-1` = all eligible issues, `N` = first `N`
  (default `1` when unset — do not register the variable to keep the default).
- Anti-loop: the claim mutates labels with the Actions `GITHUB_TOKEN`, which
  does not re-trigger workflows; trigger stays exclusive to the `kairos:ci`
  label (never `status:*`).
- To pick up a new worker version, bump the tag in the `uses:` ref and the
  `worker_ref` input (2 places).

## Versioning

Semantic versioning with annotated tags. Current: `v1.2.0` (mirrors the
reusable workflow from `kairos-worker-poc @ kairos/v1.2.0`; BREAKING: the
caller must select/claim issues and pass `issue_number` — the LLM no longer
picks issues).
