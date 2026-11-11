# feature-module

A Claude Code skill that scaffolds a self-contained React/TypeScript feature as a **vertical slice**. Data access, types, validation, UI and any state the feature owns all live in one folder, in the same shape every time. Routes stay thin and just import the feature's entry component.

The skill itself is in [SKILL.md](SKILL.md). This README is for people; Claude only reads `SKILL.md`.

## What it generates

```
features/<name>/
  api/
    repository.ts   pure async functions that call the shared HTTP client (no React)
    queries.ts      read hooks (useQuery)
    actions.ts      write hooks (useMutation) that invalidate affected query keys
  types.ts          entity + API response types
  schemas.ts        optional: zod schemas for forms
  component/
    index.tsx       the feature entry (what a route renders)
    *.tsx           subcomponents
  provider/         optional: context + useX hook, only for shared state
  reducer/          optional: useReducer state machine
  utils.ts          optional: pure helpers
  mapper/           optional: API shape → UI shape
```

Before it writes anything, the skill reads the host project and matches it: where features live, the shared HTTP client, TanStack Query/SWR/RTK Query, singular vs plural folder names, where tests go, and how routes are wired (Next App Router, TanStack Router, React Router). It creates only the smallest shape the feature needs.

## Using it in another React project

Pick one of the options below.

### Option 1: Project-level (shared with your team)

Copy the skill into the React project's `.claude/skills/` folder and commit it. Everyone who opens the project in Claude Code gets the skill.

```bash
cd /path/to/your-react-project
```

```bash
mkdir -p .claude/skills/feature-module
```

```bash
curl -fsSL https://raw.githubusercontent.com/sarliram/claude-skills/main/feature-module/SKILL.md -o .claude/skills/feature-module/SKILL.md
```

Or, if you have this repo cloned locally:

```bash
cp -R /path/to/claude-skills/feature-module .claude/skills/
```

Result:

```
your-react-project/
  .claude/
    skills/
      feature-module/
        SKILL.md
  src/
  package.json
```

### Option 2: Personal (available in all your projects)

Install it once in your home directory, and Claude Code loads it in every project you open:

```bash
mkdir -p ~/.claude/skills && cp -R /path/to/claude-skills/feature-module ~/.claude/skills/
```

### Option 3: Symlink (stays in sync with this repo)

To have a `git pull` in this repo update the skill everywhere, symlink it instead of copying it:

```bash
ln -s /path/to/claude-skills/feature-module ~/.claude/skills/feature-module
```

(Use `.claude/skills/feature-module` inside a project for a project-level symlink. Symlinks aren't portable for teammates, so commit a real copy if the team needs it.)

### Option 4: Claude apps (claude.ai / desktop)

Zip the skill folder and upload it as a custom skill in the app's settings (the Skills section under Capabilities):

```bash
cd /path/to/claude-skills && zip -r feature-module.zip feature-module
```

## Invoking the skill

Start Claude Code in your React project:

```bash
claude
```

The skill **triggers automatically** when your request matches its description. You don't need to name it. Example prompts:

- `create a feature called notifications`
- `scaffold a payouts module — endpoints are GET /payout/list and POST /payout`
- `add the data layer for invoices`
- `wire up queries and mutations for products, follow our usual pattern`

You can also call it **explicitly** by name:

```
/feature-module invoices
```

## Tips for best results

- **Give the real API endpoints.** The skill won't invent REST paths. If you leave them out, it asks or leaves clearly marked stubs.
- **Share the response shape** if you have it (a sample JSON or the backend type). Otherwise the skill stubs `types.ts` and flags the guesses.
- **Have at least one existing feature.** The skill copies a neighbouring feature's idioms (import style, UI library, error handling). A neighbouring feature always takes priority over the skill's templates.
- **Make sure a shared HTTP client exists** (`src/client.ts`, `lib/api.ts`, an axios instance, …). `repository.ts` always goes through it and never calls `fetch` directly.
- Afterwards, check the follow-ups the skill reports: endpoints to confirm, nav links to add, and types it guessed.
