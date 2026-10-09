---
name: jujutsu
description: "REQUIRED before any version-control operation, especially Git/jj commands or detached HEAD. If .jj exists, use jj workflows and Git-colocation safeguards."
allowed-tools: Bash(jj *)
---

# Jujutsu (jj) Version Control System

**Targets jj v0.46.0** - Commands may differ in other versions.

## Agent execution

- Use canonical command names: built-in aliases can be disabled.
- Pass `--no-pager` for output and use explicit description flags (`-m` or the destination-message squash form). A pager flag does not disable editors or interactive UIs; use the non-interactive forms below.
- After mutations, inspect status and the relevant history or operation diff. Success can mean no changes.
- For passive inspection, use `jj --no-pager --at-op=@ --ignore-working-copy log`. It reads recorded state, omitting unsnapshotted edits; use ordinary commands to review current files.
- `--no-integrate-operation` still creates operations and permits external effects, including pushes. It is not a dry-run.

## Core model

- `@` is a commit. Ordinary jj commands snapshot on-disk changes; there is no staging area or required finalization step.
- Mutable commits can be rewritten. Default immutable heads are trunk, local tags, and untracked remote bookmarks/tags; their ancestors are protected too. Inspect the [immutability rules](https://docs.jj-vcs.dev/v0.46.0/config/#set-of-immutable-commits) before using `--ignore-immutable`, and override only for an explicitly intended protected-history rewrite.
- Prefer stable **change IDs**. **Commit IDs** change when a commit is rewritten.

Revsets select commits with `-r`:

| Revset | Selects |
|--------|---------|
| `@` / `@-` | Working-copy commit / its parent |
| `::@` / `@::` | Ancestors / descendants of `@` |
| `trunk()..@` | Branch commits beyond trunk |
| `bookmarks()` | Local bookmark targets |
| `divergent()` | Multiple visible versions of a change |
| `change_id(<id>)` | All visible versions of that change |

## Workflow

For file changes:

1. Inspect status. Reuse a blank `@`; otherwise run `jj new` to preserve the current revision.
2. Describe the intended change before editing: `jj describe -m "Add login validation"`.
3. Edit files.
4. Before finishing, review the change with `jj --no-pager show @` and confirm:
   - The diff contains one intended logical change.
   - The description is an imperative, sentence-case phrase without a full stop.
   - Unrelated changes are separated using [splitting](#splitting-commits); misplaced edits are squashed or absorbed into the appropriate change.

## Command reference

Replace placeholders before running commands. For pattern-taking bookmark/remote options, `exact:` selects a literal name; unqualified selectors use globs.

| Action | Command |
|--------|---------|
| Status | `jj --no-pager status` |
| History / history with patches | `jj --no-pager log` / `jj --no-pager log -p` |
| Inspect a revision | `jj --no-pager show --git <id>` |
| Working-copy unified diff | `jj --no-pager diff --git` |
| Review stack patches oldest first | `jj --no-pager show --git --reversed -r 'trunk()..@'` |
| New empty child / with description | `jj new` / `jj new -m "<description>"` |
| Edit a revision | `jj edit <id>` |
| Edit previous / next revision | `jj prev -e` / `jj next -e` |
| Squash into parent with combined description | `jj squash -m "<combined-description>"` |
| Squash, retaining destination description and discarding source descriptions | `jj squash --use-destination-message` |
| Absorb edits into mutable ancestors that last changed those lines | `jj absorb` |
| Abandon a commit, rebasing descendants to its parents | `jj abandon <id>` |
| Discard all / selected working-copy edits | `jj restore` / `jj restore <paths>` |
| Restore paths from a revision | `jj restore --from <id> <paths>` |
| Create / move bookmark | `jj bookmark create <name> -r <id>` / `jj bookmark move exact:<name> --to <id>` |
| Advance a named bookmark to a chosen descendant | `jj bookmark advance exact:<name> --to <id>` |
| List bookmarks / include all remote states | `jj --no-pager bookmark list` / `jj --no-pager bookmark list --all-remotes` |
| Delete bookmark | `jj bookmark delete exact:<name>` |
| Clone / initialize existing Git checkout | `jj git clone <url>` / `jj git init --colocate` |

Bare `squash` can open an editor when both descriptions are nonempty. `restore` discards current changes; it does not move them into another commit.

Other diff formats can look different without indicating corruption. `show -r <revset>` displays individual patches; `diff -r <revset>` aggregates changes. For content-history queries, consult [revset functions](https://docs.jj-vcs.dev/v0.46.0/revsets/#functions) or [file search](https://docs.jj-vcs.dev/v0.46.0/cli-reference/#jj-file-search).

## Refining Commits

### Splitting Commits

For a non-interactive whole-file split of `@` or an ancestor, supply filesets and `-m`; omit `-i`, `--tool`, and `--editor`:

```bash
jj split -r @- -m "<selected-description>" path/to/file1 path/to/file2
```

Without placement flags, selected changes retain the original change ID and receive the supplied description. Remaining changes become a new child with the original description.

For selected working-copy files, use `jj commit -m "<description>" <paths>` instead; unselected changes remain in a new working-copy commit.

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

## Bookmarks

Bookmarks follow target rewrites, not newly created commits. Bare `bookmark advance` uses configurable selection/target revsets and can move several closest bookmarks; specify the name and target. Backward or sideways moves require an intentional `bookmark move --allow-backwards`.

Track a particular remote bookmark to propagate future fetches into the same-named local bookmark:

```bash
jj bookmark track exact:my-feature --remote exact:origin
jj bookmark untrack exact:my-feature --remote exact:origin
```

Untracking keeps the local bookmark but stops that propagation. Separate name/remote patterns scope the operation; literal `my-feature@origin` symbols also work. See [tracking policies](https://docs.jj-vcs.dev/v0.46.0/bookmarks/#remotes-and-tracked-bookmarks).

## Workspaces

A **workspace** is a working copy plus its associated repo. One repo can have multiple workspaces — each with its own working directory and working-copy commit (`@`) — all sharing the same commits, operations, and bookmarks. This is jj's equivalent of `git worktree`.

For workspace operations beyond the commands below, consult the [official docs](https://docs.jj-vcs.dev/v0.46.0/working-copy/#workspaces).

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
- **Propagation.** Commands read the shared operation log and see other workspaces' commits/bookmarks at command boundaries; there is no filesystem watcher.
- **Stale working copy.** If another workspace rewrites this workspace's `@` (e.g. via `jj squash`, `rebase`, `abandon`), jj refuses commands here until you run `jj workspace update-stale`. Same recovery path if a command was interrupted mid-update.
- **Shared `@` is sharp-edged.** `jj edit <id>` lets two workspaces point at the same change without warning. When one mutates it, the other goes stale; if the stale one had unsnapshotted edits, `update-stale` preserves them as a **divergent commit** with the same change ID. Use [divergence recovery](#resolving-divergence) below. Avoid sharing `@` unless both workspaces are read-only.

For operation recovery, follow [Undoing Operations](#undoing-operations).

## Git Integration

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
3. Review the target's content and description against the [workflow](#workflow) completion criteria.
4. Preview the remote changes, then execute the authorized push:

```bash
jj --no-pager git push --dry-run --remote exact:origin --bookmark exact:my-feature
jj git push --remote exact:origin --bookmark exact:my-feature
```

Explicitly pushing a bookmark that tracks no remote automatically tracks it. If it already tracks another remote, creating it on a new endpoint requires intentional tracking there first, e.g. `jj bookmark track my-feature@backup`.

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

Convergence can rebase descendants, move local bookmarks, and introduce file conflicts. Confirm one visible version remains and review its content and affected descendants/bookmarks. If heuristics require a decision, `--no-interactive` fails rather than prompting; choose explicit recovery using the [divergence guide](https://docs.jj-vcs.dev/v0.46.0/guides/divergence/). Abandon a version by commit ID only after establishing it is unwanted.

## Running Checks Across a Stack

`jj run` executes a command in isolated per-revision working copies. By default, successful file edits rewrite the selected revisions and descendants are rebased.

For checks, use an explicit revset, `--ignore-changes` to discard command edits, and `--root` to run from each checkout's root. In 0.46, sparse patterns are inherited from the current workspace; use `--sparse-patterns full` when the check needs the entire tree.

```bash
# Cargo example; substitute the project's existing non-interactive check command
jj --no-pager run -r 'trunk()..@' --root --ignore-changes --sparse-patterns full -- cargo test
```

For intentional transforms, omit `--ignore-changes` and review `jj --no-pager operation show -p --git` plus status afterward. Keep normal failure handling for verification: `--ignore-errors` continues after failures and does not make them affect the exit code of `jj run`. See the [run reference](https://docs.jj-vcs.dev/v0.46.0/cli-reference/#jj-run) for concurrency, reused working copies, and clean runs.

## Configuration

Repo/workspace config lives outside the repository; find it with `jj config path --repo` / `--workspace`. For requested changes, use `config set`/`unset` rather than an editor.

`jj config set --file <path> <key> <value>` (or `unset --file`) modifies a specific loaded file. `--user` modifies the first user file, which later files can override. Global `--config-file` loads additional settings for that invocation; it does not select a file to modify. See [configuration locations and precedence](https://docs.jj-vcs.dev/v0.46.0/config/).

## Specialized workflows

Use version-pinned references when the task needs these less-common branches:

- [Tag management](https://docs.jj-vcs.dev/v0.46.0/cli-reference/#jj-tag) for tag lifecycle, tracking, or explicitly requested publication.
- [File editing](https://docs.jj-vcs.dev/v0.46.0/cli-reference/#jj-file-edit) and [deletion](https://docs.jj-vcs.dev/v0.46.0/cli-reference/#jj-file-delete) for changing files in another revision without switching `@`; `file edit` opens an editor.
- [Bisect](https://docs.jj-vcs.dev/v0.46.0/cli-reference/#jj-bisect-run) for fault localization: use a dedicated workspace and retain endpoint checks.
- [Fix configuration](https://docs.jj-vcs.dev/v0.46.0/config/#enforce-code-formatting-on-modified-lines) for configured per-file transforms; current line-range arguments use the `line-range-args` array.

