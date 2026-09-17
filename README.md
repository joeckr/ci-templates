# ci-templates

Repository for storing reusable CI templates and GitHub Actions workflows.

---

## Workflows

### `actionlint.yml`
Reusable workflow to run Actionlint for linting GitHub Actions workflows.

#### Features
- **Workflow Linting**: Uses `actionlint` to static check GitHub Actions workflows.
- **Fast Execution**: Downloads the official actionlint binary to run locally in the runner.

#### Example Usage

```yaml
name: Run Actionlint

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  actionlint:
    uses: joeckr/ci-templates/.github/workflows/actionlint.yml@main
```

---

### `betterleaks.yml`
Reusable workflow to run Betterleaks for secret detection.

#### Features
- **Config Detection**: Automatically checks for the presence of a `.betterleaks.toml` file in the root of the repository. If found, it uses the provided configuration; otherwise, it runs a full scan with default settings.
- **Secret Scanning**: Downloads the latest Betterleaks binary to explicitly execute full repository scans to detect hardcoded secrets, passwords, and API keys.

#### Example Usage

```yaml
name: Betterleaks Scan

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  betterleaks:
    uses: joeckr/ci-templates/.github/workflows/betterleaks.yml@main
```

---

### `build-oci-modified.yml`
Reusable matrix container build workflow for building and pushing multi-platform container images based on upstream versions.

#### Features
- **Dynamic Matrix from JSON**: Automatically parses a `versions.json` configuration file into a GitHub Actions build matrix.
- **Multi-Platform Support**: Sets up QEMU and Docker Buildx to build for multiple architectures (default: `linux/amd64,linux/arm64`).
- **Flexible Tagging**: Automatically tags images using major version (`<image>:<major>`), upstream version (`<image>:<upstream>`), and optionally `:latest` and `:lts` flags.
- **Vulnerability Scanning (Trivy)**: Non-blocking security scanning on pull requests and pushes, surfacing findings in the GitHub Security tab via SARIF and in the Actions Job Summary table.
- **SBOM Generation**: Automatically produces Software Bill of Materials (CycloneDX JSON or SPDX JSON) and uploads them as workflow artifacts per matrix version.
- **Buildx Caching**: Leverages GitHub Actions cache (`type=gha`) for fast incremental builds.
- **Dry-Run & PR Safety**: Skips image push on pull requests or when `dry-run: true`.

#### Example `versions.json`
```json
[
  {
    "major": "1",
    "upstream": "1.28.3",
    "latest": true,
    "lts": false
  },
  {
    "major": "2",
    "upstream": "2.1.0",
    "latest": false,
    "lts": true
  }
]
```

#### Example Usage

```yaml
name: Build and Push Container Images

on:
  push:
    branches: [ main ]
    tags: [ 'v*' ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    permissions:
      contents: read
      packages: write
      security-events: write
    uses: joeckr/ci-templates/.github/workflows/build-oci-modified.yml@main
    with:
      image: 'my-app'
      # Optional configurations:
      # versions-file: 'versions.json'
      # platforms: 'linux/amd64,linux/arm64'
      # dockerfile: 'Dockerfile'
      # context: '.'
      # registry: 'ghcr.io'
      # dry-run: false
      # enable-trivy: true
      # enable-sbom: true
      # trivy-severity: 'CRITICAL,HIGH'
      # trivy-ignore-unfixed: false
      # sbom-format: 'cyclonedx'
```

#### Inputs
| Input | Description | Required | Default |
| --- | --- | --- | --- |
| `image` | Name of the image to build | **Yes** | — |
| `versions-file` | Path to `versions.json` relative to repo root | No | `'versions.json'` |
| `org` | Name of the org or individual user | No | `${{ github.actor }}` |
| `context` | Build context path | No | `'.'` |
| `dockerfile` | Path to Dockerfile relative to context | No | `'Dockerfile'` |
| `platforms` | Target container platforms | No | `'linux/amd64,linux/arm64'` |
| `registry` | Container registry to push to | No | `'ghcr.io'` |
| `dry-run` | Build images on PR or test without pushing | No | `false` |
| `enable-trivy` | Run Trivy vulnerability scanner | No | `true` |
| `enable-sbom` | Generate and upload an SBOM artifact | No | `true` |
| `trivy-severity` | Vulnerability severities to scan for (comma-separated) | No | `'CRITICAL,HIGH'` |
| `trivy-ignore-unfixed` | Ignore vulnerabilities without an available fix | No | `false` |
| `sbom-format` | SBOM output format (`cyclonedx` or `spdx-json`) | No | `'cyclonedx'` |

