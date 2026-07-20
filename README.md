# CICD

Central, shared CI/CD engine for Chisel/RTL project repos across the organization. This repo contains the actual CI/CD logic once; project repos each carry only a few thin "caller" files that reference it — no CI/CD logic is duplicated per project.

## Structure

```
.github/
├── actions/
│   └── setup-toolchain/
│       └── action.yml        Shared toolchain setup (composite action)
└── workflows/
    ├── Reusable-RTL-CI.yml       CI: merge-check, tests, lint
    ├── Reusable-RTL-CD.yml       CD: RTL generation + GitHub Release
    └── validate.yml          Self-check: lints this repo's own workflow files
```

## How a Project Repo Plugs Into This

A project repo needs three small files under its own `.github/workflows/`:

```yaml
# RTL-CI.yml
on: [push, pull_request]
jobs:
  ci:
    uses: shau-08/CICD/.github/workflows/Reusable-RTL-CI.yml@main
    secrets: inherit
```

```yaml
# RTL-CD.yml
on: workflow_dispatch
jobs:
  cd:
    permissions:
      contents: write
    uses: shau-08/CICD/.github/workflows/Reusable-RTL-CD.yml@main
    with:
      build_output_path: generated_sv_dir
    secrets: inherit
```

```yaml
# validate.yml
on:
  pull_request:
    paths: ['.github/workflows/**']
  push:
    branches: [main]
    paths: ['.github/workflows/**']
jobs:
  actionlint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: bash <(curl -s https://raw.githubusercontent.com/rhysd/actionlint/main/scripts/download-actionlint.bash)
      - run: ./actionlint -color
```

The `@main` reference is what makes this a shared engine — updating logic here propagates to every project repo automatically, without editing them individually.

### What a project repo must provide

For `Reusable-RTL-CI.yml` / `Reusable-RTL-CD.yml` to work against it, a project repo needs:

- A `Makefile` with, at minimum, a `test` target. A `lint` target is optional — CI probes for it (`make -n lint`) and skips cleanly if absent.
- A `Makefile` with an `rtl-dispatch` target (or pass `rtl_target_override` as a CD input to bypass it) for CD to generate RTL.
- A `build.sc` with exactly one top-level module whose `millSourcePath` is `os.pwd` — the lint step auto-detects the project name from this convention; more than one, or none, and it fails with a clear error rather than guessing.
- Optionally, a `cd.config` file (`RTL_TARGET=rtl` or `RTL_TARGET=lazyrtl`, plus a matching `TARGET=...`) to control CD without needing workflow inputs.

## `Reusable-RTL-CI.yml`

Triggered via `workflow_call`. Two jobs:

- **`check-mergeable`** — PRs targeting `main` only. Attempts a real merge against `main` before anything else runs; fails fast and clearly if there's a conflict, rather than surfacing as a confusing test failure later.
- **`ci`** — runs on every other trigger. Calls `setup-toolchain`, then `make test`, then the auto-detecting lint step described above.

**Secrets:** `CI_SUBMODULE_PAT` (required) — see [Private Dependency Access](#private-dependency-access-pat-based) below.

## `Reusable-RTL-CD.yml`

Triggered via `workflow_call` only — being invoked at all already means a caller explicitly wants a release, so there's no `event_name` guard on the job (an earlier version had one; it was removed because it silently skipped this job whenever called from a workflow that was itself triggered by something other than `workflow_dispatch`, e.g. `on-dependency-release.yml`'s `repository_dispatch`-triggered chain).

**Inputs:**

| Input | Default | Purpose |
|---|---|---|
| `tag_name` | auto-generated (`<repo>-<branch>-<date>-<sha>`) | Release tag |
| `build_output_path` | `generated_sv_dir` | Where generated RTL is packaged from |
| `rtl_target_override` | empty (uses the repo's own `cd.config`) | Force a specific `make` target instead |

**Secrets:** `CI_SUBMODULE_PAT` (required), `RELEASE_TOKEN` (optional — falls back to the automatic `GITHUB_TOKEN`; only needed if a repo should publish releases somewhere other than itself).

Aborts the release (rather than publishing an empty/broken one) if no `.sv` files are found in `build_output_path` after generation.

## `validate.yml`

Self-contained — deliberately does **not** call this repo's own reusable workflows, since it's the thing responsible for catching mistakes in them. Runs `actionlint` against every workflow file whenever one changes.

## `setup-toolchain/action.yml`

The shared composite action both CI and CD call. In order:

1. Configures git to rewrite `git@github.com:` submodule URLs to authenticated HTTPS, using the `submodule-token` input (i.e. `CI_SUBMODULE_PAT`) — not an SSH key/agent. `.gitmodules` itself never needs to change.
2. Checks out `playground` (the shared dependency/toolchain repo, defaults to `morphingmachines/playground`) and symlinks it as a sibling directory (`../playground`), matching what every project `Makefile` expects.
3. Fails fast with one clear error if any submodule uses an SSH URL but no token was provided — instead of letting every such submodule fail its own confusing retry loop.
4. Updates both `playground`'s own submodules and the calling repo's own `dependencies/` submodules — **pinned commit, one level deep only** (not `--remote`, not recursive).
5. Warns (non-blocking) if any submodule is behind its own remote branch's latest commit — informational only, doesn't change what gets built.
6. Installs Java 17, Mill, `firtool`, and a pinned build of `verilator` (each cached).
7. Resolves whichever `mill` launcher the calling repo actually expects — its own committed `./mill`, or the sibling `playground/mill` if it doesn't have one.

**Inputs:** `mill-version` (default `0.11.6`), `firtool-version` (default `1.62.1`), `verilator-version` (default `5.012` — must match exactly what `playground/Makefile.include`'s `check-verilator` expects; this check is exact-match, not `>=`), `playground-repo` (default `morphingmachines/playground`), `submodule-token`.

## Private Dependency Access (PAT-based)

Private `dependencies/*` submodules are accessed over authenticated HTTPS using a single org-wide PAT — not per-repo SSH deploy keys.

1. Create a PAT with **Contents: Read-only** access to every repo in the org.
2. Store it as an org-level Actions secret named exactly **`CI_SUBMODULE_PAT`**, with repository access set to **All repositories** (Org Settings → Secrets and variables → Actions — not any individual repo's own settings, and not your personal account's settings either — both are common ways this ends up invisible to a repo despite existing).
3. Every project repo's `RTL-CI.yml`/`RTL-CD.yml` forwards it automatically via `secrets: inherit` — nothing to configure per repo.

Project repos never reference this secret directly by name except in that one `secrets: inherit` line — it flows through to `Reusable-RTL-CI.yml`/`Reusable-RTL-CD.yml` → `setup-toolchain`'s `submodule-token` input automatically.

## Known Issues / Cleanup TODO

- **`playground.hash`** — at least one project repo (`TileLinkExplorer`) ships a `playground.hash` file, presumably intended to pin `playground` to a specific commit. `setup-toolchain/action.yml` does not currently read or use this file anywhere — `playground` is checked out at its live default-branch HEAD regardless. Either wire this in, or remove the file from project repos to avoid implying a guarantee that isn't actually enforced.

