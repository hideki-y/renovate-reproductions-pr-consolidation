# `bumpVersions` skips a project when multiple `pyproject.toml`s are consolidated into one PR

First, read the [Renovate minimal reproduction instructions](https://github.com/renovatebot/renovate/blob/main/docs/development/minimal-reproductions.md).

## Current behavior

This repository has two independent Python projects, each with its own
`pyproject.toml` + `uv.lock`:

- `project-a/pyproject.toml` — depends on `requests==2.31.0`, `click==8.1.3`
- `project-b/pyproject.toml` — depends on `flask==3.0.0`

`.renovaterc.json5` has two `packageRules`:

1. A grouping rule (`matchFileNames: ["project-a/**", "project-b/**"]` +
   `groupName`/`groupSlug: "consolidated"`) so every dependency update across
   both projects lands on one branch/PR instead of one PR per dependency.
2. A `bumpVersions` rule (`matchUpdateTypes: ["major", "minor", "patch"]`)
   that patch-bumps each project's own `version` field in
   `{{packageFileDir}}/pyproject.toml` and `{{packageFileDir}}/uv.lock`
   whenever one of its dependencies is updated. Both patterns are templated
   on `{{packageFileDir}}`, so the same single rule is meant to apply
   identically to `project-a/` and `project-b/`.

Running Renovate v46.2.5 (the latest release at the time) via the
`.github/workflows/renovate.yml` workflow in this repo (the official
`renovatebot/github-action@v46.2.5`) produced
[PR #1](https://github.com/hideki-y/renovate-reproductions-pr-consolidation/pull/1),
"Update consolidated dependencies".

**The grouping worked as expected**: all three dependency updates (`click`,
`flask`, `requests`) landed in that single PR on branch
`renovate/consolidated`.

**The version bump did not apply to both projects.** Only `project-a`'s own
package version was bumped; `project-b`'s was left untouched, even though
`flask` was updated in the same PR under the identical rule:

| File | Own `version =` bumped? | Dependency updated? |
|---|---|---|
| `project-a/pyproject.toml` | ✅ `0.1.0` → `0.1.1` | ✅ `requests`, `click` |
| `project-a/uv.lock` | ✅ `0.1.0` → `0.1.1` | ✅ `requests`, `click` |
| `project-b/pyproject.toml` | ❌ unchanged (`0.1.0`) | ✅ `flask` |
| `project-b/uv.lock` | ❌ unchanged (`0.1.0`) | ✅ `flask` |

```diff
--- a/project-a/pyproject.toml
+++ b/project-a/pyproject.toml
-version = "0.1.0"
+version = "0.1.1"
...
-    "requests==2.31.0",
+    "requests==2.34.2",
-    "click==8.1.3",
+    "click==8.5.0",
```

```diff
--- a/project-b/pyproject.toml
+++ b/project-b/pyproject.toml
 version = "0.1.0"          # unchanged
 requires-python = ">=3.11"
 dependencies = [
-    "flask==3.0.0",
+    "flask==3.1.3",
 ]
```

### Note on local dry-runs

`renovate --platform=local` (any `--dry-run` tier, or a real run) never
executes this stage: the `local` platform doesn't write files or create
branches at all, by design. `bumpVersions` execution can only be observed via
a real run against an actual git-hosting platform (GitHub, here) — hence the
GitHub Actions workflow in this repo instead of a local reproduction script.

## Reproducing

1. Fork/clone this repository (or use it directly, if you have write access).
2. Ensure **Settings → Actions → General → "Allow GitHub Actions to create
   and approve pull requests"** is enabled.
3. Manually trigger the `Renovate (reproduction)` workflow:
   `gh workflow run renovate.yml` (or Actions tab → "Run workflow").
4. Inspect the resulting PR's diff: `project-a`'s `pyproject.toml`/`uv.lock`
   version is bumped; `project-b`'s is not.

## Expected behavior

Both `project-a/pyproject.toml` and `project-b/pyproject.toml` should have
their own `version` bumped, since both are matched by the identical
`bumpVersions` rule and both had a dependency updated within the same
consolidated PR.

## Link to the Renovate Issue or Discussion

Not yet filed.
