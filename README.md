# github-actions-multi-arch-manifest

A GitHub Action that creates and pushes multi-platform Docker manifests for amd64 and arm64 architectures to Harbor and optionally Amazon ECR.

## Description

This action combines platform-specific Docker images (`-amd64` and `-arm64`) into multi-architecture manifests. It creates manifests in Harbor (and ECR by default):
1. One for the specified image tag
2. One for the Git SHA (used by the tag-release workflow)

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `image-tag` | Image tag to create manifest for | Yes | - |
| `service-name` | Name of service for location to retrieve images in container repo | Yes | - |
| `business-unit` | Business unit of app for Harbor | Yes | - |
| `git-sha` | Git SHA to create manifest for | No | `github.sha` |
| `aws-account` | AWS account name to assume role for ECR | No | `bisnow` |
| `only-harbor` | Skip ECR and only push manifests to Harbor | No | `false` |

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
        business-unit: ${{ env.BUSINESS_UNIT }}
        aws-account: ${{ env.AWS_ACCOUNT || 'bisnow' }}
```

To push to Harbor only (skips AWS role assumption and ECR login):

```yaml
    - name: Create multi-arch manifests
      uses: bisnow/github-actions-multi-arch-manifest@v1.0
      with:
        image-tag: ${{ inputs.image-tag || env.TAG }}
        service-name: ${{ env.SERVICE_NAME }}
        business-unit: ${{ env.BUSINESS_UNIT }}
        only-harbor: 'true'
```

## Prerequisites

- Platform-specific images must already exist in the target registries with the `-amd64` and `-arm64` suffixes
- Harbor authentication must be configured on the runner (authentication is handled by the runner, not the action)
- Harbor images should be located at `harbor.bisnow.cloud/{business-unit}/{service-name}`
- When pushing to ECR (default): AWS credentials must be available; the action handles role assumption and ECR login automatically
- ECR images should be located at `560285300220.dkr.ecr.us-east-1.amazonaws.com/{service-name}`

## What It Does

1. *(If pushing to ECR)* Assumes the specified AWS role and logs into Amazon ECR
2. Sets up Docker Buildx
3. **Verifies all architecture images exist** in the target registries (with retry logic up to 2.5 minutes)
   - Waits for `{registry}:{image-tag}-amd64` and `{image-tag}-arm64`
   - Waits for `{registry}:{git-sha}-amd64` and `{git-sha}-arm64`
4. **Creates multi-platform manifests** in each target registry:
   - `{registry}:{image-tag}` combining `{image-tag}-amd64` and `{image-tag}-arm64`
   - `{registry}:{git-sha}` combining `{git-sha}-amd64` and `{git-sha}-arm64`
5. **Verifies all manifests** were created correctly:
   - Confirms each manifest contains `linux/amd64` and `linux/arm64` platforms
   - Verifies both tags reference the same manifest digest per registry (ensuring tag lookup works)

## Versioning

This action uses rolling major version tags. You can pin to:

- A specific version: `@v3.1.0` (exact, never changes)
- A major version: `@v3` (recommended, gets bug fixes and new features)

When a new semantic version tag (e.g., `v3.2.0`) is pushed, a GitHub Actions workflow automatically updates the corresponding major version tag (`v3`) to point to the new release.
