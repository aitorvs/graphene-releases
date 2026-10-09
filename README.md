# graphene

**Stacked pull requests for git and GitHub**

graphene (`gt`) helps you manage stacked git workflows — chains of dependent pull requests that land faster and review cleaner than monolithic changes.

## Installation

```bash
brew tap aitorvs/graphene
brew install graphene
```

After installation, the `gt` command will be available.

## Quick Start

```bash
# Track your trunk branch
gt init

# Create a new branch stacked on current
gt create feature-a -m "Add API endpoint" -a

# Create another branch stacked on feature-a
gt create feature-b -m "Add tests for endpoint" -a

# Submit both PRs (each targets its parent)
gt submit

# See your stack
gt log
```

## Tutorial: Building a Stacked Workflow

Let's walk through a realistic example: adding a new API endpoint to a web application, then adding tests for it in a separate PR.

### Setup

Start in a git repository with a clean working tree on your trunk branch (usually `main`):

```bash
git checkout main
git pull
gt init
```

`gt init` tells graphene which branch is your trunk. It creates `.git/graphene/state.json` to track your stacks.

### Step 1: Create the first branch

Add the API endpoint:

```bash
# Make some changes (e.g., add routes, controllers)
gt create add-user-endpoint -m "Add POST /api/users endpoint" -a
```

This:
- Creates branch `add-user-endpoint` stacked on `main`
- Stages all changes (`-a` flag)
- Commits them with your message
- Graphene now tracks this branch

### Step 2: Stack a second branch on top

Without going back to `main`, create the tests:

```bash
# Make test changes
gt create add-user-tests -m "Add tests for POST /api/users" -a
```

Now you have:
```
main
└── add-user-endpoint
    └── add-user-tests  (current)
```

Check your stack:
```bash
gt log
```

Output:
```
main
└── add-user-endpoint (PR #123)
    └── add-user-tests (current)
```

### Step 3: Submit both PRs

```bash
gt submit
```

This creates two PRs:
- **PR #123**: `add-user-endpoint` targeting `main`
- **PR #124**: `add-user-tests` targeting `add-user-endpoint`

Reviewers can review each PR independently. The tests PR shows only test changes (not the endpoint implementation).

### Step 4: Make changes to the bottom branch

Reviewer feedback on PR #123: "Add input validation." Make the change:

```bash
# Go back to the first branch
gt down

# Edit files, then commit
gt modify -m "Add input validation" -a
```

`gt modify` amends the branch and **automatically restacks** `add-user-tests` on top. Both PRs update on GitHub.

### Step 5: Merge the stack

After PR #123 is approved and merged:

```bash
gt sync
```

`gt sync`:
- Fetches from GitHub
- Detects PR #123 was merged
- Deletes `add-user-endpoint` locally
- Restacks `add-user-tests` directly onto `main`
- Updates PR #124 to target `main`

Now you can merge PR #124.

## Navigation Commands

Move through your stack without typing branch names:

```bash
gt up          # Move to parent branch
gt down        # Move to child branch (picks if multiple)
gt top         # Jump to top of stack
gt bottom      # Jump to bottom/root of stack
gt trunk       # Jump to trunk branch
```

## Command Reference

Run `gt --help` to see all commands, or `gt <command> --help` for details on a specific command.

Key commands:
- `gt init` — Set up graphene in a repository
- `gt create <branch>` — Create a new stacked branch
- `gt modify` — Amend current branch and restack
- `gt submit` — Create/update pull requests
- `gt sync` — Fetch, clean up merged branches, restack
- `gt log` — Show your stack tree
- `gt merge` — Merge PRs in your stack (bottom-up or cascade)

## Troubleshooting

Having issues? [File an issue](https://github.com/aitorvs/graphene/issues) or check existing reports.

---

## About

graphene is a CLI tool for managing stacked git workflows with GitHub. It tracks branch relationships, handles restacking automatically, and keeps your PRs in sync with GitHub.

**Early Access**: graphene is currently in early access. Feedback welcome!

**Note:** This repository contains prebuilt binaries and documentation. Source code is maintained separately.
