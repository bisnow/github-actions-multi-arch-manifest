# github-actions-multi-arch-manifest

A GitHub Action that creates and pushes multi-platform Docker manifests for amd64 and arm64 architectures in Amazon ECR.

## Description

This action combines platform-specific Docker images (`-amd64` and `-arm64`) into multi-architecture manifests. It creates two manifests:
1. One for the specified image tag
2. One for the Git SHA (used by the tag-release workflow)

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `image-tag` | Image tag to create manifest for | Yes | - |
| `ecr-registry` | ECR registry URL (e.g., `560285300220.dkr.ecr.us-east-1.amazonaws.com/biscred-api`) | Yes | - |
| `aws-account` | AWS account name for assuming role | No | `bisnow` |
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
        ecr-registry: ${{ env.ECR_REGISTRY }}
```

## Prerequisites

- Platform-specific images must already exist in ECR with the `-amd64` and `-arm64` suffixes
- AWS credentials must be configured (the action handles ECR login automatically)
- The action assumes an AWS role using `bisnow/github-actions-assume-role-for-environment@main`

## What It Does

1. Assumes the specified AWS role
2. Logs into Amazon ECR
3. Sets up Docker Buildx
4. **Verifies all architecture images exist** (with retry logic up to 2.5 minutes)
   - Waits for `{ecr-registry}:{image-tag}-amd64` and `{ecr-registry}:{image-tag}-arm64`
   - Waits for `{ecr-registry}:{git-sha}-amd64` and `{ecr-registry}:{git-sha}-arm64`
5. **Creates multi-platform manifests**:
   - `{ecr-registry}:{image-tag}` from `{image-tag}-amd64` and `{image-tag}-arm64`
   - `{ecr-registry}:{git-sha}` from `{git-sha}-amd64` and `{git-sha}-arm64`
6. **Verifies both manifests** were created correctly:
   - Confirms both manifests contain `linux/amd64` and `linux/arm64` platforms
   - Verifies both tags reference the same manifest digest (ensuring tag lookup works)

   ## Versioning

This action uses rolling major version tags. You can pin to:

- A specific version: `@v3.1.0` (exact, never changes)
- A major version: `@v3` (recommended, gets bug fixes and new features)

When a new semantic version tag (e.g., `v3.2.0`) is pushed, a GitHub Actions workflow automatically updates the corresponding major version tag (`v3`) to point to the new release.