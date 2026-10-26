# Graphene

Stacked pull requests for Git and GitHub. The executable is **`gt`**. An optional
shell shortcut is `alias g=gt`; packaging does not install `g` or `graphene`.

This guide targets **v0.6.0**, including stack reshaping, maintained stack comments, template-first PR
descriptions, operation progress, a read-only stack doctor and portable agent integration. Installation below selects that exact release.
Human pilot and independent coding-agent lifecycle readiness are verified
separately; publication alone does not establish those verdicts. Check
`gt --version` and `gt <command> --help` against the version you installed.

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
version=v0.6.0
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

## Coding agents

Install the shared workflow skill for your user account to use it across
repositories:

```sh
gt agents install --global --agent codex
gt agents status --global --agent codex
```

For Claude Code, replace `--agent codex` with `--agent claude`. Omit `--agent`
to install or check both agents. Global installation can run outside a repository.

For a project-specific installation, run these inside that repository instead:

```sh
gt agents install --agent codex
gt agents status --agent codex
```

Keep the same scope when managing the skill:

| Task | Example for a global Codex installation |
| --- | --- |
| Preview installation changes | `gt agents install --global --agent codex --dry-run` |
| Refresh after upgrading Graphene | `gt agents install --global --agent codex` |
| Check installed files and compatibility | `gt agents status --global --agent codex` |
| Remove the installed skill | `gt agents uninstall --global --agent codex` |

Omit `--global` for the repository installation. If status reports a potential
skill collision, review both scopes before choosing which copy to keep.

Start a fresh agent session after installing or refreshing the skill. Status
reports installation and compatibility; confirm that the agent discovers
Graphene in the fresh session. The same CLI and skill support both agents;
independent agent lifecycle validation is a separate readiness task.
Use `gt capabilities --json` and `gt inspect --json` for machine-readable state.

## First stack

Start in your Git repository on its trunk with a clean worktree and a normal
`origin` remote. Fetch/push must resolve to one identical URL; separate push
URLs are rejected by v0.6.0.

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

Before `sync` or `restack` rebases an adopted branch, Graphene retains its
verified remote head when it matches the local head. A later `submit` updates
the existing PR with an exact-head lease; a concurrent remote update rejects
the push. Branches that already diverged require scoped publication approval.
Offline restacking grants no new publication authority.

Graphene stores branch-parent relationships and replay boundaries locally in
`.git/graphene/state.json`. `log` works offline and shows cached PR numbers;
it does not fetch current GitHub status. Preserve this metadata if it fails
to load. A branch's parent relationship is explicit, rather than guessed
from Git history.

## Review feedback and navigation

```sh
gt co feature-a            # explicit checkout in v0.6.0
# Edit files for review feedback.
gt modify -a               # amend; automatically restack descendants locally
gt modify -a -c -m "More"  # alternatively add a new commit and restack
gt submit                  # publish changed heads and PR descriptions/bases
```

