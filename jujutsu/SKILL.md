---
name: jujutsu
description: "**REQUIRED** - Always activate FIRST on any git/VCS operations (commit, status, branch, push, etc.), especially when HEAD is detached. If `.jj/` exists -> this is a Jujutsu (jj) repo - raw git commands can corrupt data. Essential git safety instructions inside. DO NOT IGNORE."
allowed-tools: Bash(jj *)
---

# Jujutsu (jj) Version Control System

This skill helps you work with Jujutsu, a Git-compatible VCS with mutable commits and automatic rebasing.

**Targets jj v0.46.0** - Commands may differ in other versions.

## Important: Automated/Agent Environment

Use canonical command names such as `status`, `describe`, and `operation`: built-in aliases can be disabled by configuration.

1. **Disable the pager** on commands that produce output:

```bash
jj --no-pager log
jj --no-pager diff --git
jj --no-pager show <change-id>
```

2. **Supply descriptions with `-m`** instead of opening an editor:

```bash
jj describe -m "message"
jj squash -m "Combined description"
```

`--no-pager` does not disable editors or interactive UIs. Use the non-interactive split and convergence workflows below; reserve editor-based commands for interactive use.

3. **Verify the changed state**, not just the exit code. After mutations, inspect `jj --no-pager status` and the relevant history or operation diff. Successful commands can be no-ops.

4. **For passive/background inspection**, read recorded state without snapshotting current files:

```bash
jj --no-pager --at-op=@ --ignore-working-copy log
```

This deliberately omits unsnapshotted edits. Use ordinary commands for reviewing current on-disk changes. `--no-integrate-operation` still creates operations and permits external effects, including pushes; it is not a dry-run.

### Configuration

Repo/workspace configuration lives outside the repository. Discover its location with `jj config path --repo` or `--workspace`; use `jj config set`/`unset` for requested non-interactive changes rather than `config edit`.

