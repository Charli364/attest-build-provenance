The `subject-path` parameter should identify the artifact for which you want
   to generate an attestation.

### Inputs

See [action.yml](action.yml)

```yaml
- uses: actions/attest-build-provenance@v2
  with:
    # Path to the artifact serving as the subject of the attestation. Must
    # specify exactly one of "subject-path" or "subject-digest". May contain a
    # glob pattern or list of paths (total subject count cannot exceed 1024).
    subject-path:

    # SHA256 digest of the subject for the attestation. Must be in the form
    # "sha256:hex_digest" (e.g. "sha256:abc123..."). Must specify exactly one
    # of "subject-path" or "subject-digest".
    subject-digest:

    # Subject name as it should appear in the attestation. Required unless
    # "subject-path" is specified, in which case it will be inferred from the
    # path.
    subject-name:

    # Whether to push the attestation to the image registry. Requires that the
    # "subject-name" parameter specify the fully-qualified image name and that
    # the "subject-digest" parameter be specified. Defaults to false.
    push-to-registry:

    # Whether to attach a list of generated attestations to the workflow run
    # summary page. Defaults to true.
    show-summary:

    # The GitHub token used to make authenticated API requests. Default is
    # ${{ github.token }}
    github-token:

  build:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
      attestations: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4
      - name: Build artifact
        run: make my-app
      - name: Attest
        uses: actions/attest-build-provenance@v2
        with:
          subject-path: '${{ github.workspace }}/my-app'

the specific image being attested is identified by the supplied digest.

Attestation bundles are stored in the OCI registry according to the [Cosign
Bundle Specification][10].

> **NOTE**: When pushing to Docker Hub, please use "index.docker.io" as the
> registry portion of the image name.

```yaml
name: build-attested-image

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      packages: write
      contents: read
      attestations: write
    env:
      REGISTRY: ghcr.io
      IMAGE_NAME: ${{ github.repository }}

    steps:
      - name: Checkout
        uses: actions/checkout@v4
      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - name: Build and push image
        id: push
        uses: docker/build-push-action@v5.0.0
        with:
          context: .
          push: true
          tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
      - name: Attest
        uses: actions/attest-build-provenance@v2
        id: attest
        with:
          subject-name: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          subject-digest: ${{ steps.push.outputs.digest }}
          push-to-registry: true


