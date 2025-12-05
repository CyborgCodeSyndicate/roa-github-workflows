# ROA GitHub Workflows

Reusable GitHub Actions for the Cyborg Code Syndicate’s Ring of Automation (ROA) test framework. This repository packages CI/CD workflows that standardize testing, reporting, and release across ROA projects.

## Table of Contents
- [Folder contents](#folder-contents)
- [Quick start](#quick-start)
  - [Reuse the tests workflow](#reuse-the-tests-workflow)
  - [Deploy (from this repo)](#deploy-from-this-repo)
- [Workflows](#workflows)
  - [ROA Tests](#roa-tests)
    - [High-level flow](#high-level-flow)
    - [Inputs (selected)](#inputs-selected)
    - [Permissions](#permissions)
    - [Implementation details](#implementation-details)
  - [Deploy](#deploy)
    - [Required repo secrets](#required-repo-secrets)
- [Customization tips](#customization-tips)
- [Author](#author)
 
---
## Overview
This repo provides two primary workflows:

- `roa-tests.yml` — reusable job for splitting and running API/UI tests in parallel, aggregating Allure results, and optionally publishing to GitHub Pages.
- `deploy.yml` — release pipeline to version-bump and deploy artifacts to GitHub Packages.

These workflows codify ROA’s CI/CD conventions so teams can adopt a consistent pipeline by referencing a pinned tag of this repository from their own workflows.

--- 

 ## Folder contents
 
 | Workflow | Description | Triggers | Jobs | Integrations |
 |---|---|---|---|---|
 | `roa-tests.yml` (reusable) | Reusable tests pipeline: dynamic test splitting, parallel execution, optional Selenium Grid for UI, Allure aggregation, and Pages publishing. Reused via `uses:` across repos. | `workflow_call` | `split` → `matrix` → `run` → `merge` → `publish` | Maven, Selenium Grid (UI), Allure, GitHub Pages |
 | `deploy.yml` | Standardizes release mechanics (version bump + deploy to GitHub Packages), aligned with ROA release flow. | `push` (main), `workflow_dispatch` | `deploy` | GitHub Packages, semantic version bumping |
 ---

## Quick start

### Reuse the tests workflow
Create a workflow (e.g., `.github/workflows/ci-tests.yml`) that calls the reusable workflow in this repo:
```yaml
name: CI Tests
on:
  pull_request:
  push:
    branches: [ main ]

jobs:
  api-tests:
    uses: CyborgCodeSyndicate/roa-github-workflows/.github/workflows/roa-tests.yml@v1
    with:
      kind: api
      test_profiles: e2e,smoke
      tags_include: smoke
      junit_threads_per_job: 5
      max_methods_per_job: 30
    secrets:
      maven_user: ${{ secrets.GH_PACKAGES_USER }}
      maven_token: ${{ secrets.GH_PACKAGES_PAT }}
      # pages_token is optional; falls back to github.token
```

UI example with Selenium Grid and Allure publish:

```yaml
jobs:
  ui-tests:
    uses: CyborgCodeSyndicate/roa-github-workflows/.github/workflows/roa-tests.yml@v1
    with:
      kind: ui
      use_grid: true
      headless: true
      grid_node_max_sessions: 5
      pages_publish: true
    secrets:
      maven_user: ${{ secrets.GH_PACKAGES_USER }}
      maven_token: ${{ secrets.GH_PACKAGES_PAT }}
      pages_token: ${{ secrets.GH_PAGES_PAT }} # optional; PAT recommended for cross-fork publishes
```

*Replace `@v1` with a released tag for this repository. Pinning to a tag is strongly recommended.*

### Deploy (from this repo)
The `deploy.yml` workflow runs on `push` to `main` and manually via the Actions tab (`workflow_dispatch`). Manual inputs:

- `version_bump`: `major | minor | patch | none` (default `none`)
- `modules`: optional comma-separated Maven modules to deploy

Required secrets for deployment:

- `GH_PACKAGES_USER`
- `GH_PACKAGES_PAT`

The workflow uses these with `CyborgCodeSyndicate/utilities/pipelines/deploy` to bump, build and deploy artifacts to GitHub Packages.

---

# Workflows

## ROA Tests

**Path:** `.github/workflows/roa-tests.yml`

### High-level flow
1. **Split**: Build test graph and produce a JSON of test groups (`max_methods_per_job`, `parallel_methods`, `max_runners`).
2. **Matrix**: Read JSON via `jq` and create a dynamic matrix.
3. **Run**: For each group, set up Maven, optionally start Selenium Grid (UI), and run the subset with configured JUnit parallelism.
4. **Merge**: Collect and merge Allure results across all matrix jobs.
5. **Publish**: Optionally publish the merged Allure report to GitHub Pages (into `gh-pages/<run_number>/`).

### Inputs (selected)
- **Git checkout**
  - `repository` (optional): Repository containing the Maven project under test (defaults to current repo)
  - `ref` (optional): Branch/tag/commit to checkout (defaults to current ref)

- **Core**
  - `kind` (required): `api` or `ui`
  - `java_version` (default `17`)
  - `project_module`: Maven module path; empty runs at repo root (no `-pl`)
  - `test_profiles` (default `e2e`): comma/space/semicolon separated
  - `tags_include`, `tags_exclude`: JUnit tags filters

- **Splitting & parallelism**
  - `max_methods_per_job` (default `20`)
  - `parallel_methods` (default `true`)
  - `max_runners` (default `10`)
  - `junit_threads_per_job` (default `5`)
  - `json_output_name` (default `grouped-tests`)

- **UI (Selenium Grid)**
  - `use_grid` (default `true`), `headless` (default `true`)
  - `grid_hub_image` (default `selenium/hub:4.10.0`)
  - `grid_node_image` (default `selenium/node-chrome:4.10.0`)
  - `grid_node_shm` (default `2g`), `grid_node_max_sessions` (default `5`)

- **Reporting & publishing**
  - `allure_version` (default `2.29.0`)
  - `pages_publish` (default `true`)

- **Maven setup**
  - `maven_server_ids` (default `github-roa-libraries,github-roa-plugins`)
  - `mvn_extra_split_args`, `mvn_extra_test_args`

- **Secrets**
  - `maven_user` (required), `maven_token` (required)
  - `pages_token` (optional; falls back to `github.token`)

### Permissions
- Sets `contents: write` to enable Pages publishing.

### Implementation details
- Uses `CyborgCodeSyndicate/utilities/pipelines/shared/setup-maven` to generate `~/.m2/settings.xml` for GitHub Packages based on `maven_server_ids` and provided secrets.
- For UI runs, a minimal Selenium Grid is started via `docker-compose` and scaled based on the test subset size and `junit_threads_per_job`.
- Allure is downloaded and invoked directly to merge and generate the report.
- Allure results are uploaded from `target/allure-results` (or `<project_module>/target/allure-results` when set).
- When `project_module` is specified, artifacts are named with the module suffix (e.g., `allure-results-{module}-{jobIndex}`) and Pages are published to `{run_number}/{module}`.
- A run summary includes a direct link to the published Allure report when `pages_publish` is true.

*Note: An `allure_results_path` input exists but the workflow computes the upload path from Maven defaults; override behavior by adjusting the upload step if your project writes to a different directory.*

---

## Deploy

**Path:** `.github/workflows/deploy.yml`

- Triggered on push to `main` and manually via `workflow_dispatch`.
- Calls `CyborgCodeSyndicate/utilities/pipelines/deploy@v1.3.2` with:
  - `server_ids`: `github-roa-github-workflows`
  - `version_bump`: from dispatch input (default `none`)
  - `deploy_server_id`: `github-roa-github-workflows`
  - `modules`: optional list from dispatch input
  - `github_token`, `github_packages_user`, `github_packages_token` from repo secrets
- Requires `permissions: contents: write` and `packages: write`.

### Required repo secrets
- `GH_PACKAGES_USER` - GitHub username or machine user
- `GH_PACKAGES_PAT` - PAT with `read:packages`, `write:packages`, and (usually) `repo`

---

## Customization tips
- **Pin versions** of reusable workflows and composite actions (e.g., `@v1.3.x`) to avoid breaking changes. The `run` job in `roa-tests.yml` currently references `setup-maven@main`; pinning to a tag is recommended for consumers.
- **Tune parallelism** using `max_methods_per_job` and `junit_threads_per_job` to balance speed vs. resource limits.
- **Target a submodule** by setting `project_module` so the workflow builds and runs only that Maven module (`-pl <module> -am`).
- **Tag filtering** with `tags_include`/`tags_exclude` lets you run `smoke`, `regression`, or other slices on demand.
- **Disable Grid** for API-only projects with `use_grid: false`.

---
## Author
**Cyborg Code Syndicate 💍👨💻**