To modify a specific loaded configuration file, use `jj config set --file <path> <key> <value>` (or `unset --file`). `--user` targets the first loaded user file, which later files can override. Global `--config-file` loads additional settings for an invocation; it does not select a file to modify. Consult the [configuration docs](https://docs.jj-vcs.dev/v0.46.0/config/) for locations and precedence.

## Core Concepts

### The Working Copy is a Commit

In jj, your working directory is always a commit (referenced as `@`). Most jj commands automatically snapshot on-disk changes unless snapshotting is explicitly disabled. There is no staging area.

There is no need to run `jj commit` to finalize changes — the working copy is always a commit. Use `jj commit <paths>` only to commit a subset of files while keeping the rest in the working copy (see "Splitting Commits").

### Commits Are Mutable

Mutable commits can be described, squashed, rebased, and absorbed after creation. By default, trunk, local tag targets, untracked remote bookmark/tag targets, and their ancestors are immutable. Inspect the [immutability rules](https://docs.jj-vcs.dev/v0.46.0/config/#set-of-immutable-commits) when an edit is rejected; use `--ignore-immutable` only when rewriting protected history is explicitly intended.

### Change IDs vs Commit IDs

- **Change ID**: A stable identifier (like `tqpwlqmp`) that persists when a commit is rewritten — prefer these when referencing commits
- **Commit ID**: A content hash (like `3ccf7581`) that changes when commit content changes

### Revsets

jj uses a revset language to select commits in commands. Common revsets:

- `@` — the working copy commit
- `@-` — the parent of the working copy
- `::@` — all ancestors of `@`
- `@::` — all descendants of `@`
- `trunk()..@` — commits between trunk and `@` (your branch)
- `bookmarks()` — all commits with bookmarks
- `divergent()` — visible commits with multiple versions of the same change
- `change_id(<id>)` — all visible versions of a change, including divergent versions

Use revsets with `-r` flags: `jj --no-pager log -r 'trunk()..@'`

## Essential Workflow

### Starting Work: Describe First, Then Code

**Always create your commit message before writing code:**

Validate that you're on a blank revision with `jj --no-pager status`. If you are not, create one:

```bash
jj new
```

```bash
# First, describe what you intend to do
jj describe -m "Add user authentication to login endpoint"

# Then make your changes - they automatically become part of this commit
# ... edit files ...

# Check status
jj --no-pager status
```

### Creating Atomic Commits

Each commit should represent ONE logical change. Use this format for commit messages:

```
Examples:
- "Add validation to user input forms"
- "Fix null pointer in payment processor"
- "Remove deprecated API endpoints"
- "Update dependencies to latest versions"
```

### Viewing History

```bash
# View recent commits
jj --no-pager log

# View with patches
jj --no-pager log -p

# View specific commit
jj --no-pager show <change-id>

# Review each revision's patch oldest first
jj --no-pager show --git --reversed -r 'trunk()..@'

# View diff of working copy (use --git for familiar +/- format)
jj --no-pager diff --git
```

Use `jj diff --git` for standard unified diffs with `+`/`-` lines. Other configured diff formats can look different without indicating corruption. `show -r <revset>` displays individual patches; `diff -r <revset>` aggregates the selected changes. For content-history searches, consult the [revset functions](https://docs.jj-vcs.dev/v0.46.0/revsets/#functions) and [historical file search](https://docs.jj-vcs.dev/v0.46.0/cli-reference/#jj-file-search).

### Moving Between Commits

```bash
# Create a new empty commit on top of current
jj new

# Create new commit with message
jj new -m "Commit message"

# Edit an existing commit (working copy becomes that commit)
jj edit <change-id>

# Edit the previous commit
jj prev -e

# Edit the next commit
jj next -e
```

## Refining Commits

### Squashing Changes

Move changes from the current commit into its parent with an explicit combined description:

```bash
jj squash -m "Combined description"
```

Use `jj squash --use-destination-message` only when deliberately retaining the destination description and discarding source descriptions. Bare `jj squash` can open an editor when both descriptions are nonempty. `-i`, `--tool`, and `--editor` require interactive tooling.

### Splitting Commits

For a non-interactive whole-file split, provide both filesets and `-m`. This works on the working-copy revision or an existing ancestor:

```bash
jj split -r @- -m "Extract validation" path/to/file1 path/to/file2
```

Without placement flags, selected changes stay in the original change and remaining changes become a new child. The selected part receives the supplied description; the other part keeps its original description. Bare `jj split` opens interactive tooling; avoid `-i`, `--tool`, and `--editor` in agent workflows.

For selected files from the working copy, `jj commit` is a convenient alternative:

```bash
jj commit -m "Extract validation" path/to/file1 path/to/file2
```

Only the given paths go into the described commit; other changes remain in a new working-copy commit.

`jj restore <paths>` discards changes to those paths; it does not move them into another commit. Use `split` or `commit` when preserving both parts.

### Absorbing Changes

Automatically distribute changes to the commits that last modified those lines:

```bash
# Absorb working copy changes into appropriate ancestor commits
jj absorb
```

### Abandoning Commits

Remove a commit entirely (descendants are rebased to its parent):

```bash
jj abandon <change-id>
```

### Undoing Operations

Undo acts on the last recorded operation, not necessarily the previous shell command. Inspect the operation log before recovery; it identifies the originating workspace:

```bash
jj --no-pager --at-op=@ --ignore-working-copy operation log --limit 10
```

- `jj undo` reverses the latest operation; repeated calls walk backward.
- `jj redo` walks forward through undone operations.
- `jj operation revert <operation-id>` reverses a selected earlier operation rather than restoring the entire repository to that point.
- `jj revert` creates inverse commit changes; use it when reverting published changes instead of undoing repository operations or rewriting protected history.

`undo`/`redo` refuse operations from another workspace by default. Run them in the originating workspace; use `--allow-cross-workspace` only for intentional cross-workspace recovery. These commands do not reverse external effects such as a remote push.

Operation diffs can filter revisions. For a full audit, use `jj --no-pager operation show <operation-id> --show-changes-in 'all()' --git`. Consult the [operation-log docs](https://docs.jj-vcs.dev/v0.46.0/operation-log/) for targeted recovery and restoration.

### Rebasing Commits

Move commits to different parents while preserving their changes:

```bash
# Rebase the current branch
jj rebase --onto <destination>

# Move only the selected revision
jj rebase -r <change-id> --onto <destination>

# Move a revision and all its descendants
jj rebase -s <change-id> --onto <destination>

# Move the current branch onto the configured trunk
jj rebase --onto 'trunk()'
```

`-r` excludes descendants from the move, but they can still be rewritten onto the selected revision's former parents. Inspect the resulting graph, especially when moving ancestor commits.

### Restoring Files

Discard changes to specific files or restore files from another revision:

```bash
# Discard working-copy changes relative to the parent
jj restore

# Discard changes to specific files
jj restore path/to/file.txt

# Restore files from a specific revision
jj restore --from <change-id> path/to/file.txt
```

## Working with Bookmarks (Branches)

Bookmarks are jj's equivalent to git branches. They follow rewrites of their target, but do not automatically advance when you create new commits.

```bash
# Create a bookmark at current commit
jj bookmark create my-feature -r@

# Move bookmark to a different commit
jj bookmark move my-feature --to <change-id>

# Advance a named ancestor bookmark to the chosen descendant
jj bookmark advance exact:my-feature --to @

# List bookmarks
jj --no-pager bookmark list

# Delete a bookmark
jj bookmark delete my-feature
```

Use `--to @-` instead when the finished revision is the parent of an empty working-copy commit. Bare `bookmark advance` uses configurable selection and target revsets and can move multiple closest bookmarks; name the intended bookmark and target explicitly. Keep `bookmark move` for intentional backward or sideways moves.

Local bookmarks follow remote updates only when the corresponding remote bookmarks are tracked. Track or untrack a specific name and remote:

```bash
jj --no-pager bookmark list --all-remotes
jj bookmark track exact:my-feature --remote exact:origin
jj bookmark untrack exact:my-feature --remote exact:origin
```

Untracking keeps the local bookmark but stops future fetched updates from propagating into it. Separate name/remote patterns scope the operation; literal `my-feature@origin` symbols also remain supported. Consult the [bookmark tracking docs](https://docs.jj-vcs.dev/v0.46.0/bookmarks/#remotes-and-tracked-bookmarks) for other policies.

## Workspaces

A **workspace** is a working copy plus its associated repo. One repo can have multiple workspaces — each with its own working directory and working-copy commit (`@`) — all sharing the same commits, operations, and bookmarks. This is jj's equivalent of `git worktree`.

Useful for running a long build or test in one workspace while editing in another. Workspaces are a rarely-needed feature; consult the [official docs](https://docs.jj-vcs.dev/v0.46.0/working-copy/#workspaces) for anything beyond the basics below.

### Common commands

```bash
# Create a new workspace (defaults: name = basename of path, parent = current @'s parent)
jj workspace add ../my-tests
jj workspace add --name tests --no-colocate -r <change-id> ../my-tests   # jj-only, explicit name and base
jj workspace add --colocate ../git-tests   # create a corresponding Git worktree

# Inspect
jj --no-pager workspace list
jj --no-pager workspace list -T 'name ++ "\t" ++ root ++ "\n"'   # names and recorded roots
jj workspace root                         # current workspace
jj workspace root --name tests            # named workspace

# Inspect or toggle Git colocation for the current workspace
jj --no-pager git colocation status
jj git colocation enable
jj git colocation disable

# Remove a named workspace and its directory (run from another workspace)
jj workspace remove <name>

# Rename current workspace
jj workspace rename <new-name>
```

In `jj log`, each workspace's `@` appears as `<workspace-name>@`.

### Removing workspaces

When asked to remove a workspace, use `jj workspace remove <name>` so its directory is deleted too. Run it from another workspace in the same repo. The main workspace cannot be removed.

Before removal, inspect `jj --no-pager workspace list` and `jj workspace root --name <name>` to confirm the target and record its path. Stop processes using that workspace and preserve any ignored files you need: tracked changes are snapshotted into a commit before removal, but ignored files are deleted without being saved.

After removal, verify that the workspace is no longer listed and its recorded directory is gone. Do not substitute `jj workspace forget` plus manual directory deletion.

Use `jj workspace forget <name>` only when explicitly keeping the directory or clearing registration for a missing/inaccessible workspace. It leaves the workspace directory on disk; for a colocated workspace, it also unregisters the corresponding Git worktree.

### Key semantics

- **Isolation by default.** `jj workspace add` gives the new workspace its own fresh empty commit; workspaces don't start out sharing `@`, and on-disk files are never live-mirrored between them.
- **Colocation.** `workspace add` creates a Git worktree by default only when the current workspace is colocated and `git.colocate` is `true`. Use `--colocate` or `--no-colocate` to choose explicitly. Git 2.42.0 or newer is required. `git colocation status` includes the workspace name; `enable`/`disable` also work in child workspaces, creating/removing their Git worktree.
- **Recorded roots.** `workspace list` and `root` show recorded paths even when unreachable; `root` emits a warning. A displayed path does not prove the directory exists. Workspaces created before jj 0.38.0 may lack a recorded root.
- **List template labels.** Color rules for workspace fields use labels such as `workspace name` and `workspace root`.
- **Propagation at command boundaries.** Ordinary jj commands snapshot the current workspace's files and read the op log, so they see commits/bookmarks made by other workspaces. Passive inspection can skip snapshotting as described above. There is no filesystem watcher.
- **Stale working copy.** If another workspace rewrites this workspace's `@` (e.g. via `jj squash`, `rebase`, `abandon`), jj refuses commands here until you run `jj workspace update-stale`. Same recovery path if a command was interrupted mid-update.
- **Shared `@` is sharp-edged.** `jj edit <id>` lets two workspaces point at the same change without warning. When one mutates it, the other goes stale; if the stale one had unsnapshotted edits, `update-stale` preserves them as a **divergent commit** with the same change ID. Use [divergence recovery](#resolving-divergence) below. Avoid sharing `@` unless both workspaces are read-only.

### Agent guidance

- Always pass `--no-pager` to `jj workspace list`.
- Don't `jj edit` a change another workspace already has as its `@` — main cause of accidental divergence.
- For undo/redo or targeted recovery, follow [Undoing Operations](#undoing-operations) and identify the originating workspace first.

## Git Integration

### Working with Existing Git Repos

```bash
# Clone a git repository
jj git clone <url>

# Initialize jj in an existing git repo
jj git init --colocate
```

### Fetching Remote Changes

Fetch configured bookmarks and tags from configured/default remote(s):

```bash
jj git fetch

# Fetch a specific bookmark from an explicit remote
jj git fetch --remote exact:origin --branch exact:main
jj --no-pager bookmark list --all-remotes exact:main
```

`git.fetch` selects default remotes; `remotes.<name>.fetch-bookmarks` and `fetch-tags` control default targets. A successful fetch does not prove that every branch was fetched or that a same-named local bookmark moved; see bookmark tracking above.

Fetched tags are tracked by same-named local tags by default. Untracking a tag does not stop fetching it or delete it. To disable tag fetching intentionally, use `remotes.<name>.fetch-tags = '~*'` in jj config, not Git's `tagOpt`. See the [fetch configuration](https://docs.jj-vcs.dev/v0.46.0/config/#default-bookmarks-and-tags-to-fetch).

Fetch can recognize rewritten remote commits when their change IDs survive, hide obsolete versions, and rebase mutable local descendants onto successors. Inspect `jj --no-pager log -r 'trunk()..@'` and status afterward. If the branch still needs moving onto updated trunk, use `jj rebase --onto 'trunk()'`; an additional rebase is not always necessary.

### Switching Between jj and git (Colocated Repos Only)

**This section only applies to colocated repos** (where both `.jj/` and `.git/` exist). In non-colocated repos, do not use git commands — they will corrupt jj state.

In a colocated repository, you can use both jj and git commands with care:

**Switching to git mode** (e.g., for merge workflows):
```bash
# First, ensure your jj working copy is clean
jj --no-pager status

# Then checkout a branch with git
git checkout <branch-name>
```

**Switching back to jj mode**:
```bash
# Use jj edit to resume working with jj
jj edit <change-id>
```

**Important notes:**
- Git may complain about uncommitted changes if jj's working copy differs from the git HEAD
- ALWAYS ensure your work is committed in jj before switching to git
- After git operations, jj will detect and incorporate the changes on next command

### Pushing Changes

Push only when the user explicitly requests it, with the intended bookmark and remote selected.

1. Inspect `jj --no-pager bookmark list --all-remotes exact:my-feature` and choose the finished target revision. If `@` is an empty working-copy child, the finished target may be `@-`.
2. Move an existing bookmark with `jj bookmark move exact:my-feature --to <change-id>`, or create a missing one with `jj bookmark create my-feature -r <change-id>`.
3. Review the target's content and description; keep commits atomic.
4. Preview the remote changes, then execute the authorized push:

```bash
jj --no-pager git push --dry-run --remote exact:origin --bookmark exact:my-feature
jj git push --remote exact:origin --bookmark exact:my-feature
```

Explicitly pushing a bookmark that tracks no remote automatically tracks it. If it already tracks another remote, creating it on a new endpoint requires intentional tracking there first, e.g. `jj bookmark track my-feature@backup`. `exact:` selects a literal name; unqualified selectors use glob patterns.

Plain push can include tracked tags in its default revision set. `--all`, `--tracked`, and `-r` can select tags as well as bookmarks; preserve explicit bookmark scoping for ordinary agent pushes. Broad pushes can skip references with private or conflicted commits, so inspect the output rather than assuming every selected reference was pushed.

`git.push` can select multiple remotes through patterns or arrays, and `--remote` is repeatable. `revsets.git-push` can change the default revision selection. Select one explicit remote for the ordinary workflow; choose multiple endpoints only intentionally. Consult the [push reference](https://docs.jj-vcs.dev/v0.46.0/cli-reference/#jj-git-push) for tag publication and specialized overrides.

## Handling Conflicts

jj allows committing conflicts — you can resolve them later:

```bash
# View conflicts
jj --no-pager status
```

**Agent conflict resolution**: Edit conflicted files directly to replace conflict markers with resolved content, then run `jj --no-pager status` to verify resolution. Reserve `jj resolve` for configured non-interactive merge tools or interactive use.

### Resolving Divergence

Divergence means multiple visible versions of one change, not necessarily a file conflict. Inspect the versions first; if combining them is intended, use scoped, non-interactive convergence:

```bash
jj --no-pager log -r 'divergent()'
jj --no-pager show --git -r 'change_id(<change-id>)'
jj converge --no-interactive -r 'change_id(<change-id>)'
jj --no-pager operation show -p --git
jj --no-pager log -r 'change_id(<change-id>)'
jj --no-pager status
```

Replace `<change-id>` with an unambiguous prefix from the log. A bare divergent change ID is ambiguous; `change_id(...)` selects all visible versions, while an offset such as `<id>/0` selects only one.

Convergence can rebase descendants, move local bookmarks, and introduce file conflicts. Review the resulting versions and content, not just the exit code. If heuristics require a decision, `--no-interactive` fails rather than prompting; choose an explicit recovery using the [divergence guide](https://docs.jj-vcs.dev/v0.46.0/guides/divergence/) instead of retrying interactively. Abandon a version by commit ID only after establishing it is unwanted.

## Preserving Commit Quality

**IMPORTANT**: Because commits are mutable, always refine them before considering work done:

1. **Review your commit**: `jj --no-pager show @` or `jj --no-pager diff --git`
2. **Is it atomic?** One logical change per commit
3. **Is the message clear?** Use imperative verb phrase in sentence case format with no full stop: e.g. "Add login endpoint", "Fix null pointer in payment processor", "Remove deprecated API endpoints"
4. **Are there unrelated changes?** Use `jj commit -m "Selected change" <paths>` for working-copy files or the non-interactive split workflow for existing revisions
5. **Should changes be elsewhere?** Use the squashing workflow above or `jj absorb`

### Running Checks Across a Stack

`jj run` executes a command in isolated per-revision working copies. By default, successful file edits rewrite the selected revisions and descendants are rebased.

For checks, use an explicit revset, `--ignore-changes` to discard command edits, and `--root` to run from each checkout's root. In 0.46, sparse patterns are inherited from the current workspace; use `--sparse-patterns full` when the check needs the entire tree.

```bash
# Cargo example; substitute the project's existing non-interactive check command
jj --no-pager run -r 'trunk()..@' --root --ignore-changes --sparse-patterns full -- cargo test
```

For intentional transforms, omit `--ignore-changes` and review `jj --no-pager operation show -p --git` plus status afterward. Keep normal failure handling for verification: `--ignore-errors` continues after failures and does not make them affect the exit code of `jj run`. See the [run reference](https://docs.jj-vcs.dev/v0.46.0/cli-reference/#jj-run) for concurrency, reused working copies, and clean runs.

### Specialized Workflows

Use version-pinned references when the task needs these less-common branches:

- [Tag management](https://docs.jj-vcs.dev/v0.46.0/cli-reference/#jj-tag) for tag lifecycle, tracking, or explicitly requested publication.
- [File editing](https://docs.jj-vcs.dev/v0.46.0/cli-reference/#jj-file-edit) and [deletion](https://docs.jj-vcs.dev/v0.46.0/cli-reference/#jj-file-delete) for changing files in another revision without switching `@`; `file edit` opens an editor.
- [Bisect](https://docs.jj-vcs.dev/v0.46.0/cli-reference/#jj-bisect-run) for fault localization: use a dedicated workspace and retain endpoint checks.
- [Fix configuration](https://docs.jj-vcs.dev/v0.46.0/config/#enforce-code-formatting-on-modified-lines) for configured per-file transforms; current line-range arguments use the `line-range-args` array.

## Quick Reference

| Action | Command |
|--------|---------|
| Describe commit | `jj describe -m "message"` |
| View status | `jj --no-pager status` |
| View log | `jj --no-pager log` |
| View diff | `jj --no-pager diff --git` |
| Review stack oldest first | `jj --no-pager show --git --reversed -r 'trunk()..@'` |
| New commit | `jj new -m "message"` (check status first; skip if `@` is empty) |
| Edit commit | `jj edit <id>` |
| Squash to parent | `jj squash -m "Combined description"` |
| Split selected files | `jj split -r <id> -m "Selected change" <paths>` |
| Auto-distribute | `jj absorb` |
| Rebase | `jj rebase --onto <destination>` |
| Abandon commit | `jj abandon <id>` |
| Undo last operation | `jj undo` |
| Redo undone operation | `jj redo` |
| Revert selected operation | `jj operation revert <operation-id>` |
| Converge one divergent change | `jj converge --no-interactive -r 'change_id(<id>)'` |
| Restore files | `jj restore [paths]` |
| Create bookmark | `jj bookmark create <name>` |
| Advance named bookmark | `jj bookmark advance exact:<name> --to <id>` |
| Fetch remote | `jj git fetch` |
| Preview bookmark push | `jj --no-pager git push --dry-run --remote exact:<remote> --bookmark exact:<name>` |
| Push requested bookmark | `jj git push --remote exact:<remote> --bookmark exact:<name>` |
| Add workspace | `jj workspace add <path>` |
| List workspaces | `jj --no-pager workspace list` |
| Remove workspace and directory | `jj workspace remove <name>` |
| Fix stale working copy | `jj workspace update-stale` |

## Best Practices Summary

1. **Describe first**: Set the commit message before coding
2. **One change per commit**: Keep commits atomic and focused
3. **Use change IDs**: They're stable across rewrites
4. **Refine commits**: Leverage mutability for clean history
5. **Embrace the workflow**: No staging area, no stashing - just commits
