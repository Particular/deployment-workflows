# deployment-workflows

Shared workflows and actions for Particular's release deployment process.

- [Usage](#usage)
- Actions: [create-draft-release-with-assets](#create-draft-release-with-assets) · [push-nuget-packages](#push-nuget-packages) · [determine-image-tags](#determine-image-tags)
- Workflows: [verify-authorized-user](#verify-authorized-user) · [generate-release-notes](#generate-release-notes) · [publish-release-notes](#publish-release-notes) · [push-powershell-packages](#push-powershell-packages) · [push-release-info-to-aws-s3](#push-release-info-to-aws-s3) · [push-docker-images](#push-docker-images)

## Usage

```yaml
# Action - a step inside your job
- uses: Particular/deployment-workflows/.github/actions/<name>@<version>

# Workflow - a job in your workflow
jobs:
  my-job:
    uses: Particular/deployment-workflows/.github/workflows/<name>.yml@<version>
    secrets: inherit # workflows declare no secrets, so inherit is required when they need any
```

- Every action/workflow with a `dry-run` input logs what it *would* do instead of pushing/publishing.
- Scripts run in `pwsh`.
- A release's `version` is its tag (e.g. `1.2.3`), unless noted.



## Actions

### create-draft-release-with-assets

Creates a draft GitHub release for the current `github.ref_name` and attaches every file found under the given paths.

**Assumes:** artifacts are already built and sitting in the given paths. Job has `contents: write`.

```yaml
# Basic
- uses: Particular/deployment-workflows/.github/actions/create-draft-release-with-assets@<version>
  with:
    asset-paths: nugets/
```

```yaml
# Full
- uses: Particular/deployment-workflows/.github/actions/create-draft-release-with-assets@<version>
  id: draft
  with:
    asset-paths: nugets/, powershellModules/, assets/*.zip # comma-separated paths/globs, searched recursively; default nugets,assets
    dry-run: false                                         # default false
- run: echo "${{ steps.draft.outputs.assets }}"            # comma-separated list of attached files
```

### push-nuget-packages

Downloads the `.nupkg` assets from a release and pushes them to the NuGet feed for the target environment (duplicates are skipped).

| `deploy-environment` | Assets come from | Pushed to | Auth |
|---|---|---|---|
| `staging` | the **draft** release whose *name* is `version` | Feedz (`particular-software/packages`) | `api-key` |
| `production` | the **published** release tagged `version` | nuget.org | NuGet trusted publishing (OIDC) |

**Assumes:**
- The release already has `.nupkg` assets.
- `api-key` is always required (only used for `staging`).
- `production`: job has `id-token: write` and `contents: read`.

```yaml
# Basic
- uses: Particular/deployment-workflows/.github/actions/push-nuget-packages@<version>
  with:
    version: ${{ github.ref_name }}
    deploy-environment: production
    api-key: ${{ secrets.FEEDZ_API_KEY }}
```

```yaml
# Full
- uses: Particular/deployment-workflows/.github/actions/push-nuget-packages@<version>
  with:
    version: 1.2.3                       # release tag (production) or release *name* (staging)
    deploy-environment: staging          # staging | production
    api-key: ${{ secrets.FEEDZ_API_KEY }} # required; used for the staging push
    exclude-package-pattern: '*.Tests.*' # default: none. Glob of package filenames to skip
    dry-run: false                       # default false
```

### determine-image-tags

Works out which Docker tags a version should get (`1.2.3`, `1.2`, `1`, `latest`) by comparing it with the repo's published releases.

**Assumes:** `version` is `X.Y.Z` or `X.Y.Z-prerelease`. Prereleases only get their exact tag. Floating tags only move forward (a backport won't take `latest` or the major tag).

```yaml
- uses: Particular/deployment-workflows/.github/actions/determine-image-tags@<version>
  id: tags
  with:
    version: 1.2.3
- run: |
    echo "${{ steps.tags.outputs.image-tags }}"  # comma-separated, e.g. 1.2.3,latest,1,1.2
    echo "${{ steps.tags.outputs.latest }}"      # true/false
    echo "${{ steps.tags.outputs.major }} ${{ steps.tags.outputs.minor }} ${{ steps.tags.outputs.patch }}"
    echo "${{ steps.tags.outputs.prerelease }}"  # label, empty when stable
```

## Workflows

### verify-authorized-user

Fails unless the triggering user (`github.actor`) is listed in the `AUTHORIZED_RELEASE_USERS` variable. Use as a gate with `needs:`.

**Assumes:** repo/org variable `AUTHORIZED_RELEASE_USERS` is a comma-separated list of GitHub usernames.

```yaml
jobs:
  authorize:
    uses: Particular/deployment-workflows/.github/workflows/verify-authorized-user.yml@<version>
  release:
    needs: authorize
    # ...
```

The `actor` and `authorized-users` inputs are test-only overrides and fail outside this repository.

### generate-release-notes

Fills the draft release's notes from the closed issues of the milestone named after the version (via GitReleaseManager).

**Assumes:**
- A milestone named exactly `version` exists.
- Repo has `global.json`, `.release/GitReleaseManager.yaml` and `.release/.templates/default/issue-note.sbn`.
- Caller grants `contents: write`.

```yaml
# Basic
generate-notes:
  uses: Particular/deployment-workflows/.github/workflows/generate-release-notes.yml@<version>
  with:
    version: 1.2.3
```

```yaml
# Full
generate-notes:
  uses: Particular/deployment-workflows/.github/workflows/generate-release-notes.yml@<version>
  with:
    version: 1.2.3 # milestone and release name
    dry-run: false # default false
```

### publish-release-notes

Publishes the draft release for `version` (via GitReleaseManager).

**Assumes:** the draft release exists. Caller grants `contents: write`.

```yaml
# Basic
publish:
  uses: Particular/deployment-workflows/.github/workflows/publish-release-notes.yml@<version>
  with:
    version: 1.2.3
```

```yaml
# Full
publish:
  uses: Particular/deployment-workflows/.github/workflows/publish-release-notes.yml@<version>
  with:
    version: 1.2.3 # release tag
    dry-run: false # default false
```

### push-powershell-packages

Downloads matching release assets and publishes them to the PowerShell gallery for the target environment.

- `staging`: assets come from the **draft** release named `version`; pushed to the Feedz gallery.
- `production`: assets come from the **published** release tagged `version`; pushed to the PowerShell Gallery.

**Assumes:** the release already has the assets; the gallery variables/secrets below are available to the caller. The gallery is registered if missing.

| | `staging` | `production` |
|---|---|---|
| vars | `FEEDZ_GALLERY_NAME`, `FEEDZ_GALLERY_SOURCE_LOCATION` | `POWERSHELL_GALLERY_NAME`, `POWERSHELL_GALLERY_SOURCE_LOCATION` |
| secret | `FEEDZ_GALLERY_KEY` | `POWERSHELL_GALLERY_KEY` |

```yaml
# Basic
push-ps:
  uses: Particular/deployment-workflows/.github/workflows/push-powershell-packages.yml@<version>
  with:
    version: 1.2.3
    deploy-environment: production
    package-pattern: '*.nupkg'
  secrets: inherit
```

```yaml
# Full
push-ps:
  uses: Particular/deployment-workflows/.github/workflows/push-powershell-packages.yml@<version>
  with:
    version: 1.2.3                # release tag (production) or release *name* (staging)
    deploy-environment: staging   # staging | production
    package-pattern: '*.nupkg'    # asset glob to download
    dry-run: false                # default false
  secrets: inherit
```

### push-release-info-to-aws-s3

Uploads a release's info (tag, URL, publish date, `.exe` assets) as JSON to `s3://<bucket>/<product lowercase>.txt`, then verifies the upload.

**Assumes:** the release is published (not draft/prerelease), tagged `X.Y.Z`, and has at least one `.exe` asset. Bucket is in `us-west-2`.

```yaml
# Basic
push-s3:
  uses: Particular/deployment-workflows/.github/workflows/push-release-info-to-aws-s3.yml@<version>
  with:
    version: 1.2.3
    product: ServiceInsight
    bucket: my-bucket
  secrets: inherit # needs AWS_ACCESSKEY and AWS_SECRETKEY secrets
```

```yaml
# Full
push-s3:
  uses: Particular/deployment-workflows/.github/workflows/push-release-info-to-aws-s3.yml@<version>
  with:
    version: 1.2.3          # release tag
    product: ServiceInsight # file name: serviceinsight.txt
    bucket: my-bucket
    dry-run: false          # default false
  secrets: inherit # AWS_ACCESSKEY and AWS_SECRETKEY
```

### push-docker-images

Copies images from GitHub Container Registry to Docker Hub with the tags from [determine-image-tags](#determine-image-tags), and updates the Docker Hub descriptions when the version is the new `latest`.

**Assumes:**
- Images are already in `ghcr.io/particular/<name>:<version>`.
- An image config file exists (see below).
- Caller grants `contents: read` and `packages: write`.

```yaml
# Basic
push-docker:
  uses: Particular/deployment-workflows/.github/workflows/push-docker-images.yml@<version>
  with:
    version: 1.2.3
  secrets: inherit # needs DOCKERHUB_USERNAME and DOCKERHUB_TOKEN
```

```yaml
# Full
push-docker:
  uses: Particular/deployment-workflows/.github/workflows/push-docker-images.yml@<version>
  with:
    version: 1.2.3
    image-config-file: .github/docker-images.json # default shown
    dry-run: false                                # default false
  secrets: inherit # DOCKERHUB_USERNAME and DOCKERHUB_TOKEN
```

Image config file - a non-empty JSON array; `name` is both the GHCR image and the `particular/<name>` Docker Hub repo:

```json
[
  {
    "name": "servicecontrol",
    "readme_filepath": "docker/servicecontrol/README.md",
    "short_description": "Particular Service Platform monitoring"
  }
]
```