---

### `commitlint.yml`
Reusable workflow to run Commitlint for linting conventional commit messages.

#### Features
- **Commit Linting**: Uses `wagoid/commitlint-github-action` to lint commit messages.
- **Dynamic Fallback Config**: Generates a fallback configuration to disable standard line-length limits for the header, body, and footer, accommodating detailed, longer commit messages without requiring downstream repos to maintain their own configuration.

#### Example Usage

```yaml
name: Run Commitlint

on:
  push:
    branches: [ main ]
  pull_request:

jobs:
  commitlint:
    uses: joeckr/ci-templates/.github/workflows/commitlint.yml@main
```

---

### `gitleaks.yml`
Reusable workflow to run Gitleaks for secret detection.

#### Features
- **Config Detection**: Automatically checks for the presence of a `gitleaks.toml` file in the root of the repository. If found, it uses the provided configuration; otherwise, it runs a full scan with default settings.
- **Secret Scanning**: Downloads the latest Gitleaks binary to explicitly execute full repository scans to detect hardcoded secrets, passwords, and API keys.

#### Example Usage

```yaml
name: Gitleaks Scan

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  gitleaks:
    uses: joeckr/ci-templates/.github/workflows/gitleaks.yml@main
```

---

### `hadolint.yml`
Reusable workflow to run Hadolint for linting Dockerfiles.

#### Features
- **Dockerfile Linting**: Uses `hadolint/hadolint-action` to lint Dockerfiles and enforce best practices.
- **Configurable**: Supports custom configuration files and severity thresholds.
- **Recursive Scanning**: Optionally scan all Dockerfiles in a repository recursively.

#### Example Usage

```yaml
name: Run Hadolint

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  hadolint:
    uses: joeckr/ci-templates/.github/workflows/hadolint.yml@main
    with:
      dockerfile: 'Dockerfile'
      # Optional configurations:
      # recursive: false
      # failure-threshold: 'info'
      # config: '.hadolint.yaml'
```

#### Inputs
| Input | Description | Required | Default |
| --- | --- | --- | --- |
| `dockerfile` | Path to the Dockerfile to lint | No | `'Dockerfile'` |
| `recursive` | Lint all Dockerfiles in the repository recursively | No | `false` |
| `failure-threshold` | Fail the pipeline when issues of this severity or higher are found (error, warning, info, style) | No | `'info'` |
| `config` | Path to a custom hadolint config file | No | `''` |

---

### `push-helm-ghcr.yml`
Reusable workflow to lint, package, and push Helm charts to GitHub Container Registry (GHCR) as OCI artifacts.

#### Features
- **Helm OCI Packaging**: Packages charts and pushes them directly to GHCR via native OCI registry support (`oci://ghcr.io/...`).
- **GHCR Owner Lowercasing**: Automatically converts username/org namespace to lowercase to prevent GHCR rejected name errors.
- **Smart Chart Discovery**: Automatically scans a directory (default `chart`) for charts, or targets a specific chart via `chart-path`.
- **External Helm Repositories**: Supports adding external Helm repositories (e.g. Longhorn, Bitnami) via `helm-repos` prior to dependency resolution.
- **Conditional Dependency Build**: Checks for `Chart.lock` or `dependencies:` in `Chart.yaml` before running `helm dependency build`.
- **Linting & Safety**: Runs `helm lint` by default; automatically operates in dry-run mode on pull requests or when `dry-run: true`.
- **Step Summary**: Emits a markdown table into the GitHub Actions run summary detailing packaged charts and destination OCI URLs.

#### Example Usage

```yaml
name: Publish Helm Chart

on:
  push:
    branches: [ main ]
    tags: [ 'v*' ]
  pull_request:
    branches: [ main ]

jobs:
  helm:
    permissions:
      contents: read
      packages: write
    uses: joeckr/ci-templates/.github/workflows/push-helm-ghcr.yml@main
    with:
      charts-dir: 'chart'
      # Optional: specify a single chart instead of scanning
      # chart-path: 'chart/my-app'
      # Optional: add external Helm repositories for chart dependencies
      # helm-repos: |
      #   longhorn https://charts.longhorn.io
      # Optional: test without pushing
      # dry-run: false
    secrets:
      github-token: ${{ secrets.GITHUB_TOKEN }}
```

