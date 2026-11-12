# Claude Skills

A collection of reusable [Claude Code](https://claude.com/claude-code) skills. Each skill is a folder at the repo root that holds its `SKILL.md` (what Claude reads) and a `README.md` (how to install and use it).

```
claude-skills/
  commit/
    SKILL.md
    README.md
  feature-module/
    SKILL.md
    README.md
  README.md
```

## Available skills

| Skill | What it does | Docs |
|---|---|---|
| **commit** | Stages and commits the current changes as one coherent commit, matching the repo's own message style (Conventional Commits or prose). Stages by path, refuses secrets, never pushes or skips hooks unless asked. | [commit/README.md](commit/README.md) |
| **feature-module** | Scaffolds a React/TypeScript feature as a vertical slice: `features/<name>/` with `api/{repository,queries,actions}.ts`, `types.ts`, `component/index.tsx` as the entry, and optional schemas, provider and reducer. | [feature-module/README.md](feature-module/README.md) |

## Quick install

Every skill installs the same way: copy its folder into a `.claude/skills/` directory.

- **One project (shared with your team):** copy it to `<your-project>/.claude/skills/<skill-name>/` and commit it.
- **All your projects:** copy it to `~/.claude/skills/<skill-name>/`.

```bash
cp -R /path/to/claude-skills/feature-module ~/.claude/skills/
```

Each skill's README has the full options (curl, symlink, Claude apps) and example prompts.

## Adding a new skill

1. Create a folder `<skill-name>/` at the repo root.
2. Add `<skill-name>/SKILL.md` with YAML frontmatter:
   ```markdown
   ---
   name: <skill-name>
   description: What it does and when Claude should use it.
   ---

   # Instructions...
   ```
3. Add `<skill-name>/README.md` with what it does, how to install it and how to use it.
4. Add a row to the **Available skills** table above.
