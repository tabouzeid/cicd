# CI/CD Pipelines

Centralized reusable GitHub Actions workflows. Any project that calls one of these workflows automatically gets build, test, and dependency scanning — update this repo once, every project picks it up on the next run.

## Workflows

| Workflow | File |
|---|---|
| Gradle | `.github/workflows/gradle.yml` |
| Maven | `.github/workflows/maven.yml` |
| Python | `.github/workflows/python.yml` |
| Node.js | `.github/workflows/node.yml` |
| Ansible | `.github/workflows/ansible.yml` |

Each workflow runs two jobs:

1. **Build & Test** — compiles the project and runs its test suite
2. **Dependency Scan** — audits for known CVEs (via [Trivy](https://github.com/aquasecurity/trivy-action) + ecosystem tools) and reports outdated packages

Results appear in the Actions run log and, for security findings, in the repository's **Security > Code scanning** tab via SARIF upload.

---

## Using a workflow in your project

Create `.github/workflows/ci.yml` in the consuming repo:

### Gradle

```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:

jobs:
  ci:
    uses: <YOUR_GITHUB_ORG>/cicd/.github/workflows/gradle.yml@main
    with:
      java-version: '21'         # optional, default: 21
      java-distribution: temurin # optional, default: temurin
      build-command: ./gradlew assemble  # optional
      test-command: ./gradlew test       # optional
```

> **Tip:** Add the [ben-manes versions plugin](https://github.com/ben-manes/gradle-versions-plugin) to your `build.gradle` to enable the dependency update report:
> ```groovy
> plugins {
>     id 'com.github.ben-manes.versions' version '0.51.0'
> }
> ```

### Maven

```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:

jobs:
  ci:
    uses: <YOUR_GITHUB_ORG>/cicd/.github/workflows/maven.yml@main
    with:
      java-version: '21'   # optional
      build-goals: compile  # optional
      test-goals: verify    # optional
```

### Python

```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:

jobs:
  ci:
    uses: <YOUR_GITHUB_ORG>/cicd/.github/workflows/python.yml@main
    with:
      python-version: '3.12'            # optional
      install-command: pip install -e ".[dev]"  # optional
      test-command: pytest              # optional
      lint-command: ruff check .        # optional, set to '' to skip
```

### Node.js

```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:

jobs:
  ci:
    uses: <YOUR_GITHUB_ORG>/cicd/.github/workflows/node.yml@main
    with:
      node-version: '22'          # optional
      package-manager: npm        # optional: npm | yarn | pnpm
      build-command: npm run build # optional, set to '' to skip
      test-command: npm test       # optional
```

### Ansible

```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:

jobs:
  ci:
    uses: <YOUR_GITHUB_ORG>/cicd/.github/workflows/ansible.yml@main
    with:
      python-version: '3.12'           # optional
      ansible-version: latest          # optional, e.g. ">=9,<10"
      requirements-file: requirements.txt          # optional
      galaxy-requirements-file: requirements.yml   # optional
      lint-paths: .                    # optional
      molecule-scenario: default       # optional, set to '' to skip
```

---

## Gitea → GitHub mirror setup

The workflow is: **push to Gitea → Gitea mirrors to GitHub → GitHub Actions runs the pipeline**.

### 1. Create a GitHub personal access token

In GitHub: **Settings > Developer settings > Personal access tokens > Fine-grained tokens**

Grant the token:
- **Contents**: Read and Write
- **Workflows**: Read and Write (required to push `.github/workflows/`)

### 2. Configure the push mirror in Gitea

In your Gitea repository:

1. Go to **Settings > Repository > Mirror Settings**
2. Click **Add Push Mirror**
3. Fill in:
   - **Remote URL**: `https://github.com/<org>/<repo>.git`
   - **Authorization**: your GitHub username + the token from step 1
   - **Sync on commit**: enabled
   - **Interval**: `0h0m0s` (sync immediately on push) or a schedule like `8h`
4. Click **Add Push Mirror**

From this point, every `git push` to Gitea will automatically mirror to GitHub, which triggers the GitHub Actions pipeline.

### 3. Enable required GitHub repository settings

In each mirrored GitHub repository:

- **Settings > Actions > General**: set to *Allow all actions*
- **Settings > Code security**: enable *Dependabot alerts* and *Code scanning* to see security results

---

## How updates propagate

```
cicd repo (this repo)
  └── .github/workflows/gradle.yml  ← edit here

my-gradle-project (GitHub)
  └── .github/workflows/ci.yml
        uses: <org>/cicd/.github/workflows/gradle.yml@main
                                              ^^^^^^^^^^
                                        pinned to a branch
```

Because consuming workflows reference `@main`, any change merged to `main` in this repo takes effect on the **next pipeline run** in every consuming project — no changes needed in those repos.

To roll out a breaking change safely, push it to a branch first, test against one project using `@my-branch`, then merge to `main`.

---

## Dependency scanning summary

| Ecosystem | CVE detection | Outdated package report |
|---|---|---|
| Gradle | Trivy + GitHub dependency graph | `dependencyUpdates` task (optional plugin) |
| Maven | Trivy + GitHub dependency graph | `mvn versions:display-dependency-updates` |
| Python | pip-audit + Trivy | `pip list --outdated` |
| Node.js | npm/yarn/pnpm audit + Trivy | `npm/yarn/pnpm outdated` |
| Ansible | pip-audit + Trivy | `pip list --outdated` |

Security findings are uploaded as SARIF and appear in **Security > Code scanning** on each GitHub repository.
