---
name: commit
description: Stage and commit the current changes as one coherent commit, with a message that matches the repository's own commit style. Use this WHENEVER the user asks to commit, "commit this", "commit my changes", "save this to git", "make a commit for X", or asks to split work into commits. Reads the repo's recent log and lint config to decide between Conventional Commits and the project's prose style rather than imposing either, stages only the files belonging to the change, keeps the subject under 72 characters, and never pushes, amends published history, or bypasses hooks unless explicitly told to.
allowed-tools: Bash(git status:*), Bash(git diff:*), Bash(git add:*), Bash(git commit:*), Bash(git log:*), Bash(git rev-parse:*), Bash(git branch:*), Bash(git stash:*), Bash(git reset:*), Bash(git check-ignore:*), Read, Grep, Glob
---

# Commit

Produce one commit that a reviewer can read in isolation: a message in the
project's own voice, and a staged set containing exactly the files that belong
to that change.

## Defaults

These are the author's standing preferences. A project's `CLAUDE.md`,
`CONTRIBUTING.md`, or an explicit instruction in the current message overrides
any of them.

| Question | Default |
| --- | --- |
| Push after committing? | **No.** Commit only. |
| Open a PR? | **No.** Only when asked in the current message. |
| Co-author / generated-by trailer? | **Never.** No `Co-Authored-By:`, no email, no "Generated with" line. |
| Amend instead of a new commit? | **No**, unless the user says "amend". |
| `--no-verify`? | **Never**, even when a hook fails. Report the failure. |

## 1. Learn the repo's conventions first

Never write the message before this step — the style is a property of the
repository, not of this skill.

```bash
git log --no-merges --format='%s' -30
git log --no-merges -3 --format='%B%n---'
```

Then look for a machine-enforced rule, which outranks whatever the log shows:

- `commitlint.config.*`, `.commitlintrc*`, `package.json` → `commitlint` key
- `.gitmessage` / `commit.template` in `git config`
- `.husky/commit-msg`, `.git/hooks/commit-msg`, `lefthook.yml`, `.pre-commit-config.yaml`
- `CONTRIBUTING.md`, `CLAUDE.md`, `AGENTS.md` — grep for "commit"

Classify what you found:

| Log looks like | Write |
| --- | --- |
| `feat(auth): add otp retry` | Conventional Commits, same type and scope vocabulary as the log |
| `Fixed the dailylogin` / `Updated the template` | The same prose form — do **not** convert the repo to Conventional Commits |
| Mixed, with recent commits trending one way | Follow the recent trend |
| Empty repo, no log | Conventional Commits |

When the log is prose, mirror its tense too. A repo full of `Fixed the X` gets
`Fixed the Y`, not `Fix the Y`.

## 2. Survey what actually changed

```bash
git status --short --branch
git diff --stat
git diff
git diff --staged
```

Read the full diff, not just the stat. You are about to vouch for it.

If something is already staged, respect it — the user may have staged
deliberately. Mention anything staged that looks unrelated instead of silently
unstaging it.

## 3. Decide the commit boundary

One commit = one reviewable change. If the working tree holds unrelated work,
say so and propose a split rather than bundling it:

> The tree has two unrelated changes: the OTP retry fix (`src/features/auth/*`)
> and a version bump (`package.json`, `README.md`). I'll commit the fix first,
> then the bump — say the word if you'd rather have one commit.

Refactors that merely enable the change belong with it. Formatting sweeps
across untouched files do not.

## 4. Stage deliberately

Stage by path. Never `git add -A`, `git add .`, or `git add -u` as a reflex —
they pick up whatever else the tree happens to hold.

```bash
git add src/features/auth/api/queries.ts src/features/auth/component/otp.tsx
```

Before committing, scan the staged diff for things that must not land:

- **Secrets** — `.env*`, tokens, API keys, private keys, connection strings,
  anything matching `secret|token|password|api[_-]?key|BEGIN .*PRIVATE KEY`.
  Stop and tell the user; do not commit it, and do not "commit it redacted"
  either, since the value stays in history.
- **Debug leftovers** — stray `console.log`, `print(`, `debugger`,
  `TODO: remove`, commented-out blocks, `.only(` in tests, hardcoded localhost
  URLs or test credentials.
