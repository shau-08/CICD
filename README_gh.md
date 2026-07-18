# gh CLI reference for this CI/CD workflow

Commands grouped by where they fit in the actual lifecycle you built, starting
from zero.

## 1. Auth (one-time)

```bash
gh auth login                          # interactive, opens browser
gh auth login --with-token < token.txt # non-interactive, using a PAT
gh auth status                         # confirm who you're logged in as, and scopes
gh auth refresh -s repo,workflow       # add scopes to an existing login if a command complains about permissions
```

## 2. Org secrets (setting up / checking CI_SUBMODULE_PAT)

```bash
# Create or update an org-level secret, visible to all repos
gh secret set CI_SUBMODULE_PAT --org morphingmachines --visibility all < token.txt

# Same, but restricted to specific repos instead of all
gh secret set CI_SUBMODULE_PAT --org morphingmachines --visibility selected \
  --repos mmu,RRM,network_interface,RedefineIp

# List secrets (names only, never values) at org level
gh secret list --org morphingmachines

# List secrets visible to one specific repo (confirms org secret is actually reaching it)
gh secret list -R morphingmachines/mmu

# Delete a secret
gh secret delete CI_SUBMODULE_PAT --org morphingmachines
```

## 3. Triggering CI / CD manually

CI (`ci.yml`) fires automatically on push/PR — nothing to run by hand. CD
(`cd.yml`) is `workflow_dispatch`, so this is the actual "cut a release"
command:

```bash
# Untagged run -- release only, no notify
gh workflow run cd.yml -R morphingmachines/mmu

# Explicitly tagged run -- release AND notifies RedefineIp
gh workflow run cd.yml -R morphingmachines/mmu -f tag_name=RTL1p2

# List available workflows in a repo (to confirm the exact filename/ID to target)
gh workflow list -R morphingmachines/mmu

# Trigger by numeric workflow ID instead of filename, if two workflows share a name
gh workflow run 12345678 -R morphingmachines/mmu -f tag_name=RTL1p2
```

## 4. Watching / inspecting runs

```bash
# List recent runs for a repo
gh run list -R morphingmachines/mmu

# List runs for one specific workflow only
gh run list -R morphingmachines/mmu -w cd.yml

# Watch the most recent run live (auto-refreshes until it finishes)
gh run watch -R morphingmachines/mmu

# View a specific run's summary
gh run view <run-id> -R morphingmachines/mmu

# View full logs for a run (equivalent of clicking through every step in the UI)
gh run view <run-id> -R morphingmachines/mmu --log

# View logs for just the failed step(s) -- fastest way to debug
gh run view <run-id> -R morphingmachines/mmu --log-failed

# Re-run a failed run
gh run rerun <run-id> -R morphingmachines/mmu

# Re-run only the jobs that failed, not the whole thing
gh run rerun <run-id> -R morphingmachines/mmu --failed
```

## 5. Checking the trigger chain fired correctly

There's no `gh` subcommand for `repository_dispatch` events directly, but you
can confirm the chain worked by checking RedefineIp's own runs right after a
tagged dependency release:

```bash
gh run list -R morphingmachines/RedefineIp -w on-dependency-release.yml
gh run view <run-id> -R morphingmachines/RedefineIp --log
```

## 6. Releases

```bash
gh release list -R morphingmachines/mmu
gh release list -R morphingmachines/mmu --limit 50

gh release view RTL1p2 -R morphingmachines/mmu
gh release view RTL1p2 -R morphingmachines/mmu --json tagName,createdAt,assets

gh release download RTL1p2 -R morphingmachines/mmu          # download assets
gh release download RTL1p2 -R morphingmachines/mmu -D ./out # into a specific dir

# Manual release (rarely needed here since cd.yml does this automatically)
gh release create v1.0 ./generated_sv_dir.tar.gz -R morphingmachines/mmu \
  --title "v1.0" --notes "Manual release"

# Delete a release and its tag (what your cleanup workflow does automatically)
gh release delete RTL1p2 -R morphingmachines/mmu --cleanup-tag -y
```

## 7. Submodules / repo state (plain git, not gh, but you'll want these alongside)

```bash
# See what commit RedefineIp currently has pinned for each dependency
git -C RedefineIp submodule status

# See if a specific submodule is behind its tracked branch (same check CI does)
git -C RedefineIp/mmu ls-remote origin refs/heads/main
git -C RedefineIp/mmu rev-parse HEAD
```

## 8. Debugging repo/workflow visibility issues (the PAT/secret scoping problem from earlier)

```bash
# Confirm a repo is really under the org (not a personal fork)
gh repo view morphingmachines/mmu --json owner,name

# Confirm which repos an org secret is scoped to (no direct `gh` view for this --
# check via web UI, or via the API directly)
gh api orgs/morphingmachines/actions/secrets/CI_SUBMODULE_PAT

# List repos an org secret is selectively scoped to, if visibility=selected
gh api orgs/morphingmachines/actions/secrets/CI_SUBMODULE_PAT/repositories
```

## Typical end-to-end sequence for cutting a real release

```bash
gh workflow run ci.yml -R morphingmachines/mmu           # not usually needed manually, CI is automatic
gh run watch -R morphingmachines/mmu                     # confirm CI is green first

gh workflow run cd.yml -R morphingmachines/mmu -f tag_name=RTL1p2
gh run watch -R morphingmachines/mmu

gh release view RTL1p2 -R morphingmachines/mmu           # confirm mmu's own release published

gh run list -R morphingmachines/RedefineIp -w on-dependency-release.yml   # confirm the trigger fired
gh run view <run-id> -R morphingmachines/RedefineIp --log

gh release list -R morphingmachines/RedefineIp           # confirm mmu-main-RTL1p2 shows up
```
