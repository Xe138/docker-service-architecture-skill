# CI/CD Pipelines

Automated Docker builds triggered by semantic version tags for GitHub and Gitea.

## Automated Docker Builds

Both **GitHub** and **Gitea** support automated Docker builds triggered by semantic version tags. The key differences are in registry authentication, available actions, and runner capabilities.

### Trigger Pattern

Both platforms use the same tag pattern to trigger builds:

```yaml
on:
  push:
    tags:
      - 'v*.*.*'    # Simple glob (works everywhere)
      # or
      - 'v[0-9]+.[0-9]+.[0-9]+*'  # More precise regex-style
```

### Prerelease Detection

Tags containing `-alpha`, `-beta`, or `-rc` are considered prereleases and should NOT receive the `latest` tag:

```bash
# v1.0.0 → stable → gets :latest
# v1.0.0-alpha → prerelease → version tag only
# v1.0.0-beta.2 → prerelease → version tag only
# v1.0.0-rc.1 → prerelease → version tag only
```

## GitHub Actions Workflow

GitHub provides first-class actions for Docker operations with built-in caching and metadata extraction.

```yaml
# .github/workflows/build.yaml
name: Build and Push Docker Image

on:
  push:
    tags:
      - 'v*.*.*'

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write    # Required for ghcr.io

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}  # Auto-provided

      - name: Extract metadata for Docker
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=raw,value=latest,enable=${{ !contains(github.ref, '-alpha') && !contains(github.ref, '-beta') && !contains(github.ref, '-rc') }}

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and push Docker image
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

**Key features:**
- `docker/metadata-action@v5`: Automatic semver tag generation
- `docker/build-push-action@v6`: Multi-platform builds, layer caching
- `cache-from/cache-to: type=gha`: GitHub Actions native cache (fast)
- `GITHUB_TOKEN`: Auto-provided, no secret setup needed for ghcr.io

## Gitea Actions Workflow

Gitea Actions are GitHub-compatible but lack some marketplace actions. Use direct Docker commands instead.

```yaml
# .gitea/workflows/release.yml
name: Build and Push Docker Image

on:
  push:
    tags:
      - 'v*.*.*'

env:
  REGISTRY: git.example.com        # Your Gitea container registry
  IMAGE_NAME: username/projectname

jobs:
  build:
    runs-on: ubuntu-docker         # Runner with Docker access
    steps:
      - name: Checkout repository
        run: |
          git clone --depth 1 --branch ${GITHUB_REF_NAME} ${GITHUB_SERVER_URL}/${GITHUB_REPOSITORY}.git .

      - name: Extract version from tag
        id: version
        run: |
          VERSION=${GITHUB_REF#refs/tags/}
          echo "VERSION=$VERSION" >> $GITHUB_OUTPUT
          if [[ "$VERSION" == *-alpha* ]] || [[ "$VERSION" == *-beta* ]] || [[ "$VERSION" == *-rc* ]]; then
            echo "IS_PRERELEASE=true" >> $GITHUB_OUTPUT
          else
            echo "IS_PRERELEASE=false" >> $GITHUB_OUTPUT
          fi

      - name: Log in to Container Registry
        run: echo "${{ secrets.REGISTRY_TOKEN }}" | docker login ${{ env.REGISTRY }} -u ${{ gitea.actor }} --password-stdin

      - name: Build and push Docker image
        run: |
          docker build -t ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.version.outputs.VERSION }} .
          docker push ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.version.outputs.VERSION }}

          if [ "${{ steps.version.outputs.IS_PRERELEASE }}" = "false" ]; then
            docker tag ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.version.outputs.VERSION }} ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
            docker push ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
          fi

      - name: List images
        run: docker images | grep projectname
```

**Key differences from GitHub:**
- `runs-on: ubuntu-docker`: Custom runner label with Docker daemon
- Manual `git clone` instead of `actions/checkout` (Gitea compatibility)
- Direct `docker build/push` instead of build-push-action
- `secrets.REGISTRY_TOKEN`: Must be configured in Gitea secrets
- `${{ gitea.actor }}` instead of `${{ github.actor }}`

## Platform Comparison

| Feature | GitHub Actions | Gitea Actions |
|---------|---------------|---------------|
| Checkout | `actions/checkout@v4` | `git clone` command |
| Registry auth | `docker/login-action@v3` | `docker login` command |
| Build/push | `docker/build-push-action@v6` | `docker build && docker push` |
| Caching | `type=gha` (built-in) | Manual or none |
| Token for ghcr.io | Auto-provided `GITHUB_TOKEN` | N/A |
| Registry token | Auto for ghcr.io | Manual secret setup |
| Semver parsing | `docker/metadata-action` | Manual bash script |
| Actor variable | `github.actor` | `gitea.actor` |

## Multi-Service Builds

For projects with multiple services, use a matrix strategy:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        service: [api, worker, frontend]
    steps:
      - uses: actions/checkout@v4
      - name: Build and push
        run: |
          docker build -t $REGISTRY/${{ matrix.service }}:$VERSION \
            -f services/${{ matrix.service }}/Dockerfile .
          docker push $REGISTRY/${{ matrix.service }}:$VERSION
```

## Creating a Release

```bash
# Stable release
git tag v1.0.0
git push origin v1.0.0

# Prerelease (no :latest tag)
git tag v1.1.0-beta.1
git push origin v1.1.0-beta.1
```
