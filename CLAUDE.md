# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Git Conventions

Full workflow: the `/git` skill (`~/.claude/skills/git/SKILL.md`), enforced mechanically by
the `~/.claude/hooks/git-guard.sh` PreToolUse hook. These rules apply to every Minerest
repository; the Oboi tree is separate and uses `/oboi-git`.

**Everything is written in English** — commit subjects and bodies, branch slugs, PR titles and
PR bodies. When the work is described in Portuguese, translate it; do not transcribe it. The
hook blocks Portuguese commit subjects and warns on Portuguese PR titles.

**Trunk**: `master`. Every PR targets it. Never commit to it directly — branch first.

### Commits

Conventional Commits with a **module scope**. Header max 72 chars, lowercase first letter, no
trailing period, imperative mood.

```
fix(samurai): apply damage on every slash

Why the change was needed; the diff already shows what changed.

Refs REST-263
```

- **Types**: `feat` `fix` `hotfix` `docs` `style` `refactor` `perf` `test` `build` `ci`
  `chore` `revert` `release`. Never `wip`.
- **Scope** is the module that changed, never the task ID:
  `bom`, `build`, `deps`.

  Shared across all repos: `build`, `ci`, `deps`, `docs`, `test`, `repo`.
- **Task reference goes in the footer**, not the scope — `Refs REST-1234` for a Linear task,
  `Closes #26` for a GitHub issue, omitted when neither exists.
- **`REST-0000` is banned.** It exists only because an earlier rule demanded a task ID where
  none existed. A module scope always exists, so nothing needs to be invented.
- No `Co-Authored-By` trailer, no AI attribution, anywhere.

### Branches

```
<type>/<TASK-ID>/<slug>    fix/REST-263/samurai-slash-damage
<type>/gh-<issue>/<slug>   fix/gh-26/block-ghost-abilities
<type>/<slug>              fix/samurai-slash-damage
```

Slug is lowercase kebab-case, 2-4 words. Both `REST-XXXX` (Linear) and `gh-NN` (GitHub issue)
are valid references; never fabricate one.

Auto-generated branch names — `claude/*` from cloud sessions, `worktree-*` and
adjective-triples like `abstract-purring-twilight` from `EnterWorktree` — **must be renamed
before pushing**:

```bash
git branch -m "$(git rev-parse --abbrev-ref HEAD)" "<type>/<slug>"
```

Rename before the PR exists: renaming a branch with an open PR auto-closes it on GitHub.

### Staging and pushing

- Stage specific files. Never `git add -A` or `git add .`.
- Never force push the trunk; prefer `--force-with-lease` on feature branches.
- Never `--no-verify`.
