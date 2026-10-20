# spnngl/.github

Shared GitHub Actions workflows, actions and Renovate preset of the `spnngl`
repositories.

Callers use `@main` on purpose: a change here applies everywhere at once. The
shared Renovate preset keeps Renovate from pinning these references.

## Reusable workflows

| Workflow | Trigger it from | Job permissions | Inputs |
|----------|-----------------|-----------------|--------|
| [`conventional-commits.yml`](.github/workflows/conventional-commits.yml) | `pull_request` | `contents: read`, `pull-requests: read` | `allowed-commit-types` |
| [`betterleaks.yml`](.github/workflows/betterleaks.yml): secrets in the git history | `pull_request` | `contents: read`, `security-events: write` | `confidence` |
| [`betterleaks-github.yml`](.github/workflows/betterleaks-github.yml): secrets in PRs, issues, comments | `schedule` | `contents: read`, `issues: read`, `pull-requests: read`, `security-events: write` | `confidence` |
| [`renovate.yml`](.github/workflows/renovate.yml): `renovate-config-validator --strict` | `pull_request` / `push` on the Renovate config | `contents: read` | `config-files` |
| [`image-scan.yml`](.github/workflows/image-scan.yml): vulnerabilities in the repository's ghcr.io images, with `scan-image` | `schedule` | `contents: read`, `packages: read`, `security-events: write` | `packages`, `max-age-days` |

A called workflow gets at most the permissions of the calling job, so grant
them there:

```yaml
name: Secret Scanning

on:
  pull_request:
    branches:
      - main

permissions: {}

# Cancels the run of an outdated commit.
concurrency:
  group: ${{ github.workflow }}-${{ github.event.pull_request.number || github.ref }}
  cancel-in-progress: true

jobs:
  betterleaks:
    permissions:
      contents: read
      security-events: write
    uses: spnngl/.github/.github/workflows/betterleaks.yml@main
```

Repository-specific settings stay in the calling repository:
`.betterleaks.toml` and `.betterleaksignore` at its root, triggers, `paths`
filters.

The betterleaks workflows don't upload the findings of pull requests from
forks: their token can't write to code scanning. The scan still fails the
job.

## Actions

[`setup-betterleaks`](.github/actions/setup-betterleaks/action.yml): installs
the pinned betterleaks release, checksum-verified. Both betterleaks workflows
use it at `@main`, so a pull request changing it only takes effect once
merged; CI here tests the version on `main`.

```yaml
- uses: spnngl/.github/.github/actions/setup-betterleaks@main
```

[`scan-image`](.github/actions/scan-image/action.yml): scans one image with
Grype and Trivy (both, Trivy has no Fedora data), pinned and
checksum-verified. Logs every finding, sums them up in the job summary, and
with `fail: true` fails on a Critical vulnerability with a fix. Its SARIF
outputs (`grype-sarif`, `trivy-sarif`) keep the High and Critical findings.
Ignore files, at the repository root: `.grype.yaml` (`ignore:` rules) and
`.trivyignore.yaml`. `image-scan.yml` uses it on the pushed images; to scan
an image that is built but not pushed, in the build job, give its OCI
layout. In a pull request, give the pushed image it replaces as
`baseline`: only the findings it doesn't have fail (not those of upstream's
binaries, nor a vulnerability published since), and nothing fails when it
can't be scanned (not pushed yet):

```yaml
- run: podman save --format oci-dir -o "$RUNNER_TEMP/oci" "$TAG"
- uses: spnngl/.github/.github/actions/scan-image@main
  with:
    oci-layout: ${{ runner.temp }}/oci
    baseline: ${{ env.TAG }}
    platform: linux/${{ env.ARCH }}
    fail: true
```

A registry image instead: `image: ghcr.io/<owner>/<name>:<tag>` and
`platform: linux/<arch>`.

## Renovate preset

[`renovate.default.json5`](renovate.default.json5): `config:best-practices`,
semantic commits, `Europe/Paris` timezone, `dependencies` label, automerge of
non-major updates under `.github/` and of every GitHub Actions update, and
versions annotated with a
`# renovate:` comment in workflows and actions:

```yaml
env:
  # renovate: datasource=github-releases depName=betterleaks/betterleaks
  BETTERLEAKS_VERSION: "1.9.0"
```

Not named `default.json`, so callers give the file name (Renovate only
looks up `default.json` for a bare `github>spnngl/.github`):

```json5
{
  $schema: "https://docs.renovatebot.com/renovate-schema.json",
  extends: ["github>spnngl/.github:renovate.default.json5"],
}
```
