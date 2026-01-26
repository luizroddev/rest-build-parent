# CI/CD Workflows

This project provides **three reusable GitHub Actions workflows** that any Minerest repository can call. They handle building, deploying snapshots, and releasing your project — so you don't have to write CI/CD from scratch.

All workflows live in `.github/workflows/` and are invoked from other repositories using `workflow_call`.

---

## Before You Start

Every workflow requires a repository secret called **`PACKAGES_TOKEN`**. This is a GitHub Personal Access Token (PAT) that authenticates Maven against GitHub Packages.

| Workflow | Required PAT Scopes |
|---|---|
| Build | `read:packages` |
| Deploy Snapshot | `read:packages`, `write:packages` |
| Release | `read:packages`, `write:packages`, `repo` |

To set it up, go to your repository on GitHub: **Settings > Secrets and variables > Actions > New repository secret**, name it `PACKAGES_TOKEN`, and paste your PAT.

---

## 1. Build (`ci-build.yml`)

**Purpose:** Compile your code and run tests. Use this as your main CI check on every push or pull request.

### What it does, step by step

1. Checks out your code.
2. Sets up Java (default: Java 22, Temurin).
3. Runs `mvn clean package -DskipTests` to compile.
4. Runs `mvn test` to execute your test suite.

If any step fails, the workflow fails and you'll see the error in the Actions tab.

### How to use it

Create a file `.github/workflows/build.yml` in your repository:

```yaml
name: Build

on:
  push:
    branches: [master]
  pull_request:
    branches: [master]

jobs:
  build:
    uses: luizroddev/rest-build-parent/.github/workflows/ci-build.yml@master
    secrets:
      PACKAGES_TOKEN: ${{ secrets.PACKAGES_TOKEN }}
```

### Optional inputs

| Input | Default | Description |
|---|---|---|
| `java-version` | `22` | Java version (`17` or `22`) |
| `java-distribution` | `temurin` | JDK distribution |
| `maven-args` | _(empty)_ | Extra arguments passed to every Maven command |

Example with custom inputs:

```yaml
jobs:
  build:
    uses: luizroddev/rest-build-parent/.github/workflows/ci-build.yml@master
    with:
      java-version: '17'
      maven-args: '-pl my-module'
    secrets:
      PACKAGES_TOKEN: ${{ secrets.PACKAGES_TOKEN }}
```

---

## 2. Deploy Snapshot (`ci-deploy-snapshot.yml`)

**Purpose:** Publish a SNAPSHOT version of your artifact to GitHub Packages. Use this to share work-in-progress builds with other modules that depend on yours.

### What it does, step by step

1. Checks out your code.
2. Sets up Java.
3. **Validates** that your `pom.xml` version ends with `-SNAPSHOT`. If it doesn't, the workflow fails immediately — this prevents accidental snapshot deploys of release versions.
4. Runs `mvn clean deploy -DskipTests` to compile and publish the artifact.

After it succeeds, other projects can pull your latest snapshot by running `mvn -U clean install`.

### How to use it

Create a file `.github/workflows/deploy-snapshot.yml` in your repository:

```yaml
name: Deploy Snapshot

on:
  push:
    branches: [master]

jobs:
  deploy:
    uses: luizroddev/rest-build-parent/.github/workflows/ci-deploy-snapshot.yml@master
    secrets:
      PACKAGES_TOKEN: ${{ secrets.PACKAGES_TOKEN }}
```

### Optional inputs

Same as the Build workflow: `java-version`, `java-distribution`, and `maven-args`.

### Important

- Your `pom.xml` version **must** end with `-SNAPSHOT` (e.g., `1.0.0-SNAPSHOT`). Otherwise the workflow will fail on purpose.
- SNAPSHOT versions can be overwritten freely — every deploy replaces the previous snapshot.

---

## 3. Release (`ci-release.yml`)

**Purpose:** Perform a full release. This is the most complete workflow — it removes the `-SNAPSHOT` suffix, deploys the release artifact, tags the commit, and prepares the next development version. Use this when you're ready to ship a stable version.

### What it does, step by step