#### Inputs
| Input | Description | Required | Default |
| --- | --- | --- | --- |
| `chart-path` | Path to a single chart directory (e.g. `chart/my-chart`). If provided, `charts-dir` is ignored | No | `''` |
| `charts-dir` | Directory containing Helm charts relative to repo root | No | `'chart'` |
| `registry` | Container registry to push to | No | `'ghcr.io'` |
| `org` | Organization or user owning the registry namespace | No | `${{ github.repository_owner }}` |
| `subpath` | Optional subpath under registry namespace | No | `'charts'` |
| `helm-version` | Helm version to install | No | `'latest'` |
| `helm-repos` | Optional newline-separated list of Helm repositories to add before dependency build | No | `''` |
| `version` | Optional version override for chart packaging | No | `''` |
| `app-version` | Optional appVersion override for chart packaging | No | `''` |
| `dependency-update` | Run `helm dependency build` prior to packaging | No | `true` |
| `lint` | Run `helm lint` prior to packaging | No | `true` |
| `dry-run` | Package and lint charts without pushing to registry | No | `false` |

#### Secrets
| Secret | Description | Required | Default |
| --- | --- | --- | --- |
| `github-token` | Token for authenticating with the registry | No | `secrets.GITHUB_TOKEN` |

---

### `semantic.yml`
Reusable workflow for automated Semantic Versioning (SemVer), Conventional Commits analysis, Helm chart metadata synchronization, and GitHub Releases.

#### Features
- **Conventional Commits Analysis**: Automatically determines version bumps (`major`, `minor`, `patch`) from commit history since the last git tag.
- **Helm Chart Metadata Synchronization**: Automatically updates `version` and `appVersion` in `Chart.yaml` using `yq` and commits the changes back to the repository (`[skip ci]`).
- **Automated Tagging & Releases**: Tags the release commit and publishes a GitHub Release with auto-generated categorized release notes.
- **Composable Workflow Outputs**: Emits `version`, `tag`, `bump`, and `released` so downstream workflows (e.g. `push-helm-ghcr.yml`) can immediately consume the new version.
- **Dry-Run & PR Preview**: Operates safely in dry-run mode on pull requests or when `dry-run: true` is provided.

#### Example Usage

##### 1. Standalone Repository Release
```yaml
name: Release

on:
  push:
    branches: [ main ]

jobs:
  release:
    permissions:
      contents: write
    uses: joeckr/ci-templates/.github/workflows/semantic.yml@main
    secrets:
      github-token: ${{ secrets.GITHUB_TOKEN }}
```

##### 2. Chained with Helm Chart Publishing (`push-helm-ghcr.yml`)
```yaml
name: Release & Publish Helm Chart

on:
  push:
    branches: [ main ]

jobs:
  semver:
    permissions:
      contents: write
    uses: joeckr/ci-templates/.github/workflows/semantic.yml@main
    with:
      chart-path: 'chart/my-app' # Automatically bumps version & appVersion in Chart.yaml
    secrets:
      github-token: ${{ secrets.GITHUB_TOKEN }}

  publish-helm:
    needs: semver
    if: needs.semver.outputs.released == 'true'
    permissions:
      contents: read
      packages: write
    uses: joeckr/ci-templates/.github/workflows/push-helm-ghcr.yml@main
    with:
      chart-path: 'chart/my-app'
      version: ${{ needs.semver.outputs.version }}
      app-version: ${{ needs.semver.outputs.version }}
```

#### Inputs
| Input | Description | Required | Default |
| --- | --- | --- | --- |
| `chart-path` | Path to Helm chart directory (e.g. `chart` or `chart/my-app`). If provided, `Chart.yaml` is updated | No | `'chart'` |
| `update-chart-version` | Update the `version` field in `Chart.yaml` | No | `true` |
| `update-chart-app-version` | Update the `appVersion` field in `Chart.yaml` | No | `true` |
| `tag-prefix` | Tag prefix for Git tags (e.g. `v`) | No | `'v'` |
| `initial-version` | Fallback version if no Git tags exist | No | `'0.1.0'` |
| `create-release` | Create a GitHub Release with auto-generated notes | No | `true` |
| `create-tag` | Create and push Git tag | No | `true` |
| `prerelease` | Mark release as prerelease | No | `false` |
| `draft` | Create release as draft | No | `false` |
| `dry-run` | Calculate version and preview without pushing changes | No | `false` |

