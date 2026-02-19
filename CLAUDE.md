# CLAUDE.md — govpilot/.github

This file gives Claude (and any AI assistant) the context needed to work effectively in this repository.

---

## What This Repository Is

`govpilot/.github` is the **single source of truth** for shared GitHub Actions workflow templates, PR templates, and engineering standards across all GovPilot and SDL repositories.

Files here are **templates** — they are copied into individual repos' `.github/workflows/` directories and customized per-stack. This repo does not run workflows itself.

---

## Company Structure

The parent company is **GovPilot**. GovPilot acquired **SDL (Spatial Data Logic)**, which had previously acquired **Hedgerow**. All three product lines live under the `govpilot` GitHub org.

| Company | Products |
|---|---|
| GovPilot | Jet (Core Platform), GovAlert, Integrations/Payments |
| SDL | Connect (web portal), Desktop, Field/Mobile 4 |
| Hedgerow | Desktop/Windows infrastructure (acquired by SDL) |

---

## Key Files

| File | Purpose |
|---|---|
| `workflow-templates/deploy-staging.yml` | CI/CD: deploy on merge to `staging` branch |
| `workflow-templates/deploy-production.yml` | CI/CD: deploy on merge to `main` (supports gated approval) |
| `workflow-templates/jira-close-ticket.yml` | Auto-close Jira ticket on merge to `main` |
| `workflow-templates/weekly-version-tag.yml` | Weekly version tag + GitHub Release + Beamer release notes |
| `workflow-templates/pr-lint.yml` | Enforce Jira ticket key in every PR |
| `pull_request_template.md` | Standard PR template with Jira + Release Note fields |
| `product-groups.json` | Maps every govpilot repo to a product group for versioning |
| `README.md` | Onboarding guide for teams adopting these workflows |

---

## Product Groups (`product-groups.json`)

This file is the **most frequently updated** file in this repo. It maps repositories to product groups so the weekly versioning workflow knows which repos belong to which product.

### Current product groups

| Slug | Product Name | Company | Key Repos |
|---|---|---|---|
| `govpilot-core` | GovPilot Core (Jet) | GovPilot | `jet`, `pphtml`, `InternalApi`, `GPDataService`, etc. |
| `govpilot-integrations` | Integrations & Payments | GovPilot | `GPPayments`, `GPIntegrationAPI`, etc. |
| `govpilot-govAlert` | GovAlert | GovPilot | `GovAlert2`, `GPGovAlertService` |
| `sdl-desktop` | SDL Desktop | SDL | `sdldesktop` |
| `sdl-connect` | SDL Connect | SDL | `sdl-mono`, `document-search`, `cloud_functions`, `sdlcitizen2`, `von-karajan`, `SDLPortalSync`, `pit` |
| `sdl-field-mobile` | SDL Field / Mobile 4 | SDL | `sdlmobile-rn` |
| `hedgerow-desktop` | Hedgerow Desktop | Hedgerow | `sdl-ad-user-mgmt`, `Framework.WPF`, `Framework.Core`, `signing-binaries` |

### Rules for product-groups.json
- **Do not include devops, tooling, QA, or test repos** — they don't get versioned or release-noted
- Archived repos are excluded
- Each repo should appear in **exactly one** product group
- When adding a new repo, open a PR with a `DOPS-XXX` ticket key in the title

---

## Jira Project Keys

| Key | Project | Used by |
|---|---|---|
| `COR` | GovPilot Core | Jet/Core platform tickets |
| `BPM` | Business Process Mgmt | Workflow/BPM tickets |
| `INT` | Integrations | Payments, integration tickets |
| `GA` | GovAlert | GovAlert tickets |
| `DSK` | Desktop | SDL Desktop tickets |
| `SDL` | SDL Connect | Connect platform tickets |
| `MOB` | Mobile | SDL Mobile/Field app tickets |
| `HDR` | Hedgerow Desktop | Hedgerow Windows infra tickets |
| `DOPS` | DevOps | CI/CD, infrastructure, tooling tickets |
| `MUI` | Mobile & UI | (legacy — being migrated) |

---

## GitFlow Rules (Quick Reference)

1. `main` → production. **Never commit directly.** Branch protected.
2. `staging` → pre-production. Auto-deploys on merge.
3. Feature branches: always from `main`, named `feature/TICKET-KEY-short-description`
4. Fix branches: from `main`, named `fix/TICKET-KEY-short-description`
5. All PRs target `main`. Never open a PR directly to `staging`.
6. Every PR must include a Jira ticket key — `pr-lint` enforces this.
7. After merging to `main`, deployer fast-forward merges `main → staging`.
8. Production deploy: automatic (non-gated) or after approval (gated repos).
9. Jira ticket auto-closes on merge to `main`.
10. Weekly version tags run every Monday — do not create manual version tags.

Full GitFlow docs: https://govpilot.atlassian.net/wiki/spaces/KB/pages/877461523

---

## Common Tasks for AI Assistants

### Adding a new repo to product-groups.json
1. Identify which product it belongs to (see the table above — ask if unclear)
2. Add `"govpilot/repo-name"` to the correct product's `repos` array
3. Do not add devops, test, or archived repos
4. Commit with a `DOPS-XXX` ticket reference

### Updating a workflow template
1. Edit the file in `workflow-templates/`
2. Update the header comment if the trigger or behavior changes
3. Update the README's Workflow Reference table if the trigger/effect changed
4. Note in the commit message which repos will need to pull the update

### Adding a new product group
1. Add a new object to the `products` array in `product-groups.json`
2. Required fields: `slug`, `name`, `company`, `beamer_segment`, `jira_projects`, `repos`
3. Add the new slug to the README's product groups table
4. Add the new slug to the CLAUDE.md product groups table

---

## Links

| Resource | URL |
|---|---|
| Confluence Guide (GitFlow, CI/CD, Release) | https://govpilot.atlassian.net/wiki/spaces/KB/pages/877461523 |
| DOPS Epic (DOPS-117) | https://govpilot.atlassian.net/browse/DOPS-117 |
| govpilot GitHub org | https://github.com/govpilot |
| Jira DOPS project | https://govpilot.atlassian.net/jira/software/projects/DOPS |
