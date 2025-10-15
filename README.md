# Build Deploy Docker

![Latest Release](https://img.shields.io/github/v/release/p6m-workflows/build-deploy-docker?style=flat-square&label=Latest%20Release&color=blue)

## Description

A reusable GitHub Actions workflow for building and deploying multi-architecture Docker images. This workflow builds Docker images for both `linux/amd64` and `linux/arm64` platforms in parallel, then combines them into a single multi-arch manifest and pushes to your Docker registry.

**Key Features:**
- Multi-architecture support (amd64 and arm64)
- Parallel builds for faster execution
- GitHub Actions cache support for layer caching
- Flexible tag management using docker/metadata-action format
- Support for build args, secrets, and secret files
- Customizable runners for arm64 builds
- Optional push (can run in cache-only mode)

## Usage

Add this workflow as a job in your repository's workflow file:

```yaml
jobs:
  build-and-deploy:
    uses: p6m-workflows/build-deploy-docker/.github/workflows/workflow.yml@main
    with:
      tags: |
        type=ref,event=branch
        type=ref,event=pr
        type=semver,pattern={{version}}
        type=semver,pattern={{major}}.{{minor}}
    secrets:
      username: ${{ secrets.DOCKER_USERNAME }}
      password: ${{ secrets.DOCKER_PASSWORD }}
```

## Inputs

### Required Inputs

| Name | Type | Description |
|------|------|-------------|
| `tags` | `string` | Tags to apply to the final multi-arch image. Uses the same format as [docker/metadata-action](https://github.com/docker/metadata-action). |

### Optional Inputs

| Name | Type | Default | Description |
|------|------|---------|-------------|
| `github_ref` | `string` | _(current ref)_ | The GitHub ref to checkout. If not specified, defaults to the ref that triggered the workflow. |
| `registry` | `string` | `vars.ARTIFACTORY_HOSTNAME` | The hostname of the Docker registry (as used in `docker login`). |
| `image-name` | `string` | `${{ github.repository }}` | The namespaced Docker image name. Example: `myorg/myimage` |
| `context` | `string` | `.` | The path to the Docker build context. |
| `dockerfile` | `string` | `./Dockerfile` | The absolute path to the Dockerfile to build. |
| `build-args` | `string` | `""` | Build args to pass to docker build in `key=value` format (one per line). |
| `linux-arm-runner` | `string` | `ubuntu-latest` | The `runs-on` tag for the linux arm64 runner. |
| `push` | `boolean` | `true` | Whether to push the built images to the registry. Set to `false` to only refresh cache. |

### Secrets

| Name | Required | Description |
|------|----------|-------------|
| `username` | Yes | The username to authenticate with the Docker registry. |
| `password` | Yes | The password to authenticate with the Docker registry. |
| `secrets` | No | Raw secrets to pass to docker build as environment variables in `key=value` format (one per line). |
| `secret-envs` | No | Environment variables to passthrough to docker build in `key=envname` format (one per line). |
| `secret-files` | No | Files to mount from the host machine into docker build in `key=filepath` format (one per line). |

## Outputs

| Name | Description |
|------|-------------|
| `digest` | The SHA256 digest of the pushed multi-arch manifest. Available only when `push: true`. |

## Examples

### Basic Usage

Build and push a Docker image with semantic version tags:

```yaml
name: Build Docker Image

on:
  push:
    tags:
      - 'v*'

jobs:
  docker:
    uses: p6m-workflows/build-deploy-docker/.github/workflows/workflow.yml@main
    with:
      tags: |
        type=semver,pattern={{version}}
        type=semver,pattern={{major}}.{{minor}}
        type=semver,pattern={{major}}
    secrets:
      username: ${{ secrets.DOCKER_USERNAME }}
      password: ${{ secrets.DOCKER_PASSWORD }}
```

### Custom Dockerfile Location

```yaml
jobs:
  docker:
    uses: p6m-workflows/build-deploy-docker/.github/workflows/workflow.yml@main
    with:
      context: ./app
      dockerfile: ./app/Dockerfile.prod
      tags: |
        type=ref,event=branch
        type=sha
    secrets:
      username: ${{ secrets.DOCKER_USERNAME }}
      password: ${{ secrets.DOCKER_PASSWORD }}
```

### With Build Arguments

```yaml
jobs:
  docker:
    uses: p6m-workflows/build-deploy-docker/.github/workflows/workflow.yml@main
    with:
      build-args: |
        NODE_VERSION=18
        BUILD_ENV=production
      tags: |
        type=raw,value=latest
        type=sha
    secrets:
      username: ${{ secrets.DOCKER_USERNAME }}
      password: ${{ secrets.DOCKER_PASSWORD }}
```

### Using Custom Registry and Image Name

```yaml
jobs:
  docker:
    uses: p6m-workflows/build-deploy-docker/.github/workflows/workflow.yml@main
    with:
      registry: ghcr.io
      image-name: myorg/myapp
      tags: |
        type=ref,event=branch
        type=semver,pattern={{version}}
    secrets:
      username: ${{ github.actor }}
      password: ${{ secrets.GITHUB_TOKEN }}
```

### With Docker Build Secrets

```yaml
jobs:
  docker:
    uses: p6m-workflows/build-deploy-docker/.github/workflows/workflow.yml@main
    with:
      tags: |
        type=sha
    secrets:
      username: ${{ secrets.DOCKER_USERNAME }}
      password: ${{ secrets.DOCKER_PASSWORD }}
      secrets: |
        NPM_TOKEN=${{ secrets.NPM_TOKEN }}
        API_KEY=${{ secrets.API_KEY }}
```

### Cache-Only Mode (No Push)

Useful for validating builds or warming up the cache:

```yaml
jobs:
  docker:
    uses: p6m-workflows/build-deploy-docker/.github/workflows/workflow.yml@main
    with:
      push: false
      tags: |
        type=sha
    secrets:
      username: ${{ secrets.DOCKER_USERNAME }}
      password: ${{ secrets.DOCKER_PASSWORD }}
```

### Using Custom ARM64 Runner

If you have self-hosted ARM64 runners:

```yaml
jobs:
  docker:
    uses: p6m-workflows/build-deploy-docker/.github/workflows/workflow.yml@main
    with:
      linux-arm-runner: self-hosted-arm64
      tags: |
        type=ref,event=branch
    secrets:
      username: ${{ secrets.DOCKER_USERNAME }}
      password: ${{ secrets.DOCKER_PASSWORD }}
```

### Capturing the Digest Output

```yaml
jobs:
  docker:
    uses: p6m-workflows/build-deploy-docker/.github/workflows/workflow.yml@main
    with:
      tags: |
        type=sha
    secrets:
      username: ${{ secrets.DOCKER_USERNAME }}
      password: ${{ secrets.DOCKER_PASSWORD }}

  use-digest:
    needs: docker
    runs-on: ubuntu-latest
    steps:
      - name: Use the digest
        run: |
          echo "Image digest: ${{ needs.docker.outputs.digest }}"
```