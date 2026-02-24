# github-actions-multi-arch-manifest

A GitHub Action that creates and pushes multi-platform Docker manifests for amd64 and arm64 architectures in Harbor.

## Description

This action combines platform-specific Docker images (`-amd64` and `-arm64`) into multi-architecture manifests. It creates two manifests:
1. One for the specified image tag
2. One for the Git SHA (used by the tag-release workflow)

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `image-tag` | Image tag to create manifest for | Yes | - |
| `service-name` | Name of service for location to retrieve images in container repo | Yes | - |
| `namespace` | Namespace of app for Harbor | Yes | - |
| `git-sha` | Git SHA to create manifest for | No | `github.sha` |

## Usage

```yaml
create-manifest:
  name: Create Multi-Platform Manifest
  runs-on: arc-runners-bisnow
  needs: build-and-push
  if: needs.build-and-push.result == 'success'
  steps:
    - name: Create multi-arch manifests
      uses: bisnow/github-actions-multi-arch-manifest@v1.0
      with:
        image-tag: ${{ inputs.image-tag || env.TAG }}
        service-name: ${{ env.SERVICE_NAME }}
        namespace: ${{ env.NAMESPACE }}
```

## Prerequisites

- Platform-specific images must already exist in Harbor with the `-amd64` and `-arm64` suffixes
- Harbor authentication must be configured on the runner (authentication is handled by the runner, not the action)
- Images should be located at `harbor.bisnow.cloud/{namespace}/{service-name}`

## What It Does

1. Sets up Docker Buildx
2. **Verifies all architecture images exist** (with retry logic up to 2.5 minutes)
   - Waits for `harbor.bisnow.cloud/{namespace}/{service-name}:{image-tag}-amd64` and `{image-tag}-arm64`
   - Waits for `harbor.bisnow.cloud/{namespace}/{service-name}:{git-sha}-amd64` and `{git-sha}-arm64`
3. **Creates multi-platform manifests**:
   - `harbor.bisnow.cloud/{namespace}/{service-name}:{image-tag}` from `{image-tag}-amd64` and `{image-tag}-arm64`
   - `harbor.bisnow.cloud/{namespace}/{service-name}:{git-sha}` from `{git-sha}-amd64` and `{git-sha}-arm64`
4. **Verifies both manifests** were created correctly:
   - Confirms both manifests contain `linux/amd64` and `linux/arm64` platforms
   - Verifies both tags reference the same manifest digest (ensuring tag lookup works)

## Versioning

This action uses rolling major version tags. You can pin to:

- A specific version: `@v3.1.0` (exact, never changes)
- A major version: `@v3` (recommended, gets bug fixes and new features)

When a new semantic version tag (e.g., `v3.2.0`) is pushed, a GitHub Actions workflow automatically updates the corresponding major version tag (`v3`) to point to the new release.
