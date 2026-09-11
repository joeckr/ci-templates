# ci-templates

Repository for storing reusable CI templates and GitHub Actions workflows.

---

## Workflows

### `push-helm-ghcr.yml`
Reusable workflow to lint, package, and push Helm charts to GitHub Container Registry (GHCR) as OCI artifacts.

#### Features
- **Helm OCI Packaging**: Packages charts and pushes them directly to GHCR via native OCI registry support (`oci://ghcr.io/...`).
- **GHCR Owner Lowercasing**: Automatically converts username/org namespace to lowercase to prevent GHCR rejected name errors.
- **Smart Chart Discovery**: Automatically scans a directory (default `chart`) for charts, or targets a specific chart via `chart-path`.
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
| `subpath` | Optional subpath under registry namespace | No | `''` |
| `helm-version` | Helm version to install | No | `'latest'` |
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

### `build-oci-modified.yml`
Reusable matrix container build workflow for building and pushing multi-platform container images based on upstream versions.

#### Features
- **Dynamic Matrix from JSON**: Automatically parses a `versions.json` configuration file into a GitHub Actions build matrix.
- **Multi-Platform Support**: Sets up QEMU and Docker Buildx to build for multiple architectures (default: `linux/amd64,linux/arm64`).
- **Flexible Tagging**: Automatically tags images using major version (`<image>:<major>`), upstream version (`<image>:<upstream>`), and optionally `:latest` and `:lts` flags.
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

## Local Development & Conventional Commits (`mise` & `prek`)

This repository uses [`mise`](https://mise.jdx.dev/) for environment and tool management, and [`prek`](https://github.com/j178/prek) (a fast Rust-based git hook runner configured via `prek.toml`) to enforce [Conventional Commits](https://www.conventionalcommits.org/) pre-commit.

### Setup

1. **Install tools using mise**:
   ```bash
   mise install
   ```

2. **Install git hooks**:
   ```bash
   mise run hooks:install
   # or directly:
   prek install
   ```

3. **Run hooks manually on all files**:
   ```bash
   mise run hooks:run
   # or directly:
   prek run --all-files
   ```

### Configured Hooks (`prek.toml`)
- **Conventional Commits**: Validates commit messages via `compilerla/conventional-pre-commit` on `commit-msg`.
- **Branch Protection**: Prevents direct commits to `main` via `no-commit-to-branch` (enforcing feature branches and pull requests).
- **Secret Detection**: Scans for leaked credentials via `gitleaks` on `pre-commit`.
- **YAML Validation**: Validates YAML syntax across workflows and charts via `check-yaml` (`--allow-multiple-documents`).
- **Code Hygiene**: Trims trailing whitespace and ensures clean file endings.

### Conventional Commits Format
Commits must follow the Conventional Commits specification:
- `feat: add new feature` -> triggers **minor** release
- `fix: resolve issue` -> triggers **patch** release
- `feat!: breaking redesign` or footer `BREAKING CHANGE:` -> triggers **major** release
- `chore:`, `docs:`, `ci:`, `test:`, `refactor:` -> maintenance changes

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
    uses: <org-name>/<repo-name>/.github/workflows/gitleaks.yml@main
```
