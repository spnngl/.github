# spnngl/.github

Reusable workflows, composite actions and the Renovate preset shared by the
spnngl repositories (charts, images). `README.md` is the consumers' doc.

## Blast radius

Consumers call everything at `@main`: a merge ships to every repository at
once, with no version to roll back to.

- Keep inputs backward compatible: a new input gets a default. Renaming or
  removing an input, or needing a new permission, breaks every caller until
  each one is updated (a called workflow gets at most the caller job's
  permissions).
- Update the `README.md` table (trigger, job permissions, inputs) in the same
  change.

## Workflows and actions

- `uses: ./...` in a reusable workflow resolves in the caller's checkout, so
  workflows call own actions as `spnngl/.github/.github/actions/<name>@main`.
  A pull request therefore runs the `main` version of the actions: an action
  change is tested only once merged. Merge a new action in its own pull
  request first, then the workflows using it; once `main` requires checks,
  that first merge needs an admin bypass.
- `ci.yml` calls the workflows through `./` paths, so a pull request tests its
  own workflow changes. Add every new reusable workflow there.
- Hardening is pinning plus minimal permissions; step-security harden-runner
  was removed on purpose.

## Pinning

- Third-party actions: full commit SHA with a `# vX.Y.Z` comment. Container
  images: `tag@sha256:...`.
- Tool versions: `# renovate: datasource=<datasource> depName=<name>` on the
  line right above `KEY: "<version>"`, the exact shape the preset's regex
  manager reads.
- `spnngl/*` references stay at `@main`: the preset disables their updates
  and `.github/zizmor.yml` allows them.

## Renovate preset

- Consumers extend `github>spnngl/.github:renovate.default.json5`; the file
  name is required, the bare form only looks up `default.json`.
- renovate-config-validator doesn't resolve presets: a preset change is
  exercised only by the consumers' Renovate runs.

## Verifying

Done when all three pass:

```sh
actionlint
GH_TOKEN="$(gh auth token)" uvx zizmor@<version in ci.yml> .  # token: online audits, as in CI
docker run --rm -v "${PWD}:/usr/src/app:ro" -w /usr/src/app <image in .github/workflows/renovate.yml> \
  renovate-config-validator --strict --no-global renovate.json5 renovate.default.json5
```

`--no-global`: with file arguments, the validator otherwise checks them as
global config.

A script in a `run:` block is done once it ran locally against real data
(e.g. a charts pull request through `gh api`), failure cases included.
