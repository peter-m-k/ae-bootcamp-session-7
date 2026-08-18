# CI Pipeline — Golden Path

## What `golden-path-ci.yml` Does

`.github/workflows/golden-path-ci.yml` is a **reusable workflow** (`on: workflow_call`) that centralizes the CI logic every service team needs, so individual repos don't reinvent lint/test/security/deploy plumbing. It exposes tunable inputs (`node_version`, `terraform_version`, `run_terraform_plan`, `run_terraform_apply`, `build_and_push`) and one secret input (`aws_role_arn`) so callers control behavior without forking the workflow.

| Job | Why it exists |
|---|---|
| `lint` | Fails fast on style/syntax issues in `packages/backend` and `packages/frontend` before spending CI minutes on tests or builds. |
| `test` | Runs Jest with coverage on the backend and publishes a coverage summary to the job summary. The 80% coverage floor is enforced by `coverageThreshold` in `packages/backend/package.json`, so a low-coverage PR fails here, not later in review. |
| `security-scan` | Runs `checkov --hard-fail-on HIGH` against `infra/` so infrastructure changes are checked for high-severity misconfigurations before they reach `terraform plan`/`apply`. Only runs when `run_terraform_plan` is true, since it's only relevant to repos that ship Terraform. |
| `terraform-plan` | Validates and plans the `infra/stacks/dev` stack using mock subnet/VPC vars so a full plan can run on every PR without needing a live networking stack. Uses OIDC (`aws-actions/configure-aws-credentials`) to read remote state — no long-lived AWS keys involved. |
| `docker-build` | Builds (but does not push) both Dockerfiles on every pull request to catch broken images early, without needing any AWS credentials. |
| `terraform-apply` | Applies the previously generated plan against real infrastructure. Gated behind `run_terraform_apply` and only wired up by callers to run on `push` to `main` — never on pull requests. |
| `build-and-push` | Builds and pushes both images to ECR, then forces a new ECS deployment. Gated behind `build_and_push`, and only runs after a successful `terraform-apply` so the ECR repos are guaranteed to exist. |

## How a New Service Team Adopts It

A team only needs a small caller workflow — they never modify `golden-path-ci.yml` itself. Minimum viable caller:

```yaml
name: My Service CI

on:
  push:
    branches:
      - main
  pull_request:

permissions:
  contents: read
  pull-requests: write
  id-token: write   # required at the caller level for OIDC — see "Secrets" below

jobs:
  call-golden-path:
    uses: ./.github/workflows/golden-path-ci.yml
    with:
      node_version: "20"
      run_terraform_plan: true
      run_terraform_apply: ${{ github.event_name == 'push' && github.ref == 'refs/heads/main' }}
      build_and_push: ${{ github.event_name == 'push' && github.ref == 'refs/heads/main' }}
    secrets:
      aws_role_arn: ${{ secrets.AWS_ROLE_ARN }}
```

This is exactly what `.github/workflows/todo-service-ci.yml` does for the todo-service. To adopt the golden path, a new team:

1. Copies this caller file into their own repo's `.github/workflows/`.
2. Configures the `AWS_ROLE_ARN` repository secret (see below).
3. Ensures their Terraform stack lives at `infra/stacks/dev` and exposes the same outputs (`service_url`, `cluster_name`, `backend_ecr_repository_url`, `frontend_ecr_repository_url`) the `build-and-push`/`terraform-apply` jobs depend on.

## What Each Required Check Validates

| Check | Validates | Why it's required |
|---|---|---|
| `lint` | Code conforms to the repo's ESLint rules | Cheapest, fastest signal — catches style and correctness issues before slower jobs run |
| `test` | Backend behavior is correct and coverage stays ≥ 80% | Prevents untested logic from being deployed; coverage regression is a leading indicator of quality drift |
| `security-scan` | No HIGH-severity Checkov findings in `infra/` | Blocks infrastructure misconfigurations (open ingress, missing encryption, etc.) from ever reaching `plan`/`apply` |
| `terraform-plan` | The Terraform stack is syntactically valid and produces a coherent plan | Confirms IaC changes are deployable before merge, without needing to touch real infrastructure on every PR |

## Configuring Secrets (OIDC Role ARN) for `terraform-plan`

`terraform-plan` (and `terraform-apply`, `build-and-push`) authenticate to AWS via OIDC federation instead of long-lived access keys:

1. In the AWS account, create an IAM role with a trust policy that allows GitHub's OIDC provider (`token.actions.githubusercontent.com`) to assume it, scoped to this repository (and optionally `ref:refs/heads/main`).
2. Attach the least-privilege permissions the role needs (S3 state access, ECR, ECS, etc.).
3. In the GitHub repo, add the role's ARN as a repository secret named `AWS_ROLE_ARN` (Settings → Secrets and variables → Actions).
4. The caller workflow passes it through to the reusable workflow's secret input:

   ```yaml
   secrets:
     aws_role_arn: ${{ secrets.AWS_ROLE_ARN }}
   ```

5. Inside `golden-path-ci.yml`, the job requests a token by declaring `permissions: id-token: write` **on the job**, and the caller workflow must also declare `id-token: write` in its **top-level** `permissions` block — GitHub only issues the OIDC token to workflows that request it explicitly at the caller level. Missing this on the caller causes an OIDC/permissions failure even though `golden-path-ci.yml` declares it correctly.
