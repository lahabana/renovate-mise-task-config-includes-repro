# renovate-mise-task-config-includes-repro

Minimal reproduction repository for [renovatebot/renovate#47868](https://github.com/renovatebot/renovate/pull/47868):
Renovate's `mise` manager does not extract remote (`git::`) refs from the mise
`[task_config]` `includes` array, so shared task directories pinned by git tag
are invisible to Renovate — they never appear on the dependency dashboard and
are never updated.

## What is in this repository

- [`.mise.toml`](./.mise.toml) declares `[task_config] includes` with a remote
  `git::` entry pinning **this same repository's** `tasks/` directory at tag
  `v1.0.0` (self-referential, so the repro is fully self-contained).
- [`tasks/`](./tasks) contains the tasks that the include serves.
- Tag `v1.0.1` exists with newer task content, so an update is available.

## What Renovate should do (after the fix)

With the fix from #47868, the `git::https://github.com/lahabana/renovate-mise-task-config-includes-repro.git//tasks?ref=v1.0.0`
entry is extracted as a `task-include` dependency (`github-tags` datasource)
and Renovate opens a PR bumping `ref=v1.0.0` → `ref=v1.0.1`.

## What Renovate does today (without the fix)

The entry is ignored entirely: no dependency is extracted, nothing appears in
the dependency dashboard, and no PR is created.

## Reproducing

Run Renovate (with `mise` manager enabled) against this repository and check
the extracted `packageFiles` for `mise`:

```bash
RENOVATE_PLATFORM=github \
RENOVATE_TOKEN=<token> \
RENOVATE_REPOSITORIES=lahabana/renovate-mise-task-config-includes-repro \
node lib/renovate.ts
```

mise itself resolves the include fine (`mise tasks` lists the shared tasks);
only Renovate cannot see it.
