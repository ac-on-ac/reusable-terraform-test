# Reusable Terraform Test

A reusable GitHub Actions workflow that discovers and runs Terraform test files using the [Terraform Testing Framework](https://developer.hashicorp.com/terraform/language/tests), organised by test type.

## Purpose

This workflow discovers `.tftest.hcl` files in a specified directory, categorises them by prefix, and runs them in the following order:

1. **Validation tests** — run first
2. **Unit tests** — run after validation passes
3. **Integration tests** — run after unit tests pass

Within each phase, tests are sorted alphabetically by filename. This is an intentional convention that allows test authors to control execution order by prefixing filenames with numbers (e.g. `unit-01-defaults.tftest.hcl`, `unit-02-custom-config.tftest.hcl`).

All failing tests within a phase are reported before the step exits — the workflow does not stop at the first failure.

## Test file naming convention

Test files must be placed in the `test_directory` and named using the following patterns:

| Pattern | Category | Purpose |
|---|---|---|
| `validation-*.tftest.hcl` | Validation | Tests against Terraform variable [validation blocks](https://developer.hashicorp.com/terraform/language/values/variables#custom-validation-rules). These verify that invalid inputs are correctly rejected. |
| `unit-*.tftest.hcl` | Unit | Tests that use mock providers and (where necessary) mock resources to validate module logic without deploying real infrastructure. |
| `integration-*.tftest.hcl` | Integration | Tests that run in `apply` mode and validate the behaviour of actually deployed resources. |

### Unit and integration test pairing

Unit and integration tests are written in pairs — each pair tests the same scenario and parameter set. The unit test validates logic quickly using mocks; the integration test confirms the same behaviour against real infrastructure.

Pairs should share the same descriptive suffix so their relationship is clear:

```
unit-01-defaults.tftest.hcl
integration-01-defaults.tftest.hcl

unit-02-private-endpoint-enabled.tftest.hcl
integration-02-private-endpoint-enabled.tftest.hcl
```

### Alphabetical ordering

Within each phase, tests run in alphabetical order by filename after the prefix is stripped (e.g. `validation-`, `unit-`, `integration-`). Use numeric prefixes in the descriptive suffix to control execution order where it matters:

```
unit-01-defaults.tftest.hcl       ← runs first
unit-02-with-optional-var.tftest.hcl
unit-03-edge-case.tftest.hcl      ← runs last
```

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `test_directory` | Yes | — | Directory containing the `.tftest.hcl` test files |
| `terraform_version` | No | `latest` | Version of Terraform to install (e.g. `1.11.0`) |
| `terraform_vars` | No | `''` | Newline-separated `TF_VAR_*=value` pairs to export before tests run. Only `TF_VAR_*` keys are permitted. |

## Secrets

All secrets are optional. Only supply the secrets required by the providers your module uses.

### Azure

| Secret | Description |
|---|---|
| `azure_credentials` | Azure service principal credentials JSON for `azure/login`. Triggers Azure login when present. |
| `databricks_host` | Databricks workspace host URL |
| `databricks_azure_workspace_resource_id` | Databricks Azure workspace resource ID |

### Terraform Cloud / Enterprise

| Secret | Description |
|---|---|
| `terraform_token` | Terraform Cloud or Terraform Enterprise API token |

### AWS

| Secret | Description |
|---|---|
| `aws_access_key_id` | AWS access key ID (static credentials) |
| `aws_secret_access_key` | AWS secret access key (static credentials) |
| `aws_role_to_assume` | IAM role ARN to assume via OIDC or static credentials |

At least one of `aws_access_key_id` or `aws_role_to_assume` must be provided to trigger AWS login. The AWS region defaults to `us-east-1`.

### GitHub

| Secret | Description |
|---|---|
| `gh_token` | GitHub personal access token or app token for the Terraform GitHub provider. Exposed as `GITHUB_TOKEN_PROVIDER` to avoid conflicting with the built-in `GITHUB_TOKEN`. |

### HashiCorp Vault

| Secret | Description |
|---|---|
| `vault_addr` | Vault server URL |
| `vault_token` | Vault token (token auth method) |
| `vault_role_id` | Vault AppRole role ID (AppRole auth method) |
| `vault_secret_id` | Vault AppRole secret ID (AppRole auth method) |

`vault_addr` must be provided alongside either `vault_token` (token auth) or both `vault_role_id` and `vault_secret_id` (AppRole auth) to trigger Vault login.

### Snowflake

| Secret | Description |
|---|---|
| `snowflake_account` | Snowflake account identifier |
| `snowflake_user` | Snowflake username |
| `snowflake_password` | Snowflake password |
| `snowflake_role` | Snowflake role |

Snowflake has no dedicated login action. Secrets are exposed as the `SNOWFLAKE_ACCOUNT`, `SNOWFLAKE_USER`, `SNOWFLAKE_PASSWORD`, and `SNOWFLAKE_ROLE` environment variables, which the [Terraform Snowflake provider](https://registry.terraform.io/providers/Snowflake-Labs/snowflake/latest/docs#authentication) reads natively.

### Grafana

| Secret | Description |
|---|---|
| `grafana_url` | Grafana instance URL |
| `grafana_auth` | Grafana API key or service account token |

Grafana has no dedicated login action. Secrets are exposed as the `GRAFANA_URL` and `GRAFANA_AUTH` environment variables, which the [Terraform Grafana provider](https://registry.terraform.io/providers/grafana/grafana/latest/docs) reads natively.

## Permissions

The calling workflow must grant these permissions:

| Permission | Reason |
|---|---|
| `contents: read` | Check out the repository |

## Calling this workflow

```yaml
name: Terraform Test

on:
  pull_request:
    types:
      - opened
      - synchronize
      - reopened

permissions:
  contents: read

jobs:
  test:
    uses: ac-on-ac/reusable-terraform-test/.github/workflows/test.yml@v1.0.0
    with:
      test_directory: tests
    secrets:
      azure_credentials: ${{ secrets.AZURE_CREDENTIALS }}
      terraform_token: ${{ secrets.TF_TOKEN }}
```

With optional version pinning, Terraform variables, and additional providers:

```yaml
jobs:
  test:
    uses: ac-on-ac/reusable-terraform-test/.github/workflows/test.yml@v1.0.0
    with:
      test_directory: tests
      terraform_version: 1.11.0
      terraform_vars: |
        TF_VAR_location=uksouth
        TF_VAR_environment=test
    secrets:
      azure_credentials: ${{ secrets.AZURE_CREDENTIALS }}
      terraform_token: ${{ secrets.TF_TOKEN }}
      databricks_host: ${{ secrets.DATABRICKS_HOST }}
      databricks_azure_workspace_resource_id: ${{ secrets.DATABRICKS_WORKSPACE_RESOURCE_ID }}
      aws_access_key_id: ${{ secrets.AWS_ACCESS_KEY_ID }}
      aws_secret_access_key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      aws_role_to_assume: ${{ secrets.AWS_ROLE_TO_ASSUME }}
      gh_token: ${{ secrets.GH_TOKEN }}
      vault_addr: ${{ secrets.VAULT_ADDR }}
      vault_role_id: ${{ secrets.VAULT_ROLE_ID }}
      vault_secret_id: ${{ secrets.VAULT_SECRET_ID }}
      snowflake_account: ${{ secrets.SNOWFLAKE_ACCOUNT }}
      snowflake_user: ${{ secrets.SNOWFLAKE_USER }}
      snowflake_password: ${{ secrets.SNOWFLAKE_PASSWORD }}
      snowflake_role: ${{ secrets.SNOWFLAKE_ROLE }}
      grafana_url: ${{ secrets.GRAFANA_URL }}
      grafana_auth: ${{ secrets.GRAFANA_AUTH }}
```

## `terraform_vars` security

Only keys prefixed with `TF_VAR_` are permitted in `terraform_vars`. Any other key (e.g. attempts to override `PATH` or `ACTIONS_RUNTIME_TOKEN`) will cause the workflow to fail immediately with an error identifying the disallowed key.

## Input validation

`test_directory` is validated against the repository root using `realpath` before any tests run. Path traversal values (e.g. `../../other-repo`) are rejected. The directory must also exist.

## Releases

This repository uses the [reusable-manual-release](https://github.com/ac-on-ac/reusable-manual-release) workflow to create releases. Releases are triggered manually from the **Actions** tab.