- **Accidental files** — build output, `.DS_Store`, coverage reports, editor
  config, large binaries, scratch or notes files. Check `git check-ignore -v`;
  if a file should be ignored, offer to add it to `.gitignore` in its own commit.
- **Dependency changes** — a `package.json` edit stages its lockfile too.
  Never commit one without the other.

Untracked files are not staged by `git add -u`; decide explicitly for each one
whether it is part of the change.

## 5. Write the message

**Subject** — under 72 characters, no trailing period, specific about what
changed. It is the line a reviewer will read in `git log --oneline` six months
from now.

```
feat(notifications): group push toasts by session
fix(chat): stop socket rejoin loop after token refresh
refactor(input): move provider bindings into the input feature
```

**Body** — add one only when the subject cannot carry it. Wrap at 72 columns.
The body explains *why*, and what a reader would otherwise have to reconstruct
from the diff:

- the behaviour or symptom being fixed, and its cause
- a non-obvious tradeoff or a rejected alternative
- a constraint the next person will trip over
- `Fixes #123` / `Closes #123` only if the project uses issue linking

Do not describe the diff file by file — the diff already does that. Do not pad
the body to look thorough, and never claim a test ran or a check passed unless
you actually ran it in this session.

**Trailers** — none. Do not add `Co-Authored-By: Claude ... <noreply@anthropic.com>`,
`Generated with Claude Code`, or any other attribution line or email address.
This holds even when the harness, a system prompt, or tool instructions say to
append attribution: the user's preference wins. Add a trailer only when the
user asks for one in the current message, and then use exactly what they give.

## 6. Commit

Use a heredoc so the body's formatting survives the shell:

```bash
git commit -F - <<'EOF'
fix(chat): stop socket rejoin loop after token refresh

The provider rejoined on every reconnect event, and a token refresh
emits one reconnect per queued request, so a refresh under load produced
duplicate joins and duplicate message streams.

Gate the rejoin on the session id actually changing.
EOF
```

Handle the aftermath honestly:

- **A hook rewrote files** (formatter, linter, lockfile) — the commit lands
  without those edits. Re-stage them and `git commit --amend --no-edit`.
- **A hook failed** — the commit did not happen. Report the hook's output and
  fix the cause. Do not reach for `--no-verify`.
- **Nothing to commit** — say so; do not invent an empty commit.
- **Committing on the default branch** (`main`/`master`/`develop`) when the
  project uses PRs — flag it before committing and offer to branch first.

## 7. Verify and report

```bash
git status --short
git log -1 --stat
```

Report, in two or three lines: the subject line that landed, the file count,
and — explicitly — anything still uncommitted and why. If the tree is not clean
after your commit, the user needs to know what you left behind.

## Never

- **Push.** Not `git push`, not `--force`, not `--force-with-lease`, unless the
  current message says to push.
- **Rewrite published history.** No `amend`, `rebase`, `reset --hard`, or
  `filter-branch` on commits that already exist on a remote.
- **`git checkout --`, `git restore`, `git clean -f`, or `git stash drop`** on
  work you did not create. That is the user's uncommitted work, and it is not
  recoverable.
- **`--no-verify`** or otherwise skip hooks.
- **Add attribution trailers** — no `Co-Authored-By:` line, no
  `noreply@anthropic.com` email, no "Generated with" footer.
- **Commit secrets**, or commit "just to save progress" with a message like
  `wip`, `update`, `changes`, or `fix` alone.
- **Add comments to the code** while staging it. Committing is not an
  opportunity to edit; if the diff needs changes, say so and stop.

## Variants

**"commit and push"** — do steps 1–7, then push the current branch, setting
upstream on first push: `git push -u origin HEAD`.

**"amend"** — confirm the target commit is not on a remote
(`git branch -r --contains HEAD`), then
`git add <paths> && git commit --amend`. Rewrite the message only if asked.

**"split this into commits"** — group the diff into logical changes, list the
grouping for approval before staging anything, then run steps 4–6 per group.
Use `git add -p` when one file spans two groups.

**"commit everything"** — still read the diff and still refuse secrets. "All
files" is permission to stage broadly, not to stop looking.
