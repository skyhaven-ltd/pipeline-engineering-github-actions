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
    uses: skyhaven-ltd/pipeline-engineering-github-actions/.github/workflows/reusable-terraform.yml@<sha> # v1.0.0
    with:
      # working_directory: infra
      environments: '["dev","prd"]'   # prd-only repos pass '["prd"]'
      # enable_infracost: true
      # enable_ip_whitelist: false     # dormant until state accounts move to default-action Deny
      # terraform_version: "1.14.8"
    secrets: inherit
```

`secrets: inherit` is required, not a convenience: the Azure identifiers
(`AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID`,
`AZURE_PLATFORM_SUBSCRIPTION_ID`) are per-repo **environment** secrets
(provisioned by `infra-landingzone-platform/scripts/bootstrap-deployment-identities.sh`).
An explicit `secrets:` mapping is evaluated in the caller outside any
environment scope, so environment secrets resolve to empty strings and
`azure/login` fails with "Not all values are present". With inherit, the
environment-scoped plan job resolves them per matrix leg. zizmor flags
`secrets-inherit` on the caller — ignore it for the calling workflow in the
repo's `.github/validation/zizmor.yml`.

### Plan-time secret TF_VARs (`tf_vars_json`)

Repos whose plan needs secret variables (e.g. `stripe_secret_key`) pass them as a
single JSON object secret. It is exported as `TF_VAR_<key>` before `terraform plan`:

```json
{ "stripe_secret_key": "sk_live_...", "another_var": "value" }
```

Scalar string values only — values containing newlines are not supported through
this channel. Store the JSON as the `TF_VARS_JSON` environment secret; it flows
to the workflow automatically via `secrets: inherit`.
