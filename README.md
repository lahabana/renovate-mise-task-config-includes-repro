# renovate-mise-task-config-includes-repro

Minimal reproduction repository for [renovatebot/renovate#47868](https://github.com/renovatebot/renovate/pull/47868):
Renovate's `mise` manager does not extract remote (`git::`) refs from the mise
`[task_config]` `includes` array, so shared task directories pinned by git tag
are invisible to Renovate — they never appear on the dependency dashboard and
are never updated.

## What is in this repository

[`.mise.toml`](./.mise.toml) exercises the three ways mise can depend on
external versions, two of which reference **this same repository's** `tasks/`
directory (self-referential, so the repro is fully self-contained), each pinned
at a different tag:

| Dependency form | Pinned at | Latest | Released Renovate | Renovate with #47868 |
|---|---|---|---|---|
| `[tools]` `jq = "1.7.1"` | `1.7.1` | `1.8.2` | ✅ extracted | ✅ extracted |
| `[tasks.remote-hello]` `file = "git::…//tasks/hello.sh?ref=v1.0.1"` | `v1.0.1` | `v1.0.2` | ✅ extracted (`task-remote-hello-file`) | ✅ extracted |
| `[task_config]` `includes = ["git::…//tasks?ref=v1.0.0"]` | `v1.0.0` | `v1.0.2` | ❌ **not extracted (bug)** | ✅ extracted (`task-include`) |
| `[task_config]` `includes = ["tasks"]` (local path) | — | — | ✅ ignored | ✅ ignored, no dashboard noise |

- [`tasks/`](./tasks) contains the tasks that the remote refs serve.
- Tags `v1.0.0`, `v1.0.1` and `v1.0.2` exist, so an update is available for
  every pinned ref.

## What Renovate should do (after the fix)

With the fix from #47868, the
`git::https://github.com/lahabana/renovate-mise-task-config-includes-repro.git//tasks?ref=v1.0.0`
entry is extracted as a `task-include` dependency (`github-tags` datasource)
and Renovate opens a PR bumping `ref=v1.0.0` → `ref=v1.0.2`. The local
`"tasks"` entry is dropped: local paths carry no version, and the fix avoids
emitting a skipped `unsupported-url` dependency onto every dependency
dashboard (see [review
comment](https://github.com/renovatebot/renovate/pull/47868#discussion_r4552115302)).

## What Renovate does today (without the fix)

The `[task_config] includes` entry is ignored entirely: no dependency is
extracted, nothing appears in the dependency dashboard, and no PR is created —
while the `jq` tool and the `tasks.<name>.file` dependency are picked up
normally.

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