From a branch higher in the stack, `gt modify --into feature-a` applies your
staged changes to `feature-a` without checking it out; see
[Reshape a stack](#reshape-a-stack).

Choose the submit form that fits the work:

```sh
gt submit --draft                  # create new PRs as drafts
gt submit --edit                   # edit PR descriptions in your editor
gt submit --body-file description.md --json  # supply current branch text for automation
gt submit --body-for feature-a=description-a.md --body-for feature-b=description-b.md
```

These are alternatives. `--draft` applies to new PRs. File inputs replace the
selected PR's description text, so supply the complete intended description;
use repeated `--body-for branch=path` inputs for multiple layers. `--json` is
noninteractive, so agents should provide text through files instead of `--edit`.

Stack navigation is maintained in a Graphene comment, leaving your PR description
focused on the change. For new PRs in a terminal, Graphene opens an editor using
the repository PR template when available, or a generated description. Existing
manual descriptions are preserved unless you explicitly replace them. If you
cancel, Graphene retains the draft and prints its path for reuse with
`--body-file` or `--body-for`.

If submission reports that code was published but stack comments remain pending,
use the reported recovery action:

```sh
gt submit --comments-only  # refresh published stack navigation without pushing newer local commits
```

This mode does not create PRs or change their bases or manual descriptions.

In v0.6.0, `gt co` chooses among tracked branches in a terminal;
`gt co <branch>` also accepts trunk. `gt up` moves to the parent,
`gt down [child]` to a direct child, `gt top [leaf]` to a leaf above the current
branch, `gt bottom` to the stack root, and `gt trunk` to trunk. Fork choices
are sorted and include cached PR numbers. Ctrl-C cancels without checkout.
In scripts, supply explicit choices; a sole candidate is selected directly.

After changes made outside Graphene, such as a raw Git amend/rebase, run
`gt restack` to reconcile the current stack, then `gt submit` to publish.
Local operations do not push. In v0.6.0, default summaries
remain visible with `NO_COLOR` or redirected output; `--quiet` suppresses
mutation success/progress, while errors, `log`, and dry-run results remain.
Long operations report their current phase and completed work; interactive
terminals use colors and icons, while redirected output stays readable and
`--json` retains its machine-readable contract.

## Reshape a stack

New in v0.6.0. These commands change a stack's structure or commits locally and
restack whatever above the change needs it. None of them pushes; run `gt submit`
afterwards to publish.

```sh
gt modify --into feature-a -m "Address review"  # apply staged changes to a lower branch
gt squash                                        # collapse this branch's commits into one
gt split --by-file 'docs/*' --name docs -m "Document it"  # move files into a new branch below
gt split --by-commit                             # give each commit its own branch
gt create helper -a -m "Add helper" --insert     # add a branch between this one and its children
gt fold                                          # fold this branch into its parent
gt delete feature-b                              # delete a branch; its children move to its parent
```

Every reshape takes `--dry-run`, which lists each affected branch, its commits
and what happens to its PR, without changing anything. A conflict while
restacking stops for `gt continue`, or `gt abort`, which undoes the whole
reshape, including uncommitted and staged changes.

- `modify --into <branch>` applies the staged changes (or everything with `-a`)
  to a branch below the current one, amending its last commit or adding one
  with `-c -m`, then rebases everything above it. You stay where you are with
  nothing left staged. It needs Git 2.38 or later; staged changes that conflict
  with that branch are refused, and you can `gt co` it and `gt modify` there.
- `squash` keeps the first commit's message and author unless `-m` or `--edit`
  replaces the message.
- `split --by-file <pathspec>` works on a single-commit branch: matching files
  move into a new branch below it, named with `--name` and committed with `-m`.
  Pathspecs are relative to your current directory and may repeat. The current
  branch keeps its name, PR and children.
- `split --by-commit` gives each commit its own branch below the current one,
  asking for a name for each (in scripts, pass `--name` once per commit below
  the newest, oldest first; `--dry-run` previews suggested names). No commit
  is rewritten. The current branch keeps its newest commit, so its PR shrinks
  to that commit; fold branches back together with `gt fold`. Plain
  `gt split` asks which kind of split you want.
- `create <branch> --insert` commits the staged changes like `gt create`, then
  moves the current branch's children onto the new branch.
- `fold` moves the parent up to this branch's tip and removes this branch; its
  children move onto the parent. `--keep` keeps this branch and removes the
  parent instead. A branch on trunk is merged with `gt merge`, not folded.
- `delete [branch]` drops a branch's commits from its children and moves them
  onto its parent. `--upstack` deletes everything above it too; commits not in
  trunk need `--force` outside a terminal.

A removed branch's PR stays open on GitHub, and Graphene names it so you can
close it there. Retargeted, rewritten and new branches are published by the
next `gt submit`; retarget children before deleting a removed branch's remote
branch, or GitHub closes their PRs. Reshapes refuse branches checked out in
another worktree.

## Sync, move, and merge

After a bottom PR is squash-merged on GitHub:

```sh
gt sync       # fetch trunk; reconcile all tracked stacks; clear merged layers
              # and restack survivors locally
gt submit     # publish surviving heads, PR bases, and stack comments
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
commits. The following `gt submit` publishes those heads and refreshes stack comments.
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

Human output explains what happened and the next safe action in plain English.
Recovery choices describe their consequences and leave work pending by default.
Use `--json` for stable diagnostic codes, full recorded evidence and automation.
Choose the inspection command based on what you need to check:

```sh
gt inspect    # local checkout, stack relationships and recorded recovery state
gt doctor     # check all tracked stacks for problems without changing anything
gt recover    # reconcile unfinished remote work and review supported choices
```

Use `inspect` before an operation or when checking local state; it does not fetch
or check current GitHub status. Use `doctor` when something seems wrong but you
are unsure what: it works offline, never repairs, and exits 1 when it reports
findings. `gt doctor --remote` also reads GitHub using an already supplied
`GH_TOKEN` or `GITHUB_TOKEN`. Use `recover` when an earlier command reports
unfinished or uncertain remote work; it checks recorded targets against current
facts and offers recovery choices.

For a conflict, resolve files and stage them, then use `gt continue`.
Use `gt abort` to restore the interrupted local operation's checkpoint.
Prefer these over raw Git rebase recovery while a Graphene journal exists.

- Aborting a descendant restack after `modify` keeps the committed parent edit
  and restores descendants. Run `gt restack`, then `gt submit`. v0.6.0 also retains local input checkpoints under `refs/graphene/recovery/`.
- Move recovery restores branch topology and PR bases; a failed GitHub rollback
  retains its journal. Correct the underlying error before continuing/aborting.
- Completed pushes and GitHub merges remain completed after local abort.
  Inspect `gt log` and GitHub, finish local recovery, then sync/publish survivors.
- For partial or uncertain remote work, run `gt recover` (or `--json` for
  agents). Review original targets, confirmed/unknown effects and current facts.
  Available actions use `gt recover resume|replace|retire --approve <id>`;
  `gt submit --approve <id>` accepts applicable decisions inline. Terminal
  choices leave work pending by default. Merge/sync support unchanged resume.
- An unfinished submission blocks commands that change any stack, including
  `gt create` on trunk, and the error names its branches and PRs. If those PRs
  were merged or closed on GitHub meanwhile, `gt submit` cannot finish it; run
  `gt recover` and retire it.
- Direct Git amend/squash/rebase after successful submission remains ordinary
  work: restack descendants where needed, then submit. Exceptional adoption of
  rewritten externally published history requires scoped publication approval.
- Git publication uses atomic exact-head leases. PR edits use fresh best-effort
  checks; GitHub cannot guarantee protection against an edit between the check
  and PATCH. Graphene preserves unknown request evidence separately from a
  verified current result. An exact verified merge finishes cleanup without
  repeating that merge.
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
[public support issues](https://github.com/aitorvs/graphene-releases/issues). Include version,
OS/architecture, the command, observed error, and completed/remaining work.
Remove tokens and private repository details before sharing output.