1. Checks out your code (with full git history).
2. Sets up Java.
3. **Determines the release version.** If you provide one, it uses that. Otherwise, it strips `-SNAPSHOT` from the current version (e.g., `1.2.0-SNAPSHOT` becomes `1.2.0`).
4. **Determines the next SNAPSHOT version.** If you provide one, it uses that. Otherwise, it increments the patch number (e.g., `1.2.0` becomes `1.2.1-SNAPSHOT`).
5. Updates `pom.xml` to the release version and commits: `release: v1.2.0`.
6. Runs `mvn clean deploy -DskipTests` to publish the release artifact.
7. Creates a Git tag: `v1.2.0`.
8. Updates `pom.xml` to the next SNAPSHOT version and commits: `chore: bump version to 1.2.1-SNAPSHOT`.
9. Pushes both commits and the tag to the remote.

After it succeeds, your repository will have a new tag, the release artifact will be on GitHub Packages, and your branch will already be on the next development version.

### How to use it

Create a file `.github/workflows/release.yml` in your repository:

```yaml
name: Release

on:
  workflow_dispatch:

jobs:
  release:
    uses: luizroddev/rest-build-parent/.github/workflows/ci-release.yml@master
    secrets:
      PACKAGES_TOKEN: ${{ secrets.PACKAGES_TOKEN }}
```

Using `workflow_dispatch` means you trigger it manually from the GitHub Actions tab — click **"Run workflow"** when you're ready to release.

### Optional inputs

| Input | Default | Description |
|---|---|---|
| `release-version` | _(auto: strips `-SNAPSHOT`)_ | Exact version to release (e.g., `2.0.0`) |
| `next-snapshot-version` | _(auto: increments patch)_ | Next development version (e.g., `2.0.1-SNAPSHOT`) |
| `java-version` | `22` | Java version |
| `java-distribution` | `temurin` | JDK distribution |
| `maven-args` | _(empty)_ | Extra Maven arguments |

Example with explicit versions:

```yaml
jobs:
  release:
    uses: luizroddev/rest-build-parent/.github/workflows/ci-release.yml@master
    with:
      release-version: '2.0.0'
      next-snapshot-version: '2.1.0-SNAPSHOT'
    secrets:
      PACKAGES_TOKEN: ${{ secrets.PACKAGES_TOKEN }}
```

### Important

- Released versions **cannot be overwritten** on GitHub Packages. If the deploy fails midway, you may need to bump the version before retrying.
- The workflow pushes commits and tags directly to your branch. Make sure the `PACKAGES_TOKEN` has `repo` scope.
- The Git tag format is always `vX.Y.Z` (e.g., `v1.2.0`).

---

## Quick Reference

| I want to... | Workflow | Trigger |
|---|---|---|
| Check if my code compiles and passes tests | `ci-build.yml` | Push / Pull Request |
| Publish a development snapshot for other modules | `ci-deploy-snapshot.yml` | Push to `master` |
| Ship a stable release with tagging and version bump | `ci-release.yml` | Manual (`workflow_dispatch`) |

---

## Full Example

A typical repository uses all three workflows together. Here's what the complete setup looks like:

**.github/workflows/build.yml**
```yaml
name: Build
on:
  push:
    branches: [master]
  pull_request:
    branches: [master]

jobs:
  build:
    uses: luizroddev/rest-build-parent/.github/workflows/ci-build.yml@master
    secrets:
      PACKAGES_TOKEN: ${{ secrets.PACKAGES_TOKEN }}
```

**.github/workflows/deploy-snapshot.yml**
```yaml
name: Deploy Snapshot
on:
  push:
    branches: [master]

jobs:
  deploy:
    needs: build  # only deploy if build passes
    uses: luizroddev/rest-build-parent/.github/workflows/ci-deploy-snapshot.yml@master
    secrets:
      PACKAGES_TOKEN: ${{ secrets.PACKAGES_TOKEN }}

  build:
    uses: luizroddev/rest-build-parent/.github/workflows/ci-build.yml@master
    secrets:
      PACKAGES_TOKEN: ${{ secrets.PACKAGES_TOKEN }}
```

**.github/workflows/release.yml**
```yaml
name: Release
on:
  workflow_dispatch:

jobs:
  release:
    uses: luizroddev/rest-build-parent/.github/workflows/ci-release.yml@master
    secrets:
      PACKAGES_TOKEN: ${{ secrets.PACKAGES_TOKEN }}
```

With this setup:
- Every push and PR runs the **build** to catch errors early.
- Every push to `master` automatically **deploys a snapshot** (after the build passes).
- Releases are triggered **manually** whenever you decide a version is ready.
