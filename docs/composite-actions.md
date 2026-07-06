# Composite actions — consumer snippets

Pin `@<sha>` to a release commit and keep the `# vX.Y.Z` comment. Replace `<sha>`
with the real release SHA. All Azure actions default `login: true` and run
`azure/login` themselves; pass `login: "false"` when the job already ran
`azure/login` once (recommended — avoids re-authenticating per action).

## `ensure-tfstate-container`

```yaml
      - uses: skyhaven-ltd/pipeline-engineering-github-actions/actions/ensure-tfstate-container@<sha> # v1.0.0
        with:
          storage_account_name: sttfsplatformdevuks01
          # container_name: defaults to the repository name
          login: "false"
```

## `break-tfstate-lease`

Callers must keep `if: always()` so a crashed run still releases the lease.

```yaml
      - if: always()
        uses: skyhaven-ltd/pipeline-engineering-github-actions/actions/break-tfstate-lease@<sha> # v1.0.0
        with:
          storage_account_name: sttfsplatformdevuks01
          # state_key: terraform.tfstate  (powertoggle passes per-stack keys)
          login: "false"
```

## `terraform-backend-init`

```yaml
      - uses: skyhaven-ltd/pipeline-engineering-github-actions/actions/terraform-backend-init@<sha> # v1.0.0
        with:
          environment: dev
          platform_subscription_id: ${{ secrets.AZURE_PLATFORM_SUBSCRIPTION_ID }}
          # working_directory: infra
          # state_key: terraform.tfstate
```

## `checkov-plan-scan`

```yaml
      - id: checkov
        uses: skyhaven-ltd/pipeline-engineering-github-actions/actions/checkov-plan-scan@<sha> # v1.0.0
        with:
          working_directory: infra
          plan_binary: tfplan-dev.binary   # path passed to terraform -out
          # checkov_config: .github/validation/checkov.yaml
```

## `infracost-comment`

Posts a cost-only sticky PR comment. It intentionally does not run or enforce
Infracost Cloud governance policies such as tag policies; governance checks
belong in separate workflows if a repo wants them.

```yaml
      - uses: skyhaven-ltd/pipeline-engineering-github-actions/actions/infracost-comment@<sha> # v1.0.0
        with:
          plan_json_path: infra/${{ steps.checkov.outputs.plan_json }}
          api_key: ${{ secrets.INFRACOST_API_KEY }}
          github_token: ${{ github.token }}
          pull_request_number: ${{ github.event.pull_request.number }}
          comment_tag: infracost-dev
```

## `storage-network-toggle`

Dormant until the state accounts move to `default-action: Deny`. Use symmetric
add/remove; keep `if: always()` on the remove step.

```yaml
      - id: ip_add
        uses: skyhaven-ltd/pipeline-engineering-github-actions/actions/storage-network-toggle@<sha> # v1.0.0
        with:
          mode: add
          storage_account_name: sttfsplatformdevuks01
          resource_group_name: rg-tfs-platform-dev-uks-01
          login: "false"

      # ... terraform steps ...

      - if: always()
        uses: skyhaven-ltd/pipeline-engineering-github-actions/actions/storage-network-toggle@<sha> # v1.0.0
        with:
          mode: remove
          storage_account_name: sttfsplatformdevuks01
          resource_group_name: rg-tfs-platform-dev-uks-01
          ip_address: ${{ steps.ip_add.outputs.ip_address }}
          public_access_was_disabled: ${{ steps.ip_add.outputs.public_access_was_disabled }}
          login: "false"
```

**Identity requirement:** the OIDC principal needs a management-plane role on the
storage account (e.g. Storage Account Contributor) — data-plane roles cannot edit
network rules.

**Azure caveat:** IP network rules cannot allow traffic from the *same* Azure
region as the account. GitHub-hosted runners egress from US regions while the
state account is UK South, so this works today; self-hosted UK South runners
would need private endpoints instead.
