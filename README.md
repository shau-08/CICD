# CICD

Central, shared CI/CD engine for Chisel/RTL project repos across the organization. This repo contains the actual CI/CD logic once; project repos each carry only a few thin "caller" files that reference it — no CI/CD logic is duplicated per project.

## Structure

```
.github/
├── actions/
│   └── setup-toolchain/
│       └── action.yml        Shared toolchain setup (composite action)
└── workflows/
    ├── reusable-ci.yml       CI: merge-check, tests, lint
    ├── reusable-cd.yml       CD: RTL generation + GitHub Release
    └── validate.yml          Self-check: lints this repo's own workflow files
```

## How a Project Repo Plugs Into This

A project repo needs three small files under its own `.github/workflows/`:

```yaml
# ci.yml
on: [push, pull_request]
jobs:
  ci:
    uses: YOUR_ORG/CICD/.github/workflows/reusable-ci.yml@main
    secrets: inherit
```

```yaml
# cd.yml
on: workflow_dispatch
jobs:
  cd:
    permissions:
      contents: write
    uses: YOUR_ORG/CICD/.github/workflows/reusable-cd.yml@main
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

For `reusable-ci.yml` / `reusable-cd.yml` to work against it, a project repo needs:

- A `Makefile` with, at minimum, a `test` target. A `lint` target is optional — CI probes for it (`make -n lint`) and skips cleanly if absent.
- A `Makefile` with an `rtl-dispatch` target (or pass `rtl_target_override` as a CD input to bypass it) for CD to generate RTL.
- A `build.sc` with exactly one top-level module whose `millSourcePath` is `os.pwd` — the lint step auto-detects the project name from this convention; more than one, or none, and it fails with a clear error rather than guessing.
- Optionally, a `cd.config` file (`RTL_TARGET=rtl` or `RTL_TARGET=lazyrtl`, plus a matching `TARGET=...`) to control CD without needing workflow inputs.

## `reusable-ci.yml`

Triggered via `workflow_call`. Two jobs:

- **`check-mergeable`** — PRs targeting `main` only. Attempts a real merge against `main` before anything else runs; fails fast and clearly if there's a conflict, rather than surfacing as a confusing test failure later.
- **`ci`** — runs on every other trigger. Calls `setup-toolchain`, then `make test`, then the auto-detecting lint step described above.

**Secrets:** `CI_SSH_PRIVATE_KEY` (optional) — see [SSH Access](#ssh-access-for-private-dependencies) below.

## `reusable-cd.yml`

Triggered via `workflow_call`, but the underlying job only actually runs `if: github.event_name == 'workflow_dispatch'` — CD never fires automatically.

**Inputs:**

| Input | Default | Purpose |
|---|---|---|
| `tag_name` | auto-generated (`<repo>-<branch>-<date>-<sha>`) | Release tag |
| `build_output_path` | `generated_sv_dir` | Where generated RTL is packaged from |
| `rtl_target_override` | empty (uses the repo's own `cd.config`) | Force a specific `make` target instead |

**Secrets:** `CI_SSH_PRIVATE_KEY` (optional), `RELEASE_TOKEN` (optional — falls back to the automatic `GITHUB_TOKEN`; only needed if a repo should publish releases somewhere other than itself).

Aborts the release (rather than publishing an empty/broken one) if no `.sv` files are found in `build_output_path` after generation.

## `validate.yml`

Self-contained — deliberately does **not** call this repo's own reusable workflows, since it's the thing responsible for catching mistakes in them. Runs `actionlint` against every workflow file whenever one changes.

## `setup-toolchain/action.yml`

The shared composite action both CI and CD call. In order:

1. Configures SSH (if a key was provided) via `webfactory/ssh-agent`.
2. Checks out `playground` (the shared dependency/toolchain repo) and symlinks it as a sibling directory (`../playground`), matching what most project `Makefile`s expect.
3. Fails fast with one clear error if any submodule uses an SSH URL but no key was provided — instead of letting every such submodule fail its own confusing retry loop.
4. Updates both `playground`'s own submodules and the calling repo's own `dependencies/` submodules — **pinned commit, one level deep only** (not `--remote`, not recursive).
5. Warns (non-blocking) if any submodule is behind its own remote branch's latest commit — informational only, doesn't change what gets built.
6. Installs Java 17, Mill, `firtool`, and a pinned build of `verilator` (each cached; `verilator` in particular is slow to build from source, so its cache is also kept warm by a weekly scheduled workflow in project repos — see `warm-verilator-cache.yml`).
7. Resolves whichever `mill` launcher the calling repo actually expects — its own committed `./mill`, or the sibling `playground/mill` if it doesn't have one.

**Inputs:** `mill-version` (default `0.11.6`), `firtool-version` (default `1.62.1`), `verilator-version` (default `5.012` — must match exactly what `playground/Makefile.include`'s `check-verilator` expects; confirmed via a real CI failure that this check is exact-match, not `>=`), `ssh-private-key`.

## SSH Access for Private Dependencies

Several dependencies (`chipyard`, `diplomacy`, `emitrtl`, `rocket-chip`, `rocket-chip-fpga-shells`, `rocket-chip-inclusive-cache`, `redefine-header`, and potentially others as new project repos are added) are referenced via SSH URLs, some of which are private.

1. Generate a keypair: `ssh-keygen -t ed25519 -C "ci-key" -f ./ci_deploy_key -N ""`
2. Add the **public** key to an account with read access to those repos — a dedicated bot/machine account is recommended over a personal one, since an account-level key grants access to every repo that account can read (a per-repo deploy key would need to be repeated for each private dependency individually).
3. Add the **private** key as a secret named exactly `CI_SSH_PRIVATE_KEY`, either on this repo or org-wide (`gh secret set CI_SSH_PRIVATE_KEY --org YOUR_ORG --visibility all < ci_deploy_key`) so every project repo inherits it via `secrets: inherit`.

Project repos never reference this secret directly — it flows through `secrets: inherit` → the reusable workflow → `setup-toolchain`'s `ssh-private-key` input automatically.

## Known Issues / Cleanup TODO

- **`playground.hash`** — at least one project repo (`TileLinkExplorer`) ships a `playground.hash` file, presumably intended to pin `playground` to a specific commit. `setup-toolchain/action.yml` does not currently read or use this file anywhere — `playground` is checked out at its live default-branch HEAD regardless. Either wire this in, or remove the file from project repos to avoid implying a guarantee that isn't actually enforced.