#### Secrets
| Secret | Description | Required | Default |
| --- | --- | --- | --- |
| `github-token` | Token with `contents: write` permission | No | `secrets.GITHUB_TOKEN` |

#### Outputs
| Output | Description | Example |
| --- | --- | --- |
| `version` | Calculated SemVer version without prefix | `1.2.0` |
| `tag` | Git tag with prefix | `v1.2.0` |
| `previous-version` | Previous version prior to bump | `1.1.1` |
| `previous-tag` | Previous Git tag | `v1.1.1` |
| `bump` | Bump type (`major`, `minor`, `patch`, `none`) | `minor` |
| `released` | Whether a release was published (`true` / `false`) | `true` |

---

### `shellcheck.yml`
Reusable workflow to run ShellCheck for shell script static analysis.

#### Features
- **Shell Script Linting**: Uses `ludeeus/action-shellcheck` to analyze shell scripts and identify syntax issues, semantic problems, and common pitfalls.
- **Configurable Scanning**: Customize the directory to scan, severity threshold, and files to ignore.

#### Example Usage

```yaml
name: Run ShellCheck

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  shellcheck:
    permissions:
      contents: read
    uses: joeckr/ci-templates/.github/workflows/shellcheck.yml@main
    # with:
    #   scandir: '.'
    #   severity: 'style'
```

#### Inputs
| Input | Description | Required | Default |
| --- | --- | --- | --- |
| `scandir` | Directory to be searched for files | No | `'.'` |
| `format` | Output format (checkstyle, diff, gcc, json, json1, quiet, tty) | No | `'gcc'` |
| `severity` | Minimum severity of errors to consider (error, warning, info, style) | No | `''` |
| `check_together` | Run shellcheck on all files at once | No | `''` |
| `version` | Specify a concrete version of ShellCheck to use | No | `'stable'` |
| `additional_files` | A space separated list of additional filename to check | No | `''` |
| `ignore_paths` | Paths to ignore when running ShellCheck | No | `''` |
| `ignore_names` | Names to ignore when running ShellCheck | No | `''` |

---

### `zizmor.yml`
Reusable workflow to run Zizmor for workflow security linting.

#### Features
- **Security Linting**: Uses `zizmor` to audit GitHub Actions workflows for security vulnerabilities.
- **GitHub Advanced Security Integration**: Uploads SARIF results to surface findings in the GitHub Security tab.

#### Example Usage

```yaml
name: Run Zizmor

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  zizmor:
    permissions:
      contents: read
      security-events: write
    uses: joeckr/ci-templates/.github/workflows/zizmor.yml@main
```

---

## Local Development & Conventional Commits (`mise` & `hk`)

This repository uses [`mise`](https://mise.jdx.dev/) for environment and tool management, and [`hk`](https://hk.jdx.dev/) (a fast git hook runner configured via `hk.pkl`) to enforce [Conventional Commits](https://www.conventionalcommits.org/) pre-commit.

### Setup

1. **Install tools using mise**:
   ```bash
   mise install
   ```

2. **Install git hooks**:
   ```bash
   mise run hooks:install
   # or directly:
   hk install
   ```

3. **Run hooks manually on all files**:
   ```bash
   mise run hooks:run
   # or directly:
   hk run --all
   ```

### Configured Hooks (`hk.pkl`)
- **Conventional Commits**: Validates commit messages via `compilerla/conventional-pre-commit` on `commit-msg`.
- **Branch Protection**: Prevents direct commits to `main` via `no-commit-to-branch` (enforcing feature branches and pull requests).
- **Secret Detection**: Scans for leaked credentials via `gitleaks` on `pre-commit`.
- **YAML Validation**: Validates YAML syntax across workflows and charts via `check-yaml` (`--allow-multiple-documents`).
- **Workflow Linting**: Lints GitHub Actions workflows for syntax and semantics via `actionlint`.
- **Security Linting**: Audits GitHub Actions workflows for security vulnerabilities via `zizmor`.
- **Code Hygiene**: Trims trailing whitespace and ensures clean file endings.

### Conventional Commits Format
Commits must follow the Conventional Commits specification:
- `feat: add new feature` -> triggers **minor** release
- `fix: resolve issue` -> triggers **patch** release
- `feat!: breaking redesign` or footer `BREAKING CHANGE:` -> triggers **major** release
- `chore:`, `docs:`, `ci:`, `test:`, `refactor:` -> maintenance changes
