# jj VCS Agent Skill

An Agent Skill that helps AI coding agents work with the Jujutsu (jj) version control system.

## Overview

This repository provides an Agent Skill for Claude Code and other compatible AI agents to use the Jujutsu VCS. It covers non-interactive atomic commit refinement, scoped fetch/push, divergence and operation recovery, workspace management, and stack-wide checks.

## Compatibility

**Target version:** jj v0.46.0

The documented workflows target jj v0.46.0, including child-workspace Git colocation and `jj workspace remove` to delete a workspace and its directory. jj v0.46.0 requires Git 2.42.0 or newer. Commands and defaults may differ in other versions; see the [official release notes](https://github.com/jj-vcs/jj/blob/v0.46.0/CHANGELOG.md).

## What is Jujutsu?

Jujutsu (jj) is a Git-compatible version control system that offers several advantages:

- **Working copy as a commit**: Most commands automatically snapshot on-disk changes; passive inspection can explicitly skip snapshotting
- **No staging area**: Changes are moved directly between commits using `squash` and `split`
- **Automatic rebasing**: Descendant commits are automatically rebased when parent commits change
- **Mutable commits**: Mutable history can be edited, split, and squashed; trunk, tags, and other configured immutable history are protected
- **Change IDs**: Stable identifiers that persist across commit rewrites
- **Conflict handling**: Conflicts can be committed and resolved later

## Installation

If you use [just](just.systems), you can run:

```
just install
```

Copy the `jujutsu/` directory into your project's `.claude/skills/` directory:

```bash
cp -r jujutsu/ /path/to/your/project/.claude/skills/jujutsu/
```

Or install it as a global skill:

```bash
cp -r jujutsu/ ~/.claude/skills/jujutsu/
```

## Skill Contents

```
jujutsu/
└── SKILL.md    # Main skill file with jj workflow instructions
```

## Key Workflow Philosophy

The skill emphasizes:

1. **Describe-first commits**: Use `jj describe -m "message"` before making changes
2. **Atomic commits**: Each commit should represent one logical change
3. **Commit quality preservation**: Leverage jj's mutability to refine commits
4. **Non-interactive refinement**: Use message-specified squashes and file-selected splits to maintain readable history
5. **Scoped publishing and recovery**: Select intended bookmarks/remotes, preview pushes, respect immutability, and inspect operations before recovery
6. **Stack verification**: Review patches oldest first and use `jj run --ignore-changes` for checks without rewriting commits

## Contributing

Contributions are welcome. Please ensure any changes are compatible with jj v0.46.0.

## License

MIT
