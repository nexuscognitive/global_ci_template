# build-push-acr

Reusable workflow that builds container images from a release tag, squashes them, pushes them
to Azure Container Registry (ACR) and scans them with Trivy.

File: `.github/workflows/build-push-acr.yml`

## How it fits together

```
merge PR to main
  -> release-please opens/updates a release PR
  -> you merge the release PR
  -> release-please creates tag vX.Y.Z (using a GitHub App token)
  -> the tag push triggers your caller workflow
  -> build-push-acr.yml
       1. resolve-version   validates the tag, strips the "v" (v3.4.0 -> 3.4.0)
       2. build-and-push    one job per entry in `images`
```

Release images are only ever built from a strict `vX.Y.Z` tag. Every other ref (branch, PR,
manual dispatch, `v1.2.3-rc1`, `v1.2`) fails the `resolve-version` guard and builds nothing.

## Adopting it

### 1. Prerequisites

- The ACR secrets `ACR_REGISTRY`, `ACR_USERNAME` and `ACR_PASSWORD` are visible to your repo
  (org secrets, with your repo in their access list).
- Your repo is allowed to use this template. If `global_ci_template` is private, enable
  Settings -> Actions -> General -> Access -> "Accessible from repositories in the organization".
- Tags are created by **release-please using a GitHub App or PAT token, not `GITHUB_TOKEN`**.
  GitHub does not start workflows from events created with `GITHUB_TOKEN`, so the tag would
  exist but nothing would build. Do not push release tags by hand.

### 2. Example caller workflow

`.github/workflows/build-acr.yml` in your repo:

```yaml
name: Build ACR

permissions:
  contents: read

on:
  push:
    tags:
      # Glob pre-filter only. Strict vX.Y.Z validation happens in resolve-version.
      - 'v[0-9]*.[0-9]*.[0-9]*'

jobs:
  build:
    uses: nexuscognitive/global_ci_template/.github/workflows/build-push-acr.yml@main
    with:
      images: >-
        [
          {"name": "my-app", "dockerfile": "Dockerfile", "context": "."}
        ]
    secrets:
      ACR_REGISTRY: ${{ secrets.ACR_REGISTRY }}
      ACR_USERNAME: ${{ secrets.ACR_USERNAME }}
      ACR_PASSWORD: ${{ secrets.ACR_PASSWORD }}
      # Optional, only if your Dockerfile uses --mount=type=secret,id=GH_PAT:
      # GH_PAT: ${{ secrets.GH_PAT }}
```

Pin `@main` to a tag or commit SHA if you want changes to the template to be opt-in.

The tag `v3.4.0` produces `<ACR_REGISTRY>/my-app:3.4.0`.

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `images` | yes | | JSON array of images to build. See below. |
| `lfs` | no | `false` | Check out Git LFS files. |
| `timeout_minutes` | no | `60` | Per-image job timeout. Raise for very large images, since the squash cost tracks image size. |
| `push_image` | no | `true` | Push to ACR. Set `false` for dry runs: build, squash and scan only. |

### `images` entries

| Key | Required | Description |
|---|---|---|
| `name` | yes | Image repository name in ACR. |
| `dockerfile` | yes | Path to the Dockerfile. |
| `context` | yes | Docker build context. |
| `build_args` | no | List of `"KEY=VALUE"` strings, each passed as `--build-arg`. |
| `variant` | no | Suffix for the container tag: `<version>-<variant>` (e.g. `3.4.0-spark3`). |

Multiple entries build in parallel as a matrix, all from the same version:

```yaml
images: >-
  [
    {"name": "my-app", "dockerfile": "Dockerfile", "context": "."},
    {"name": "my-app", "dockerfile": "Dockerfile.spark", "context": ".",
     "variant": "spark3", "build_args": ["SPARK_VERSION=3.5.1"]}
  ]
```

This produces `my-app:3.4.0` and `my-app:3.4.0-spark3` for tag `v3.4.0`. The variant never
appears in the git tag; a repo has one version and its images may have several tags.

## Secrets

