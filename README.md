# pipeline-engineering-github-actions

Reusable GitHub Actions workflows (`workflow_call`) and composite actions shared across the Sky Haven organisation, consolidating tagging, linting, Terraform validation, and Azure state-plumbing logic that used to be copy-pasted into every `app-*` and `infra-*` repo.

Consumers pin every reference by full commit SHA with a version comment, per org policy enforced by zizmor. See [`docs/reusable-workflows.md`](./docs/reusable-workflows.md) and [`docs/composite-actions.md`](./docs/composite-actions.md) for usage snippets.
