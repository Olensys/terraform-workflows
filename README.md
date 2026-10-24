# terraform-workflows

Reusable GitHub Actions workflows for Olensys Terraform/OpenTofu modules. The repository is public, so the workflows can be called from any repository.

Callers reference the workflows by major tag: `Olensys/terraform-workflows/.github/workflows/<name>.yml@v1`. The `v1` tag is a floating alias that is moved to the latest compatible commit on `main`.

## Module repo requirements

- `.terraform-version` and `.opentofu-version` at the repo root, each containing a single version (for example `1.15.6` and `1.12.5`).
- `.pre-commit-config.yaml` using `antonbabenko/pre-commit-terraform` hooks (`terraform_fmt`, `terraform_validate`, `terraform_docs`, `terraform_tflint`, `terraform_trivy`).
- `examples/minimal`, at least one unit test file in `tests/` (for example `tests/defaults.tftest.hcl`) and (for integration) `tests/complete.tftest.hcl`.

## module-ci.yml

Inputs: none. Secrets: none. Permissions: `contents: read`.

Jobs:

- `static`: installs pre-commit, Terraform, tflint, trivy and terraform-docs, then runs `pre-commit run --all-files --show-diff-on-failure`. The hooks come from the module's own `.pre-commit-config.yaml`.
- `validate-and-test`: matrix `engine: [terraform, tofu]`. Each engine runs `init -backend=false -input=false`, `validate` and `test` in the repo root, with one `-filter` per file in `tests/*.tftest.hcl` except `tests/complete.tftest.hcl` (the integration apply); it fails if there are no unit test files, then `init -backend=false` and `validate` in `examples/minimal`.

```yaml
name: ci
on:
  pull_request:
  push:
    branches: [main]
permissions:
  contents: read
jobs:
  ci:
    uses: Olensys/terraform-workflows/.github/workflows/module-ci.yml@v1
```

## module-release.yml

Inputs: none. Secrets: none (uses `GITHUB_TOKEN`). Permissions: `contents: write`, `pull-requests: write`.

Runs `googleapis/release-please-action@v4` with `release-type: terraform-module`. Conventional commits drive the version bump and a release PR; merging it creates the `vX.Y.Z` tag and GitHub release.

Note: the organization setting "Allow GitHub Actions to create and approve pull requests" must be enabled for release PRs to open. PRs opened with `GITHUB_TOKEN` do not trigger CI workflows, so close and reopen a release PR (or push to its branch) to run checks on it.

```yaml
name: release
on:
  push:
    branches: [main]
permissions:
  contents: write
  pull-requests: write
jobs:
  release:
    uses: Olensys/terraform-workflows/.github/workflows/module-release.yml@v1
```

## module-integration.yml

Runs `<engine> init -backend=false` and `<engine> test -filter=tests/complete.tftest.hcl` against a real cloud account using OIDC. Runs are serialized per repository (`concurrency` group `integration-<repo>`, no cancellation) and time out after 60 minutes.

Inputs:

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `cloud` | yes | | `aws` or `google` |
| `engine` | no | `terraform` | `terraform` or `tofu` |

Secrets:

| Name | Used when | Description |
|------|-----------|-------------|
| `AWS_ROLE_ARN` | `cloud: aws` | IAM role assumed via OIDC |
| `AWS_REGION` | `cloud: aws` | Region for the session |
| `GCP_WORKLOAD_IDENTITY_PROVIDER` | `cloud: google` | Workload identity provider resource name |
| `GCP_SERVICE_ACCOUNT` | `cloud: google` | Service account email to impersonate |

The caller must grant `id-token: write` and `contents: read`. Never add `pull_request` to this trigger: it would hand OIDC credentials to fork PRs.

Example inputs: define repository variables `TF_VAR_<variable>` for every example input without a default; list values use JSON, e.g. `["subnet-a","subnet-b"]`. Every repository variable whose name starts with `TF_VAR_` (the prefix is matched case-insensitively, because GitHub stores variable names in upper case) is exported into the job environment before the engine runs, with the part after the prefix lowercased (`TF_VAR_PROJECT_ID` becomes `TF_VAR_project_id`) to match Terraform's case-sensitive variable names.

AWS:

```yaml
name: integration
on:
  push:
    branches: [main]
    paths:
      - '**/*.tf'
      - '**/*.tftest.hcl'
      - '.terraform-version'
      - '.opentofu-version'
      - '.github/workflows/integration.yml'
  schedule:
    - cron: '0 3 * * 0'
  workflow_dispatch:
permissions:
  id-token: write
  contents: read
jobs:
  integration:
    uses: Olensys/terraform-workflows/.github/workflows/module-integration.yml@v1
    with:
      cloud: aws
    secrets:
      AWS_ROLE_ARN: ${{ secrets.AWS_ROLE_ARN }}
      AWS_REGION: ${{ secrets.AWS_REGION }}
```

Google:

```yaml
name: integration
on:
  push:
    branches: [main]
    paths:
      - '**/*.tf'
      - '**/*.tftest.hcl'
      - '.terraform-version'
      - '.opentofu-version'
      - '.github/workflows/integration.yml'
  schedule:
    - cron: '0 3 * * 0'
  workflow_dispatch:
permissions:
  id-token: write
  contents: read
jobs:
  integration:
    uses: Olensys/terraform-workflows/.github/workflows/module-integration.yml@v1
    with:
      cloud: google
    secrets:
      GCP_WORKLOAD_IDENTITY_PROVIDER: ${{ secrets.GCP_WORKLOAD_IDENTITY_PROVIDER }}
      GCP_SERVICE_ACCOUNT: ${{ secrets.GCP_SERVICE_ACCOUNT }}
```

## License

Apache-2.0.