| Secret | Required | Purpose |
|---|---|---|
| `ACR_REGISTRY` | yes | Registry host, used for login and as the image prefix. |
| `ACR_USERNAME` | yes | Registry login. |
| `ACR_PASSWORD` | yes | Registry login. |
| `GH_PAT` | no | Passed to the build as BuildKit secret `GH_PAT`. |
| `TRINO_HOST`, `TRINO_USER`, `TRINO_PASSWORD`, `TRINO_CATALOG`, `TRINO_SCHEMA`, `TRINO_TABLE` | no | Enable uploading Trivy results to Trino. If any is missing the upload is skipped with a warning. |
| `NPM_TOKEN` | no | Declared for callers that need it. |

Avoid `secrets: inherit` unless you intend the Trino upload to run: it forwards the org
`TRINO_*` secrets and scan results will be written to the real table. Pass secrets explicitly
for tests.

## Outputs

| Output | Description |
|---|---|
| `release` | `"true"` when the run came from a `vX.Y.Z` tag. |
| `version` | Version without the leading `v` (`3.4.0`). |
| `container_tag` | Base tag used for every image, before any `variant` suffix. |

Use `container_tag` if the caller builds extra images outside this template, rather than
re-deriving a tag from `GITHUB_REF`:

```yaml
jobs:
  build:
    uses: nexuscognitive/global_ci_template/.github/workflows/build-push-acr.yml@main
    # ...
  follow-up:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: echo "Built ${{ needs.build.outputs.container_tag }}"
```

## What each image job does

1. Frees runner disk space (removes dotnet, Android, ghc, tool cache; prunes Docker).
2. Checks out the repo.
3. Computes the tag `<container_tag>[-<variant>]`.
4. Logs in to ACR.
5. Builds the image.
6. Squashes it with `docker-squash`.
7. Pushes it (skipped when `push_image: false`).
8. Scans it with Trivy: full JSON report, plus a HIGH/CRITICAL table in the job log.
9. Uploads the findings to Trino, if configured.
10. Runs the HIGH/CRITICAL report step.
11. Logs out of ACR.

## Things to know

- **Tag format:** images are tagged `1.2.3`, not `v1.2.3`.
- **Scan results are not kept by GitHub.** The Trivy JSON stays on the runner and is not
  uploaded as an artifact. The job log and the Trino table are the records.
- **The severity gate is informational.** It lists HIGH/CRITICAL findings but does not fail
  the build.
- **The image is pushed before it is scanned.**

## Design notes

- **Why a regex guard instead of only a tag filter in the caller.** GitHub tag filters are
  globs, not regexes, and cannot express "digits only, exactly three groups, nothing after".
  The guard in `resolve-version` can, and every tenant inherits it from this one place. The
  caller's glob is just a pre-filter.
- **Why pre-release tags are rejected.** Semver sorts `v1.2.3-rc1` below `v1.2.3`, so tools that
  use a semver version filter (such as updatecli) would treat it as older than what tenants
  already run. Non-release builds (`sha-<short>`, `pr-<n>`) are not supported yet.
- **Single source of the tag.** `resolve-version` is the only place the container tag is
  decided, and the build steps never read `GITHUB_REF`.
- **Disk reclaim before the scan.** Trivy exports the whole uncompressed image. Large images
  (for example a Spark distribution) filled the runner's root disk and failed the scan, which
  also skips the severity gate. The workflow prunes the build cache and gives Trivy its temp
  space on `/mnt`, the runner's large volume.

## Troubleshooting

| Symptom | Cause |
|---|---|
| Caller never starts after the release PR merges | The tag was created with `GITHUB_TOKEN`. Use a GitHub App or PAT token in release-please. |
| `Refusing to build` error in `resolve-version` | The ref is not a `vX.Y.Z` tag. Pre-release suffixes and non-tag triggers are rejected on purpose. |
| Workflow not found / secrets empty | The template repo or org secrets are not shared with your repo. See Prerequisites. |
| `no space left on device` | Image too large for the runner. Keep the disk-space step and raise `timeout_minutes` if the squash is slow. |
| Trino upload skipped warnings | The `TRINO_*` secrets were not passed. Expected for tests. |
