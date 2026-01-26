# ROA Tests Action

A reusable GitHub Action for running API and UI tests with the Ring of Automation (ROA) framework. This action provides intelligent test splitting, parallel execution, Selenium Grid support for UI tests, and automatic Allure report generation with GitHub Pages publishing.

Perfect for projects that extend from `roa-parent` and use `roa-libraries`.

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

---

## 📋 Table of Contents

- [Features](#-features)
- [Prerequisites](#-prerequisites)
- [Quick Start](#-quick-start)
- [Usage Examples](#-usage-examples)
  - [API Tests](#api-tests)
  - [UI Tests](#ui-tests)
- [Inputs](#-inputs)
- [Secrets](#-secrets)
- [Outputs](#-outputs)
- [How It Works](#-how-it-works)
- [Advanced Configuration](#-advanced-configuration)
- [Example Projects](#-example-projects)
- [Video Tutorial](#-video-tutorial)
- [License](#-license)

---

## ✨ Features

- **🔀 Intelligent Test Splitting** - Automatically distributes tests across parallel jobs for faster execution
- **⚡ Parallel Execution** - Runs tests concurrently with configurable parallelism
- **🌐 Selenium Grid Support** - Built-in Selenium Grid for UI tests with auto-scaling
- **📊 Allure Reports** - Automatic generation and aggregation of beautiful test reports
- **📄 GitHub Pages Publishing** - One-click publishing of test reports to GitHub Pages
- **🏷️ Tag-Based Filtering** - Run specific test subsets using JUnit tags (e.g., `@Smoke`, `@Regression`)
- **🎯 Multi-Module Support** - Works with both single and multi-module Maven projects
- **🔧 Highly Configurable** - Extensive customization options for test execution

---

## 📦 Prerequisites

Your project must:
- Extend from `roa-parent` POM
- Use `roa-libraries` for test automation (available on Maven Central)
- Be a Maven-based Java project
- Use JUnit 5 for test execution

### Enable GitHub Pages

To view published Allure reports, you must enable GitHub Pages in your repository:

1. **Create the `gh-pages` branch** (if it doesn't exist):
   ```bash
   git checkout --orphan gh-pages
   git reset --hard
   git commit --allow-empty -m "Initialize gh-pages branch"
   git push origin gh-pages
   git checkout main
   ```

2. Go to your repository **Settings** → **Pages**
3. Under **Source**, select **Deploy from a branch**
4. Select branch: **gh-pages** and folder: **/ (root)**
5. Click **Save**

After your first test run completes, reports will be available at: `https://<your-username>.github.io/<your-repo>/<run-number>/`

---

## 🚀 Quick Start

Create a workflow file in your repository (e.g., `.github/workflows/tests.yml`):

```yaml
name: Run Tests

permissions:
  contents: write

on:
  workflow_dispatch:

jobs:
  test:
    uses: CyborgCodeSyndicate/roa-github-workflows/.github/workflows/roa-tests.yml@v1.0.5
    with:
      kind: api
```

> **Note:** The `maven_user` and `maven_token` secrets are only required for CyborgCodeSyndicate organization members during development. Public users can omit these as ROA libraries are available on Maven Central.

---

## 📚 Usage Examples

<details>
<summary><b>API Tests</b></summary>

### Basic API Test Execution

```yaml
name: API Tests

permissions:
  contents: write

on:
  workflow_dispatch:

jobs:
  run:
    uses: CyborgCodeSyndicate/roa-github-workflows/.github/workflows/roa-tests.yml@v1.0.5
    with:
      kind: api
```

### API Tests with Tag Filtering

```yaml
name: API Tests

permissions:
  contents: write

on:
  workflow_dispatch:
    inputs:
      tagsInclude:
        description: "Tags to include (comma-separated)"
        required: false
        default: "Regression"
      tagsExclude:
        description: "Tags to exclude (comma-separated)"
        required: false
        default: "Flaky"

jobs:
  run:
    uses: CyborgCodeSyndicate/roa-github-workflows/.github/workflows/roa-tests.yml@v1.0.5
    with:
      kind: api
      tags_include: ${{ github.event.inputs.tagsInclude }}
      tags_exclude: ${{ github.event.inputs.tagsExclude }}
```

### API Tests with Custom Parallelism

```yaml
name: API Tests

permissions:
  contents: write

on:
  workflow_dispatch:

jobs:
  run:
    uses: CyborgCodeSyndicate/roa-github-workflows/.github/workflows/roa-tests.yml@v1.0.5
    with:
      kind: api
      max_methods_per_job: 30
      junit_threads_per_job: 10
      max_runners: 5
```

</details>

<details>
<summary><b>UI Tests</b></summary>

### Basic UI Test Execution

```yaml
name: UI Tests

permissions:
  contents: write

on:
  workflow_dispatch:

jobs:
  run:
    uses: CyborgCodeSyndicate/roa-github-workflows/.github/workflows/roa-tests.yml@v1.0.5
    with:
      kind: ui
```

### UI Tests with Tag Filtering

```yaml
name: UI Tests

permissions:
  contents: write

on:
  workflow_dispatch:
    inputs:
      tagsInclude:
        description: "Tags to include (comma-separated)"
        required: false
        default: "Regression"
      tagsExclude:
        description: "Tags to exclude (comma-separated)"
        required: false
        default: "Flaky"

jobs:
  run:
    uses: CyborgCodeSyndicate/roa-github-workflows/.github/workflows/roa-tests.yml@v1.0.5
    with:
      kind: ui
      tags_include: ${{ github.event.inputs.tagsInclude }}
      tags_exclude: ${{ github.event.inputs.tagsExclude }}
```

### UI Tests with Custom Grid Configuration

```yaml
name: UI Tests

permissions:
  contents: write

on:
  workflow_dispatch:

jobs:
  run:
    uses: CyborgCodeSyndicate/roa-github-workflows/.github/workflows/roa-tests.yml@v1.0.5
    with:
      kind: ui
      grid_node_max_sessions: 3
      headless: true
      use_grid: true
```

### UI Tests on Pull Request

```yaml
name: UI Tests on PR

permissions:
  contents: write

on:
  pull_request:
    branches: [ main ]

jobs:
  run:
    uses: CyborgCodeSyndicate/roa-github-workflows/.github/workflows/roa-tests.yml@v1.0.5
    with:
      kind: ui
      tags_include: Smoke
      headless: true
```

</details>

<details>
<summary><b>Multi-Module Projects</b></summary>

If your project has multiple Maven modules, specify the module containing your tests:

```yaml
name: API Tests - Specific Module

permissions:
  contents: write

on:
  workflow_dispatch:

jobs:
  run:
    uses: CyborgCodeSyndicate/roa-github-workflows/.github/workflows/roa-tests.yml@v1.0.5
    with:
      kind: api
      project_module: api-test-framework
```

</details>

<details>
<summary><b>Scheduled Test Runs</b></summary>

Run tests on a schedule (e.g., nightly regression):

```yaml
name: Nightly Regression Tests

permissions:
  contents: write

on:
  schedule:
    - cron: '0 2 * * *'  # Run at 2 AM daily
  workflow_dispatch:

jobs:
  api-tests:
    uses: CyborgCodeSyndicate/roa-github-workflows/.github/workflows/roa-tests.yml@v1.0.5
    with:
      kind: api
      tags_include: Regression
      tags_exclude: Flaky,WIP
```

</details>

---

## 🔧 Inputs

<details>
<summary><b>Core Configuration</b></summary>

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `kind` | **Yes** | - | Test type: `api` or `ui` |
| `java_version` | No | `17` | Java version for test execution |
| `test_profiles` | No | `e2e` | Maven profiles (comma/space/semicolon separated) |
| `tags_include` | No | `""` | JUnit tags to include (comma-separated) |
| `tags_exclude` | No | `""` | JUnit tags to exclude (comma-separated) |
| `project_module` | No | `""` | Maven module path (leave empty for single-module projects) |

</details>

<details>
<summary><b>Parallelism & Splitting</b></summary>

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `max_methods_per_job` | No | `20` | Maximum test methods per parallel job |
| `parallel_methods` | No | `true` | Enable method-level parallelization |
| `max_runners` | No | `10` | Maximum number of parallel runners |
| `junit_threads_per_job` | No | `5` | JUnit parallel threads per job |

</details>

<details>
<summary><b>UI & Selenium Grid (UI tests only)</b></summary>

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `use_grid` | No | `true` | Enable Selenium Grid for UI tests |
| `headless` | No | `true` | Run browsers in headless mode |
| `grid_hub_image` | No | `selenium/hub:4.10.0` | Docker image for Selenium Hub |
| `grid_node_image` | No | `selenium/node-chrome:4.10.0` | Docker image for Chrome nodes |
| `grid_node_max_sessions` | No | `5` | Max parallel sessions per node |
| `grid_node_shm` | No | `2g` | Shared memory size for nodes |

</details>

<details>
<summary><b>Reporting & Publishing</b></summary>

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `allure_version` | No | `2.29.0` | Allure CLI version |
| `pages_publish` | No | `true` | Publish Allure report to GitHub Pages |

</details>

<details>
<summary><b>Advanced Maven Configuration</b></summary>

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `maven_server_ids` | No | `github-roa-libraries,github-roa-plugins` | Maven server IDs (CyborgCodeSyndicate org only) |
| `mvn_extra_split_args` | No | `""` | Extra arguments for Maven split command |
| `mvn_extra_test_args` | No | `""` | Extra arguments for Maven test command |
| `split_profile` | No | `execution-setup` | Maven profile for test splitting |

</details>

---

## 🔐 Secrets

| Secret | Required | Description |
|--------|----------|-------------|
| `maven_user` | No | GitHub username (CyborgCodeSyndicate org only) |
| `maven_token` | No | GitHub PAT (CyborgCodeSyndicate org only) |
| `pages_token` | No | GitHub PAT for publishing to Pages (falls back to `github.token`) |

> **Note:** The `maven_user` and `maven_token` secrets are **only required for CyborgCodeSyndicate organization members** during development. Public users do not need these secrets as ROA libraries are published to Maven Central.

### For CyborgCodeSyndicate Members Only

If you're part of the CyborgCodeSyndicate organization and need to test unreleased versions:

1. Go to your repository **Settings** → **Secrets and variables** → **Actions**
2. Click **New repository secret**
3. Add the following secrets:
   - `GH_PACKAGES_USER`: Your GitHub username
   - `GH_PACKAGES_PAT`: A Personal Access Token with `read:packages` scope

---

## 📤 Outputs

The action automatically:
- ✅ Generates and merges Allure test reports
- 📊 Publishes reports to GitHub Pages (if enabled)
- 📝 Adds a summary with report link to the workflow run
- 💾 Uploads test artifacts for debugging

Access your reports at: `https://<owner>.github.io/<repo>/<run_number>/`

---

## 🔍 How It Works

<details>
<summary><b>Execution Flow</b></summary>

1. **Split** - Analyzes your test suite and creates optimal test groups
2. **Matrix** - Generates a dynamic matrix for parallel execution
3. **Run** - Executes tests in parallel with:
   - Maven setup and dependency resolution
   - Selenium Grid (for UI tests)
   - Configured JUnit parallelism
4. **Merge** - Aggregates Allure results from all parallel jobs
5. **Publish** - Generates final report and publishes to GitHub Pages

</details>

<details>
<summary><b>Test Splitting Strategy</b></summary>

The action uses intelligent test splitting to:
- Distribute tests evenly across runners
- Balance execution time
- Maximize parallelism while respecting resource limits
- Support both class and method-level splitting

</details>

<details>
<summary><b>Selenium Grid Architecture</b></summary>

For UI tests, the action:
- Starts a Selenium Hub container
- Auto-scales Chrome node containers based on test count
- Configures parallel session limits
- Tears down infrastructure after tests complete

</details>

---

## ⚙️ Advanced Configuration

<details>
<summary><b>Custom Test Profiles</b></summary>

```yaml
with:
  kind: api
  test_profiles: e2e,integration,smoke
```

</details>

<details>
<summary><b>Fine-Tuning Parallelism</b></summary>

```yaml
with:
  kind: api
  max_methods_per_job: 50      # More tests per job
  junit_threads_per_job: 15    # More threads per job
  max_runners: 3               # Fewer parallel runners
```

</details>

<details>
<summary><b>Disabling GitHub Pages Publishing</b></summary>

```yaml
with:
  kind: api
  pages_publish: false
```

</details>

<details>
<summary><b>Using Different Java Version</b></summary>

```yaml
with:
  kind: api
  java_version: "21"
```

</details>

<details>
<summary><b>Running Without Selenium Grid</b></summary>

```yaml
with:
  kind: ui
  use_grid: false  # Tests will run locally
```

</details>

---

## 📖 Example Projects

See the ROA Tests action in action:

- 🔗 [Example API Test Execution](https://github.com/CyborgCodeSyndicate/example-api-tests/actions)
- 🔗 [Example UI Test Execution](https://github.com/CyborgCodeSyndicate/example-ui-tests/actions)

---

## 🎥 Video Tutorial

📺 [Watch: Getting Started with ROA Tests Action](#)

*Video tutorial coming soon*

---

## 📄 License

Licensed under [Apache License 2.0](https://opensource.org/licenses/Apache-2.0)

---

## 👨‍💻 Author

**Cyborg Code Syndicate 💍👨💻**

For issues, questions, or contributions, please visit the [GitHub repository](https://github.com/CyborgCodeSyndicate/roa-github-workflows).