# platform-actions

Shared GitHub Actions assets maintained by the **Platform Engineering (PE)** team. Team GitOps
repos (e.g. `team-alpha-gitops`) are **thin callers** of what lives here, so the rules and topology
change in one place for every team at once — the same pattern already used by `validate.yml`
(which calls `platform-gitops/.github/workflows/validate-team-repo.yml`).

## Contents

| Path | What it is |
|------|------------|
| `catalog.yaml` | **Single source of truth** for shareable platform facts, as a hierarchy `defaults → env → cloud → region`. A fact is set at the level it applies to and inherited downward (registry, resourceGroup, subscription, and future keys like keyVault/oidcIssuer). |
| `.github/workflows/promote.yml` | **Reusable** (`on: workflow_call`) promotion workflow. Copies an image's exact digest between cluster registries (build-once), verifies provenance into stg/prd, pins the target leaf, and opens a CODEOWNERS-gated PR in the **caller** (team) repo. |
| `actions/resolve-registry` | **Composite action** that resolves a from/to cluster pair to their registry hosts and the copy `mode` (`acr`/`ecr`/`none`), reading each cluster's effective `registry` from `catalog.yaml`. |

## How a team repo uses it

`.github/workflows/promote.yml` in the team repo is a thin caller:

```yaml
name: promote
on:
  workflow_dispatch:
    inputs:
      service:            { description: "Service dir under apps/ (e.g. orders-api)", required: true }
      from_cluster:       { description: "Source leaf <env>/<cloud>/<region>",        required: true }
      to_cluster:         { description: "Target leaf <env>/<cloud>/<region>",         required: true }
      skip_justification: { description: "EMERGENCY ONLY: reason to skip a ring",      required: false, default: "" }
permissions:
  contents: write
  pull-requests: write
  id-token: write
jobs:
  promote:
    uses: imkrupakar/platform-actions/.github/workflows/promote.yml@main
    with:
      service: ${{ inputs.service }}
      from_cluster: ${{ inputs.from_cluster }}
      to_cluster: ${{ inputs.to_cluster }}
      skip_justification: ${{ inputs.skip_justification }}
    secrets: inherit
```

`secrets: inherit` passes the team repo's promotion secrets (`AZURE_PROMOTE_CLIENT_ID`,
`AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID`, `AWS_PROMOTE_ROLE_ARN`) through to the reusable workflow.

## Onboarding a new cluster's registry

Add the cluster's `registry` under its path in `catalog.yaml` (facts shared by a whole env/cloud —
a subscription, an AWS account — are set once at that level and inherited):

```yaml
environments:
  prd:
    clouds:
      azure:
        subscription: <prd-sub>
        regions:
          eastus:
            registry: prodacrplatform.azurecr.io   # <- the cluster's registry
            resourceGroup: rg-platform-prd-eus
```

The registry value is what Terraform creates for that cluster (`platform-terraform` emits
`output "acr_login_server"`; the cluster is indexed in `platform-terraform/fleet.yaml`).

## Adding other shareable facts

`catalog.yaml` is not registry-only. Any value teams/workflows must share lives here at the level
it applies to and is read the same way (`resolve-registry`'s level-walk helper). Examples: per-cluster
`keyVault` / `resourceGroup` / `oidcIssuer`, per-cloud `subscription` / `account`, platform-wide
`org` / `tenant`.

## Versioning

Callers pin `@main` to match the existing `validate.yml` convention. For stricter supply-chain
control, tag this repo (e.g. `v1`) and have teams pin the tag or a commit SHA.
