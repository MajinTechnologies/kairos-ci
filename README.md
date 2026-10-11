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
repo and point `uses:` at this public reusable workflow:

```yaml
jobs:
  worker:
    uses: MajinTechnologies/kairos-ci/.github/workflows/kairos-pipeline.yml@v1.0.0
    with:
      mode: chain
      worker_ref: kairos/v1.0.1   # tag of the PRIVATE profile in kairos-worker-poc
    secrets:
      OPENCODE_AUTH: ${{ secrets.OPENCODE_AUTH }}       # optional
      GROQ_API_KEY: ${{ secrets.GROQ_API_KEY }}         # optional
      KAIROS_WORKER_TOKEN: ${{ secrets.KAIROS_WORKER_TOKEN }}  # required
```

- `KAIROS_WORKER_TOKEN` — PAT (org secret) with `contents:read` + `metadata:read`
  on `MajinTechnologies/kairos-worker-poc`. Used to check out the private
  `.opencode/` profile at `worker_ref`.
- To pick up a new worker version, bump the tag in the `uses:` ref and the
  `worker_ref` input (2 places).

## Versioning

Semantic versioning with annotated tags. Current: `v1.0.0` (mirrors the
reusable workflow from `kairos-worker-poc @ kairos/v1.0.1`).
