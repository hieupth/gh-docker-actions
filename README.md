# Github Docker Actions

A small collection of reusable GitHub Actions related to Docker.

## Actions

### 1) `docker-multiarch-build`
Builds and pushes a **single-platform** image using **push-by-digest**, then uploads the resulting digest as a workflow artifact.

Typical usage: run this action in a matrix over platforms (e.g., `linux/amd64`, `linux/arm64`). A later job can merge those digests into one multi-arch tag.

**Inputs**
- `registry` (optional, default: `docker.io`) – Registry host for login.
- `username` (required) – Registry username.
- `password` (required) – Registry password/token.
- `image` (required) – Image reference **without tag** (e.g., `docker.io/user/app`).
- `tag` (required) – The tag name used to name digest artifacts (caller can pass `matrix.tag`).
- `platform` (required) – Single platform for this job (e.g., `linux/amd64`).
- `context` (optional, default: `.`) – Build context.
- `dockerfile` (optional, default: `dockerfile`) – Dockerfile path.
- `build_args` (optional) – Build args as newline-separated `KEY=VALUE`.
- `artifact_prefix` (optional, default: `digests`) – Artifact name prefix.
- `retention_days` (optional, default: `1`) – Artifact retention.

---

### 2) `docker-multiarch-merge`
Downloads digest artifacts created by `docker-multiarch-build`, then creates and pushes a **multi-arch manifest list** for `image:tag`.

**Inputs**
- `registry` (optional, default: `docker.io`) – Registry host for login.
- `username` (required) – Registry username.
- `password` (required) – Registry password/token.
- `image` (required) – Image reference **without tag** (e.g., `docker.io/user/app`).
- `tag` (required) – Tag to publish (e.g., `25.11`).
- `artifact_prefix` (optional, default: `digests`) – Artifact name prefix (must match build action).
- `digests_path` (optional, default: `/tmp/digests`) – Where digests are downloaded.

## Example workflow

```yaml
name: build

on:
  workflow_dispatch:
  push:
    branches: ["main"]

jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        platform: ["linux/amd64", "linux/arm64"]
        tag: ["25.11"]

    steps:
      - uses: actions/checkout@v4

      # Optional: compute build args dynamically in caller
      - name: Compute BASE_IMAGE
        id: base
        shell: bash
        run: |
          set -euo pipefail
          # Example only:
          echo "base_image=ubuntu:22.04" >> "$GITHUB_OUTPUT"

      - name: Build per-platform digest
        uses: hieupth/docker-action/docker-multiarch-build@v1
        with:
          registry: docker.io
          username: ${{ vars.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
          image: docker.io/${{ vars.DOCKERHUB_USERNAME }}/mamba
          tag: ${{ matrix.tag }}
          platform: ${{ matrix.platform }}
          dockerfile: dockerfile
          context: .
          build_args: |
            BASE_IMAGE=${{ steps.base.outputs.base_image }}

  merge:
    runs-on: ubuntu-latest
    needs: build
    strategy:
      matrix:
        tag: ["25.11"]

    steps:
      - name: Merge digests into multi-arch tag
        uses: hieupth/docker-action/docker-multiarch-merge@v1
        with:
          registry: docker.io
          username: ${{ vars.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
          image: docker.io/${{ vars.DOCKERHUB_USERNAME }}/mamba
          tag: ${{ matrix.tag }}
          artifact_prefix: digests
```

## Notes
docker-multiarch-build must upload unique artifact names per platform; docker-multiarch-merge downloads them using an artifact name pattern.

## LICENSE

This project is licensed under a [Apache License 2.0](LICENSE).

Copyritght &copy; 2025 [Hieu Pham](https://github.com/hieupth). All rights reserved.