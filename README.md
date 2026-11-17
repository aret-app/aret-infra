# aret-infra
Aret deployment: Docker Compose, k3s manifests, runbooks

## Shared CI workflows
Reusable [GitHub Actions workflows](https://docs.github.com/actions/sharing-automations/reusing-workflows)
that the other Aret repositories call, so every repo runs the same checks. All run on standard
GitHub-hosted runners, free for public repositories (including ARM).

| Workflow | What it does |
|---|---|
| [`rust-ci.yml`](.github/workflows/rust-ci.yml) | `cargo fmt --check`, `clippy -D warnings`, `cargo test`, `cargo deny check` (Rust version from the caller's `rust-toolchain.toml`). The test job has a **PostgreSQL 18 + PostGIS** service (`TEST_DATABASE_URL`, throwaway password `postgres`, CI only) and sets `REQUIRE_DB=1` so DB tests can't silently skip. |
| [`container-image.yml`](.github/workflows/container-image.yml) | Builds a Docker image natively for **amd64 and arm64**; with `push: true`, publishes one multi-arch image to `ghcr.io/aret-app/<image>` tagged `sha-<commit>` and `latest` (default branch) |

### Using them
```yaml
# .github/workflows/ci.yml in a Rust service repo
name: CI
on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read
  packages: write   # lets the image workflow push to GHCR

jobs:
  rust:
    uses: aret-app/aret-infra/.github/workflows/rust-ci.yml@main

  images:
    needs: rust
    strategy:
      matrix:
        service: [aret-identity, aret-tracks]
    uses: aret-app/aret-infra/.github/workflows/container-image.yml@main
    with:
      image: ${{ matrix.service }}
      build-args: SERVICE=${{ matrix.service }}
      push: ${{ github.event_name == 'push' }}   # pull requests only build
```

Check names appear as `rust / fmt`, `rust / clippy`, `rust / test`, `rust / cargo-deny` and
`images (…) / build (amd64|arm64)`; the Rust ones are required in each repo's `protect-main` ruleset.

Dependabot keeps the actions used here up to date ([`.github/dependabot.yml`](.github/dependabot.yml)).
