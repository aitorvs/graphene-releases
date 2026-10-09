# Graphene

Stacked pull requests for Git and GitHub. The executable is **`gt`**. An optional
shell shortcut is `alias g=gt`; packaging does not install `g` or `graphene`.

This guide describes the next pilot version built from the P0/P1 improvements.
The currently published **v0.2.0** has the core workflow, but does not include
those recovery safeguards or explicit `co/down/top` arguments. Installation
below matches v0.2.0 exactly; pilot readiness for a newer release must be
verified separately before inviting users. Check `gt --version` and
`gt <command> --help` against the version you installed.

## Install and upgrade

Install [GitHub CLI](https://cli.github.com/) and authenticate with
`gh auth login`. Graphene uses that authentication, or `GH_TOKEN`.
Remove another tool that owns `gt`, such as Graphite, before installation.

On macOS:

```sh
brew tap aitorvs/graphene
brew install --cask aitorvs/graphene/graphene
gt --version
```

For an existing formula installation, migrate once:

```sh
brew update
brew uninstall --formula graphene
brew install --cask aitorvs/graphene/graphene
```

Later upgrades use `brew update` followed by
`brew upgrade --cask aitorvs/graphene/graphene`.

On Linux (x86_64 or ARM64), install the exact release archive and verify its
SHA-256 checksum before replacing your user-local executable. Run this block
in a shell; repeat it with a newly published tag to upgrade:

```sh
set -eu
version=v0.2.0
case "$(uname -m)" in
  x86_64) arch=amd64 ;;
  aarch64|arm64) arch=arm64 ;;
  *) echo 'Unsupported architecture' >&2; exit 1 ;;
esac
asset="gt-linux-$arch.tar.gz"
work=$(mktemp -d)
url="https://github.com/aitorvs/graphene-releases/releases/download/$version"
curl -fL "$url/$asset" -o "$work/$asset"
curl -fL "$url/checksums.txt" -o "$work/checksums.txt"
(cd "$work"; awk -v file="$asset" '$2 == file {print}' checksums.txt > selected.sha256
 test -s selected.sha256
 sha256sum -c selected.sha256)
tar -xzf "$work/$asset" -C "$work" gt
mkdir -p "$HOME/.local/bin"
install -m 755 "$work/gt" "$HOME/.local/bin/gt"
rm -rf "$work"
export PATH="$HOME/.local/bin:$PATH"
gt --version
```

Add `$HOME/.local/bin` to PATH in your shell startup file to keep that setting.
Release assets and checksums are public at
[Graphene releases](https://github.com/aitorvs/graphene-releases/releases).

## First stack

Start in your Git repository on its trunk with a clean worktree and a normal
`origin` remote. Fetch/push must resolve to one identical URL; separate push
URLs are rejected by the next pilot version.

```sh
gh auth status
gt --version
git switch main                  # use develop, etc. if that is your trunk
git fetch origin
git remote set-head origin -a    # refresh origin's default branch when needed
gt init
# Edit the first layer's files.
gt create feature-a -a -m "Add feature"
# Edit the second layer's files.
gt create feature-b -a -m "Test feature"
gt log
gt submit
```

`init` detects trunk from `origin/HEAD`; without it, the current branch is the
fallback, so switch to the correct trunk first. Trunk need not be `main`.
`gt init --import-graphite` imports existing Graphite metadata. Adopt existing
Git branches bottom-up with `gt track feature-a --parent main`, then
`gt track feature-b --parent feature-a`.

Graphene stores branch-parent relationships and replay boundaries locally in
`.git/graphene/state.json`. `log` works offline and shows cached PR numbers;
it does not fetch current GitHub status. Preserve this metadata if it fails
to load. A branch's parent relationship is explicit, rather than guessed
from Git history.

## Review feedback and navigation

```sh
gt co feature-a            # explicit checkout in the next pilot version
# Edit files for review feedback.
gt modify -a               # amend; automatically restack descendants locally
gt modify -a -c -m "More"  # alternatively add a new commit and restack
gt submit                  # publish changed heads and PR descriptions/bases
```

For v0.2.0, use `git switch feature-a` for an explicit checkout. In the next
pilot version, `gt co` chooses among tracked branches in a terminal;
`gt co <branch>` also accepts trunk. `gt up` moves to the parent,
`gt down [child]` to a direct child, `gt top [leaf]` to a leaf above the current
branch, `gt bottom` to the stack root, and `gt trunk` to trunk. Fork choices
are sorted and include cached PR numbers. Ctrl-C cancels without checkout.
In scripts, supply explicit choices; a sole candidate is selected directly.

After changes made outside Graphene, such as a raw Git amend/rebase, run
`gt restack` to reconcile the current stack, then `gt submit` to publish.
Local operations do not push. In the next pilot version, default summaries
remain visible with `NO_COLOR` or redirected output; `--quiet` suppresses
mutation success/progress, while errors, `log`, and dry-run results remain.

## Sync, move, and merge

After a bottom PR is squash-merged on GitHub:

```sh
gt sync       # fetch trunk; reconcile all tracked stacks; clear merged layers
              # and restack survivors locally
gt submit     # publish surviving heads, PR bases, and stack maps
```

Sync from trunk reconciles all tracked stacks too. It does not pick a feature
branch for you; inspect `gt log` and use `gt co <branch>` afterward. A merged
current branch may require another checkout during cleanup. Unpublished local
work is preserved when it is not contained in the verified merged PR head.

```sh
gt move feature-b feature-a --dry-run
gt move feature-b feature-a              # move branch plus descendants
gt move feature-b main --branch-only    # move only that branch
gt submit
```

Move requires a clean worktree, a tracked branch, and trunk or a tracked new
parent. It rejects cycles and same-parent moves. It updates existing PR bases, preserving PR text, but does not push rewritten
commits. The following `gt submit` publishes those heads and refreshes stack maps.
With `--branch-only`, direct children stay on the old parent with unchanged
refs; check out a former child and run `gt restack` to remove inherited commits
from the moved branch, then publish with `gt submit`.

```sh
gt merge                  # squash-merge bottom-up, one commit per PR
gt merge --to feature-a   # stop after this layer
```

Merge publishes/restacks surviving heads and checks each exact head before the
next layer. It can stop while fresh CI runs; wait for checks/reviews, then
rerun from a surviving branch. Completed GitHub merges are irreversible by
local recovery. `--cascade` merges top-down into parents to produce one final
trunk commit; bottom-up is the normal daily workflow.

## Recover safely

For a conflict, resolve files and stage them, then use `gt continue`.
Use `gt abort` to restore the interrupted local operation's checkpoint.
Prefer these over raw Git rebase recovery while a Graphene journal exists.

- Aborting a descendant restack after `modify` keeps the committed parent edit
  and restores descendants. Run `gt restack`, then `gt submit`. The next pilot
  version also retains local input checkpoints under `refs/graphene/recovery/`.
- Move recovery restores branch topology and PR bases; a failed GitHub rollback
  retains its journal. Correct the underlying error before continuing/aborting.
- Completed pushes and GitHub merges remain completed after local abort.
  Inspect `gt log` and GitHub, finish local recovery, then sync/publish survivors.
- A partial or uncertain submit retains a checkpoint in the next pilot version.
  Rerun `gt submit` with the same inputs to reconcile accepted remote effects.
  Use `--retry-create` only after independently establishing that no PR was
  created, including closed PRs; an empty open-PR lookup is insufficient.
- An unexpected remote-head/lease change stops publication. Inspect concurrent
  work; do not force-push to bypass it. Restore the intended single origin URL
  if repository identity changed.
- Timeouts can mean a remote write succeeded. Inspect/reconcile before retrying.
  API calls default to 30 seconds; `GRAPHENE_API_TIMEOUT=60s` adjusts the limit.
- For access errors, check the named path's ownership/access and disk space.
  For corrupt/newer metadata, preserve the bytes, check `gt --version`, and
  upgrade or seek help. Do not delete state or reinitialize to hide the error.
- If another command holds the lock, wait. A lock file remaining on disk does
  not mean the lock is stale; deleting it can bypass mutual exclusion.

## Help and feedback

Run `gt --help` or `gt <command> --help`. Report problems through
[public support issues](https://github.com/aitorvs/graphene-releases/issues),
which do not require access to the private source repository. Include version,
OS/architecture, the command, observed error, and completed/remaining work.
Remove tokens and private repository details before sharing output.

## Maintainers

In the source checkout, `./release.sh vMAJOR.MINOR.PATCH --check` validates
without publication. Running it without `--check` publishes a release and
updates Homebrew. Release validation uses readonly dependency checks, race
tests, vet, a CGO-disabled build, and disposable workflow checks. A failed
publication retains its tag; inspect the public release before retrying.
The source README is the canonical public guide; mirror it to the public
release repository through a reviewed PR when instructions change.
