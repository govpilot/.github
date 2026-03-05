# govpilot Engineering — Shared Workflow Templates & Standards

This repository is the canonical source of GitHub Actions workflow templates, PR templates, and engineering standards used across all govpilot repositories.

## Quick Links

| Resource | URL |
|---|---|
| Confluence Guide — GitFlow, CI/CD & Release Process | [govpilot.atlassian.net/wiki/spaces/KB/pages/877461523](https://govpilot.atlassian.net/wiki/spaces/KB/pages/877461523) |
| Confluence — Automated Deployment Pipeline Plan | [govpilot.atlassian.net/wiki/spaces/KB/pages/889323523](https://govpilot.atlassian.net/wiki/spaces/KB/pages/889323523) |
| DOPS Jira Epic | [govpilot.atlassian.net/browse/DOPS-117](https://govpilot.atlassian.net/browse/DOPS-117) |

---

## Repository Purpose

This `govpilot/.github` repository holds the **single source of truth** for all govpilot and SDL (Spatial Data Logic) repositories:

- **CI/CD workflow templates** — teams copy these into their own repos' `.github/workflows/` directory
- **`jira-done-merge.yml`** — org-level workflow that triggers production deploys across multiple repos when a Jira ticket is moved to Done
- **PR template** — standardized pull request template with Jira ticket linking and release note fields
- **`product-groups.json`** — mapping of govpilot repositories to product groups, used by the weekly versioning workflow

All govpilot repositories should consume these templates rather than maintaining their own. When a template is updated here, teams pull the latest version into their repo.

---

## Automated Deployment Pipeline (`jira-done-merge.yml`)

This workflow lives in `.github/workflows/jira-done-merge.yml` and is triggered by a Jira Automation Rule when a GPDA ticket transitions to **Done**. It merges all linked release PRs across repos after a human approves.

### Flow

```
GPDA ticket → Done
      ↓
Jira Automation Rule fires
      ↓
jira-done-merge.yml triggered via workflow_dispatch
      ↓
Calls Jira dev-status API → gets all linked open PRs
  (filters: OPEN + non-draft + release/** or hotfix/** branches only)
      ↓
⏸️  WAITING — GitHub notifies reviewers (environment: production-approval)
      ↓
Reviewer clicks Approve in GitHub Actions UI
      ↓
For each PR (in parallel, fail-fast: false):
  1. Version check — skip if repo already has a newer tag on main
  2. gh pr merge --admin → merge PR
      ↓
Each repo's cicd.yml deploy-to-prod fires automatically
```

### Triggering manually

Navigate to **Actions → Merge all PRs on Jira Done → Run workflow** and provide:

| Input | Description |
|---|---|
| `jira_key` | Jira issue key, e.g. `GPDA-2778` |
| `issue_id` | Jira issue numeric ID (from the issue URL or API) |
| `allowed_repos` | Optional — comma-separated repo names to limit scope, e.g. `eForms,GPEmailService`. Leave empty for all linked repos. |

### Edge case protections

| Protection | Behaviour |
|---|---|
| Non-release PRs filtered out | Only PRs from `release/**` or `hotfix/**` branches are included — `feature/**` → `develop` PRs are ignored |
| Draft PRs filtered out | PRs with `isDraft: true` are excluded |
| Version check | Before merging, the PR's branch version (e.g. `release/1.2.3` → `1.2.3`) is compared against the latest semver tag on the repo. If main is already ahead, that repo is skipped — other repos continue |
| Per-repo isolation | Matrix jobs use `fail-fast: false` — a failure or skip in one repo does not block others |

### Required GitHub Environment

Create `production-approval` in **govpilot/.github → Settings → Environments**:
- Add required reviewers (tech leads, DevOps)
- Optionally enable "Prevent self-review"

---

## How to Use the CI/CD Templates

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

### 2. Add the `close-jira` job to your `cicd.yml`

Add the following job to your existing `.github/workflows/cicd.yml`. It runs after `deploy-to-prod` succeeds and transitions the linked Jira ticket to Done. The `if` condition mirrors the `deploy-to-prod` trigger so both fire on the same event.

```yaml
  close-jira:
    if: >
      github.event.pull_request.merged == true &&
      (github.ref == 'refs/heads/main' || github.ref == 'refs/heads/master') &&
      (startsWith(github.event.pull_request.head.ref, 'release/') ||
       startsWith(github.event.pull_request.head.ref, 'hotfix/'))
    needs: [deploy-to-prod]
    runs-on: ubuntu-latest
    steps:
      - name: Extract Jira Key from PR title
        id: key
        run: |
          KEY=$(echo "${{ github.event.pull_request.title }}" \
            | grep -oE '[A-Z]+-[0-9]+' | head -1)
          echo "key=$KEY" >> $GITHUB_OUTPUT

      - name: Transition Jira issue to Done
        if: steps.key.outputs.key != ''
        uses: atlassian/gajira-transition@v3
        with:
          issue: ${{ steps.key.outputs.key }}
          transition: "Done"
        env:
          JIRA_BASE_URL: ${{ secrets.JIRA_BASE_URL }}
          JIRA_USER_EMAIL: ${{ secrets.JIRA_USER_EMAIL }}
          JIRA_API_TOKEN: ${{ secrets.JIRA_API_TOKEN }}
```

**Required PR title convention** — the Jira key must be present:

```
✅  GPDA-2778: Remove feature flag from PPHTML
✅  GPDA-2778 - Remove feature flag
❌  Remove feature flag from PPHTML
```

### 3. Customize the build & deploy steps

Open `deploy-staging.yml` and `deploy-production.yml` and replace the placeholder build/deploy steps with your stack's actual commands. Comments in the files guide you.

### 4. Set repo-level variables

In **GitHub → Settings → Variables → Repository variables**, add:

| Variable | Description |
|---|---|
| `PRODUCT_NAME` | Your product slug (e.g. `core`, `desktop`, `sdl-core`) — used in version tags |
| `BEAMER_SEGMENT` | The Beamer audience segment for release notes (e.g. `core`, `mobile`) |

### 5. Register your repo in `product-groups.json`

Open a PR against this repo to add your repository under the correct product group in [`product-groups.json`](./product-groups.json). This is required for the weekly version tagging workflow.

---

## Workflow Reference

| File | Location | Trigger | Effect |
|---|---|---|---|
| `jira-done-merge.yml` | `govpilot/.github` | `workflow_dispatch` (via Jira Automation) | Fetches linked PRs, gates on human approval, merges all in parallel |
| `deploy-staging.yml` | each repo | Push/merge to `staging` branch | Builds and deploys to the staging environment |
| `deploy-production.yml` | each repo | Push/merge to `main` branch | Builds and deploys to production (gated repos require approval) |
| `jira-close-ticket.yml` | each repo | PR merged to `main`/`master` | Reads Jira key from PR title, transitions ticket to Done |
| `weekly-version-tag.yml` | each repo | Cron: every Monday 09:00 UTC | Creates a weekly version tag, publishes GitHub Release, posts Beamer release notes |
| `pr-lint.yml` | each repo | Pull request opened/updated | Validates that the PR title or body contains a valid Jira ticket key |

---

## Secrets Reference

The following secrets must be configured at the **organization level** (`govpilot` org → Settings → Secrets → Actions).

| Secret | Description | Used by |
|---|---|---|
| `JIRA_BASE_URL` | `https://govpilot.atlassian.net` | `close-jira` job, `jira-done-merge.yml` |
| `JIRA_USER_EMAIL` | Email of the Jira service account | `close-jira` job, `jira-done-merge.yml` |
| `JIRA_API_TOKEN` | Jira API token for the service account | `close-jira` job, `jira-done-merge.yml` |
| `GH_PAT_MERGE` | Classic GitHub PAT (`repo` + `workflow` scope) from a service account with write access to all repos | `jira-done-merge.yml` |
| `JIRA_DONE_TRANSITION_ID` | Numeric ID of the "Done" transition in Jira (GPDA: `31`) | `close-jira` job |
| `JIRA_RELEASE_NOTE_FIELD_ID` | Jira custom field ID for release note text | `close-jira` job |
| `BEAMER_API_KEY` | Beamer API key for publishing release notes | `weekly-version-tag.yml` |
| `STAGING_DEPLOY_KEY` | Deploy credential for staging | `deploy-staging.yml` |
| `PROD_DEPLOY_KEY` | Deploy credential for production | `deploy-production.yml` |

> **`GH_PAT_MERGE` note:** Must be a Classic PAT (not fine-grained) owned by a dedicated service account with at least **Write** access to all govpilot repos. Do not use a personal account.

> **`JIRA_DONE_TRANSITION_ID` note:** The transition ID may differ between Jira projects. Confirmed for GPDA: `31`. Query other projects via `GET /rest/api/3/issue/{issueKey}/transitions`.

---

## Product Groups (`product-groups.json`)

[`product-groups.json`](./product-groups.json) maps govpilot repositories to logical product groups. The weekly versioning workflow reads this file to determine which repos belong to each product.

**To add a repository:**

1. Fork/branch this repo
2. Open `product-groups.json`
3. Add your repo to the correct product group
4. Open a PR with the `DOPS` Jira ticket key in the title

---

## Branching Rules — Quick Reference

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
