# commit

A Claude Code skill that stages and commits your current changes as **one coherent commit**, with a message written in the repository's own style.

The skill itself is in [SKILL.md](SKILL.md). This README is for people; Claude only reads `SKILL.md`.

## What it does

1. **Learns the repo's conventions.** It reads the last 30 commit subjects and checks for commitlint, a commit template, commit hooks and `CONTRIBUTING.md`/`CLAUDE.md`. If the log uses Conventional Commits (`feat(auth): ...`), so does the message. If the log is prose (`Fixed the X`), the message uses prose in the same tense. An empty repo gets Conventional Commits.
2. **Reads the full diff** before writing anything, and leaves anything you already staged as it is.
3. **Draws the commit boundary.** If the tree holds unrelated changes, it proposes a split instead of bundling them.
4. **Stages by path.** It never runs `git add -A` or `git add .` as a reflex. Before committing it checks the staged diff for:
   - secrets
   - debug leftovers (`console.log`, `debugger`, `.only(`)
   - accidental files (`.DS_Store`, build output)
   - a `package.json` change without its lockfile
5. **Writes the message.** The subject is under 72 characters with no trailing period. A body, wrapped at 72 columns, is added only when the commit needs a *why*.
6. **Commits and reports** the subject, the file count, and anything left uncommitted.

### Defaults

| Question | Default |
|---|---|
| Push after committing? | No |
| Open a PR? | No |
| Co-author / generated-by trailer? | Never (no `Co-Authored-By:` line or email), even if Claude Code's own attribution setting asks for one |
| Amend instead of a new commit? | Only when you say "amend" |
| `--no-verify`? | Never. A failed hook is reported, not skipped. |

A project's `CLAUDE.md`, its `CONTRIBUTING.md` or an instruction in your message overrides any of these defaults.

## Installing it in a project

Pick one option.

### Project-level (shared with your team)

```bash
cd /path/to/your-project
```

```bash
mkdir -p .claude/skills/commit
```

```bash
curl -fsSL https://raw.githubusercontent.com/sarliram/claude-skills/main/commit/SKILL.md -o .claude/skills/commit/SKILL.md
```

Or copy it from a local clone:

```bash
cp -R /path/to/claude-skills/commit .claude/skills/
```

### Personal (all your projects)

```bash
mkdir -p ~/.claude/skills && cp -R /path/to/claude-skills/commit ~/.claude/skills/
```

### Symlink (updates with `git pull`)

```bash
ln -s /path/to/claude-skills/commit ~/.claude/skills/commit
```

### Claude apps (claude.ai / desktop)

```bash
cd /path/to/claude-skills && zip -r commit.zip commit
```

Upload the zip under Settings → Capabilities → Skills.

## Invoking the skill

The skill triggers automatically on requests like these:

- `commit this`
- `commit my changes`
- `make a commit for the OTP fix`
- `split this into commits`
- `commit and push`

Or call it by name:

```
/commit
```

### Variants

| Say | What happens |
|---|---|
| `commit and push` | Commits, then runs `git push -u origin HEAD` |
| `amend` | Amends `HEAD`, but only after checking it isn't on a remote |
| `split this into commits` | Proposes the groups for your approval, then makes one commit per group |
| `commit everything` | Stages broadly, but still reads the diff and still refuses secrets |

## Permissions

The frontmatter's `allowed-tools` pre-approves the following, so the skill runs without a permission prompt for each step:

- read-only git commands
- `git add`, `git commit`, `git stash` and `git reset`
- Read, Grep and Glob

It doesn't pre-approve `git push`, so a push still asks for your permission.
