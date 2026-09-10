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

