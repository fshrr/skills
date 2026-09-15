---
name: using-worktrunk
description: Use when starting feature work that needs an isolated workspace, when creating, listing, switching, or removing git worktrees, or whenever EnterWorktree, isolation:worktree, --worktree, .claude/worktrees, .worktrees/, or a raw git worktree add looks like the next step. Applies to every repo on this machine.
---

# Using Worktrunk

## Overview

On this machine worktrees are created with **worktrunk (`wt`)**, never with the harness's native tool or raw git. This is a standing user directive (2026-07-20) and overrides the superpowers `using-git-worktrees` skill's "prefer native tools" step.

**Core principle:** `wt switch -c` is the only way a worktree gets created. Everything else is a violation, even if it looks faster.

**Announce at start:** "Using using-worktrunk to set up an isolated workspace."

## Why wt and not native

Native tools drop the worktree in `.claude/worktrees/`, skip the project's post-start hooks (gitignored env-file copy, dependency install), and produce a workspace the user's shell integration and `wt list` cannot see. Every past "worktree missing .env.local" bug came from exactly that.

## Step 0: Layout gate (do not skip)

wt's global path template is `{{ repo_path }}/../{{ branch | sanitize }}`. It assumes the **container layout**: repo checkout at `<project>/main`, worktrees as siblings `<project>/<branch-sanitized>`.

```bash
basename "$(git rev-parse --show-toplevel)"   # must print: main
```

| Result | Action |
|---|---|
| `main` | Container layout. Proceed to Step 1. |
| anything else | **STOP.** Creating a worktree now dumps it beside the repo's parent (e.g. `~/Projects/active/feat-foo`). Ask the user which fix they want, then apply it before Step 1. If the user has said not to ask questions, apply fix 2 (reversible, no directory moves), say so in one line, and continue. |

Fixes for a flat repo (user picks):
1. **Migrate to container layout** (preferred): `mv <repo> <tmp> && mkdir <repo> && mv <tmp> <repo>/main`. Then update anything that hard-codes the old path (IDE workspaces, `.claude/settings.local.json` allowlists, cron, launchd).
2. **Per-project path override** in `~/.config/worktrunk/config.toml` (user config, not project config): 
   ```toml
   [projects."github.com/<owner>/<repo>"]
   worktree-path = "{{ repo_path }}/../{{ repo }}.{{ branch | sanitize }}"
   ```
   Get the identifier from `wt config show` (PROJECT CONFIG → Identifier).

Never proceed with "I'll flag the odd path and continue." Wrong-path worktrees are the mess this skill exists to prevent. "Don't ask me questions" is not permission to dump the worktree in the wrong place; it selects fix 2.

## Step 1: Create

```bash
wt switch -c <branch> --yes            # from the default branch
wt switch -c <branch> -b <base> --yes  # from another base
```

- `--yes` required: non-interactive sessions cannot answer the hook-approval prompt.
- Worktree lands at `<project>/<branch-sanitized>` (`feat/foo` → `feat-foo/`).
- Bash cwd does not persist between calls. Use absolute paths into the new worktree for every later command.

## Step 2: Verify hooks did their job

Project hooks live in `.config/wt.toml` (`[[post-start]]`) and `.worktreeinclude` (which gitignored files to copy). Check, don't assume:

```bash
git worktree list | grep <branch-sanitized>
ls <worktree>/<each path listed in .worktreeinclude>
ls <worktree>/node_modules | head -1     # or the equivalent for the stack
```

- No `.config/wt.toml` in the repo → hooks did not run. Copy gitignored env files and install deps by hand, then propose adding `.config/wt.toml` + `.worktreeinclude` so next time is automatic.
- Do not start a dev server as part of setup. Some repos allow one server repo-wide.

## Step 3: Finish

The user merges PRs on GitHub. So: push branch, open PR, and once merged prune:

```bash
wt remove <branch> --yes     # deletes worktree AND branch
```

`wt merge` (squash into main + cleanup) exists but is only for when the user says merge locally.

## Quick reference

| Task | Command |
|---|---|
| Create + branch | `wt switch -c <branch> --yes` |
| Switch existing | `wt switch <branch>` |
| List all | `wt list` |
| Prune merged | `wt remove <branch> --yes` |
| See resolved config / project id | `wt config show` |
| Run hooks manually | `wt hook post-start` |

## Rationalizations

| Excuse | Reality |
|---|---|
| "EnterWorktree is the native tool, superpowers says prefer native" | User directive beats skill default. Native tool is the banned path. |
| "`git worktree add` is one command, wt is overkill" | Skips hooks. That is the env-file bug, again. |
| "User didn't mention wt" | Rule applies even when unmentioned. |
| "Layout is wrong but I'll flag it and continue" | Wrong-path worktree = cleanup job for the user. Stop and ask. |
| "I'll `isolation: worktree` on the subagent, it's just a subagent" | Same tool, same ban. Create with wt, hand the subagent the path. |
| "Hooks ran, no need to check" | Hook approval can silently fail without `--yes`. Verify. |
| "User said 'EnterWorktree is fine', that's the exception" | Permission is not an override. The user set this rule so they would not have to think about it each time. Use wt, tell them in one line. |
| "User said don't ask, so I can't run the layout gate" | The gate has a no-questions default (fix 2). Run it. |

## Red flags — STOP

- About to call EnterWorktree, pass `isolation: "worktree"`, or run `git worktree add`
- A path containing `.claude/worktrees` or `.worktrees/`
- `basename` of the repo root is not `main` and you are still creating
- Told the user "worktree ready" without listing the env files

**Only exception:** the user explicitly overrides this skill for the task at hand, e.g. "use EnterWorktree instead of wt" or "skip worktrunk this time". A passing "EnterWorktree is fine" or "whatever is fastest" is not an override.
