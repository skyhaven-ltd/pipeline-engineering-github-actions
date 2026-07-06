# Reusable workflows — consumer snippets

Pin `@<sha>` to a release commit and keep the `# vX.Y.Z` comment (zizmor enforces
SHA pinning). Replace `<sha>` below with the real release SHA.

## `reusable-tag.yml`

Consumer keeps only the trigger:

```yaml
name: Tag
on:
  pull_request:
    branches: [main]
    types: [closed]
permissions: {}
jobs:
  tag:
    permissions:
      contents: write
    uses: skyhaven-ltd/pipeline-engineering-github-actions/.github/workflows/reusable-tag.yml@<sha> # v1.0.0
```

## `reusable-lint.yml`

```yaml
name: Lint
on:
  pull_request:
    branches: [main]
permissions:
  contents: read
jobs:
  lint:
    uses: skyhaven-ltd/pipeline-engineering-github-actions/.github/workflows/reusable-lint.yml@<sha> # v1.0.0
    with:
      flavor: cupcake          # or 'terraform' for infra repos
      # config_file: .mega-linter.yml
      # validate_all_codebase: false
```

The consumer keeps a small `.mega-linter.yml` holding its `DISABLE` list and
linter config paths (this replaces the per-repo `VALIDATE_*: false` wall). For
`infra` repos, disable the IaC linters (Checkov/TFLint/terrascan) — Terraform
scanning is owned by `reusable-terraform.yml`.

## `reusable-terraform.yml`

```yaml
name: PR Validation
on:
  pull_request:
    branches: [main]
    paths-ignore:
      - "**/*.md"
      - "docs/**"
permissions:
  contents: read
jobs:
  validate:
    uses: skyhaven-ltd/pipeline-engineering-github-actions/.github/workflows/reusable-terraform.yml@<sha> # v2.0.0
    with:
      # working_directory: infra
      environments: '["dev","prd"]'   # prd-only repos pass '["prd"]'
      # enable_infracost: true
      # enable_ip_whitelist: false     # dormant until state accounts move to default-action Deny
      # terraform_version: "1.14.8"
      # keyvault_name: ""              # defaults to kv-platform-<env>-uks-02
      tf_var_secrets: |
        TF_VAR_cloudflare_api_token=cloudflare-api-token
        TF_VAR_cloudflare_account_id=cloudflare-account-id
```

There is no `secrets:` block. The workflow authenticates with the caller's
GitHub **environment variables** (`AZURE_CLIENT_ID`, `AZURE_TENANT_ID`,
`AZURE_SUBSCRIPTION_ID`, `AZURE_PLATFORM_SUBSCRIPTION_ID`), provisioned by
`infra-landingzone-platform/scripts/bootstrap-platform.sh`. The `vars` context
resolves against the caller repo, and environment-level variables apply because
the plan job is environment-scoped per matrix leg.

### Plan-time secret TF_VARs (`tf_var_secrets`)

Repos whose plan needs secret variables map `TF_VAR_*` names to Key Vault
secret names, one `TF_VAR_name=kv-secret-name` per line. After Azure login the
workflow reads each secret from the platform Key Vault
(`kv-platform-<env>-uks-02`), masks it, and exports it before `terraform plan`.
The deployment SPNs hold `Key Vault Secrets User` on the platform vaults.
Multiline values (e.g. PEM keys) are supported.
