# pipeline-engineering-github-actions

Reusable GitHub Actions **workflows** (`workflow_call`) and **composite actions**
shared across the Sky Haven organisation. One version stream, one place for
`actionlint`/`zizmor` to run against the shared automation itself.

This repo is the consolidation target for what used to be copy-pasted across
every `app-*` and `infra-*` repo (tagging, linting, Terraform validation, and
the Azure state-plumbing composites).

## What's here

### Reusable workflows (`.github/workflows/`)

| Workflow | Purpose |
| -------- | ------- |
| `tag.yml` | Idempotent semver tagging on merge to `main` (branch-prefix drives the bump). |
| `megalinter.yml` | Dynamic multi-language linting via flavour-scoped MegaLinter (Super-Linter replacement). |
| `terraform-pr-validation.yml` | Terraform hygiene + zizmor + Checkov plan-aware deep analysis + Infracost. |

`ci.yml` and `release-tag.yml` are this repo's **own** CI and tagging, not for consumers.

### Composite actions (`actions/`)

| Action | Purpose |
| ------ | ------- |
| `ensure-tfstate-container` | Create the Azure state container if missing (explicit exists-check, fails loudly). |
| `break-tfstate-lease` | Break a stuck blob lease on the state file (`if: always()`). |
| `terraform-backend-init` | `terraform init` against the org backend convention, deriving RG/SA names. |
| `checkov-plan-scan` | Plan binary → JSON → Checkov `--deep-analysis` enriched with the source module. |
| `infracost-comment` | Infracost diff from a plan JSON → sticky per-env PR comment. |
| `storage-network-toggle` | Add/remove the runner egress IP on the state account firewall. |

See [`docs/`](./docs) for per-item usage snippets.

## Versioning and pinning

Tagging uses the org branch-prefix semver scheme (`major/**` → major, `minor/**`
→ minor, `patch/**` → patch) via this repo's own `tag.yml`. A floating major tag
(`v1`) is maintained automatically for humans.

**Consumers pin by full commit SHA with a version comment** (the org policy,
enforced by zizmor):

```yaml
jobs:
  tag:
    uses: skyhaven-ltd/pipeline-engineering-github-actions/.github/workflows/tag.yml@<sha> # v1.0.0
```

```yaml
      - uses: skyhaven-ltd/pipeline-engineering-github-actions/actions/break-tfstate-lease@<sha> # v1.0.0
```

## Notes for maintainers

- **Local action references inside reusable workflows** — `terraform-pr-validation.yml`
  references its sibling composites with `uses: ./actions/<name>`. GitHub resolves
  local action paths in a reusable workflow against *this* repo at the called ref,
  so consumers do not need the composites in their own checkout.
- **Line endings** — `.gitattributes` forces LF. The pre-consolidation drift was
  mostly CRLF/LF noise; keep it normalised so SHA pins and diffs stay clean.
- **Plan artifacts are ephemeral** — plan binaries and JSON are never uploaded as
  artifacts; they live and die on the runner.
