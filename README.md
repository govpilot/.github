# govpilot Engineering — Shared Workflow Templates & Standards

This repository is the canonical source of GitHub Actions workflow templates, PR templates, and engineering standards used across all govpilot repositories.

## Quick Links

| Resource | URL |
|---|---|
| Confluence Guide — GitFlow, CI/CD & Release Process | [govpilot.atlassian.net/wiki/spaces/KB/pages/877461523](https://govpilot.atlassian.net/wiki/spaces/KB/pages/877461523) |
| DOPS Jira Epic | [govpilot.atlassian.net/browse/DOPS-117](https://govpilot.atlassian.net/browse/DOPS-117) |

---

## Repository Purpose

This `govpilot/.github` repository holds the **single source of truth** for all govpilot and SDL (Spatial Data Logic) repositories:

- **CI/CD workflow templates** — teams copy these into their own repos' `.github/workflows/` directory
- **PR template** — standardized pull request template with Jira ticket linking and release note fields
- **`product-groups.json`** — mapping of govpilot repositories to product groups, used by the weekly versioning workflow

All govpilot repositories should consume these templates rather than maintaining their own. When a template is updated here, teams pull the latest version into their repo.

---

## How to Use These Templates

Follow these steps to add CI/CD to a govpilot repository:

### 1. Copy the workflow files

```bash
mkdir -p .github/workflows
```

Copy each workflow from [`workflow-templates/`](./workflow-templates/) to `.github/workflows/` in your repository:

| Source (in this repo) | Destination (in your repo) |
|---|---|
| `workflow-templates/deploy-staging.yml` | `.github/workflows/deploy-staging.yml` |
| `workflow-templates/deploy-production.yml` | `.github/workflows/deploy-production.yml` |
| `workflow-templates/jira-close-ticket.yml` | `.github/workflows/jira-close-ticket.yml` |
| `workflow-templates/weekly-version-tag.yml` | `.github/workflows/weekly-version-tag.yml` |
| `workflow-templates/pr-lint.yml` | `.github/workflows/pr-lint.yml` |
| `pull_request_template.md` | `.github/pull_request_template.md` |

### 2. Customize the build & deploy steps

Open `deploy-staging.yml` and `deploy-production.yml` and replace the placeholder build/deploy steps with your stack's actual commands (Docker, npm, Kubernetes, etc.). Comments in the files guide you.

### 3. Set repo-level variables

In **GitHub → Settings → Variables → Repository variables**, add:

| Variable | Description |
|---|---|
| `PRODUCT_NAME` | Your product slug (e.g. `core`, `desktop`, `sdl-core`) — used in version tags |
| `BEAMER_SEGMENT` | The Beamer audience segment for release notes (e.g. `core`, `mobile`) |

> **Note:** `PRODUCT_NAME` and `BEAMER_SEGMENT` are **repo-level variables**, not org secrets. Each repo must set these individually.

### 4. Configure environments in GitHub Settings

Navigate to **GitHub → Settings → Environments** and create two environments:

- **`staging`** — no protection rules required (auto-deploy on merge to `staging`)
- **`production`** — for **gated repos**, add Required reviewers; for non-gated repos, leave protection rules empty

### 5. Register your repo in `product-groups.json`

Open a PR against this repo to add your repository under the correct product group in [`product-groups.json`](./product-groups.json). This is required for the weekly version tagging workflow to include your repo.

---

## Workflow Reference

| File | Trigger | Effect |
|---|---|---|
| `deploy-staging.yml` | Push/merge to `staging` branch | Builds and deploys to the staging environment |
| `deploy-production.yml` | Push/merge to `main` branch | Builds and deploys to the production environment (gated repos require approval) |
| `jira-close-ticket.yml` | Push/merge to `main` branch | Reads the Jira ticket key from the PR body, transitions the ticket to Done |
| `weekly-version-tag.yml` | Cron: every Monday 09:00 UTC | Creates a weekly version tag, publishes a GitHub Release, posts Beamer release notes |
| `pr-lint.yml` | Pull request opened/updated | Validates that the PR title or body contains a valid Jira ticket key (e.g. `COR-123`) |

---

## Secrets Reference

The following secrets must be configured at the **organization level** (`govpilot` org → Settings → Secrets → Actions). Individual repos inherit them automatically.

| Secret | Description |
|---|---|
| `JIRA_BASE_URL` | Jira workspace URL, e.g. `https://govpilot.atlassian.net` |
| `JIRA_USER_EMAIL` | Email of the Jira service account used for API calls |
| `JIRA_API_TOKEN` | Jira API token for the service account |
| `JIRA_DONE_TRANSITION_ID` | Numeric ID of the "Done" transition in Jira (may differ per project — see note below) |
| `JIRA_RELEASE_NOTE_FIELD_ID` | Jira custom field ID that stores the release note text |
| `BEAMER_API_KEY` | Beamer API key for publishing release notes |
| `STAGING_DEPLOY_KEY` | Deploy credential for the staging environment (format depends on hosting provider) |
| `PROD_DEPLOY_KEY` | Deploy credential for the production environment |

> **`JIRA_DONE_TRANSITION_ID` note:** The transition ID may differ between Jira projects. DevOps should query each project's available transitions (`GET /rest/api/3/issue/{issueKey}/transitions`) and decide whether to store one global ID or handle per-project logic inside the workflow.

---

## Product Groups (`product-groups.json`)

[`product-groups.json`](./product-groups.json) maps govpilot repositories to logical product groups. The weekly versioning workflow (`weekly-version-tag.yml`) reads this file to determine which repos belong to each product, so it can create the correct version tag (e.g. `core-v2026.07`) and post release notes to the right Beamer segment.

**To add a repository to a product group:**

1. Fork/branch this repo
2. Open `product-groups.json`
3. Find the correct product slug (e.g. `core`, `integrations`)
4. Add your repo as `"govpilot/your-repo-name"` to the `repos` array
5. Open a PR with the `DOPS` Jira ticket key in the title

If your repo doesn't fit an existing product group, add a new group entry and include a note in your PR.

---

## Branching Rules — Quick Reference

These 10 rules summarize the full GitFlow process documented in the [Confluence guide](https://govpilot.atlassian.net/wiki/spaces/KB/pages/877461523).

1. `main` — production. Never commit directly. Protected branch.
2. `staging` — pre-production integration branch. Auto-deploys to staging on merge.
3. Feature branches — always branch from `main`, named `feature/TICKET-KEY-short-description`.
4. Bug fix branches — branch from `main`, named `fix/TICKET-KEY-short-description`.
5. All PRs must target `main`. Never open a PR directly to `staging`.
6. Every PR must include a Jira ticket key in the title or body. The `pr-lint` check enforces this.
7. After a PR merges to `main`, the deployer manually fast-forward merges `main` → `staging` to update staging.
8. Production deploy happens automatically on merge to `main` (non-gated) or after approval (gated repos).
9. Jira ticket is automatically transitioned to Done on merge to `main`.
10. Weekly version tags are created every Monday. Do not create manual version tags.

---

## Contact

For questions, open a ticket in [DOPS](https://govpilot.atlassian.net/browse/DOPS-117).
