---
name: feature-module
description: >-
  Scaffold a self-contained feature module using the vertical-slice convention:
  features/<name>/ with api/{repository,queries,actions}, types, optional schemas,
  a component/ tree whose index is the feature entry, and — only for stateful or
  realtime features — provider/ and reducer/. Routes stay thin and import the
  feature's entry component. Use this WHENEVER the user wants to create, scaffold,
  or add a new feature, module, resource, page surface, CRUD screen, or data slice
  in a React/TypeScript project, or asks to "follow the usual pattern", "our
  convention", or "the standard feature structure". Trigger on phrasings like
  "create a feature called notifications", "scaffold a payouts module", "add the
  data layer for invoices", "wire up queries for X" — even when the user does not
  name this skill. Prefer this over ad-hoc scaffolding so every feature looks the same.
---

# Feature Module Architecture

A feature is a **vertical slice**: everything one product surface needs — data access,
types, validation, UI, and any state it owns — lives in one folder, in the same shape
every time. Routes hold no logic; they import the feature's entry component.

The payoff is predictability. Any developer opening `features/<anything>/` finds the
same five or six places, so reading an unfamiliar feature costs nothing.

## The canonical shape

```
features/<name>/
  api/
    repository.ts   pure async functions that call the HTTP client; no React
    queries.ts      read hooks (useQuery)
    actions.ts      write hooks (useMutation), invalidating affected keys on success
  types.ts          entity + API response types
  schemas.ts        optional; validation schemas (zod) for forms
  component/
    index.tsx       the feature entry — what a route renders
    *.tsx           subcomponents; a deep sub-tree gets its own folder + index.tsx
  provider/         optional; React context + useX hook
  reducer/          optional; useReducer state machine, pairs with a complex provider
  utils.ts          optional; pure helpers specific to this feature
  mapper/           optional; API shape to domain shape translation
```

**Not every feature needs every part.** Add parts as the feature earns them:

| Part | Add it when |
|---|---|
| `api/` + `types.ts` + `component/` | Always — this is the minimum slice |
| `schemas.ts` | The feature has a form or validates input |
| `utils.ts` | Pure logic worth testing apart from the UI |
| `mapper/` | The API shape and the UI shape genuinely differ |
| `provider/` | The feature owns state that several subcomponents read |
| `reducer/` | That state is a machine with more than two or three transitions |

A feature that is just data plus UI is `api/`, `types.ts`, `component/`. Resist adding
a provider "for later" — a provider with one consumer is indirection, not architecture.

## Adapt to the host project before writing anything

This skill is portable; the project it lands in is not. Spend one pass establishing
these, then follow what you find — **a neighbouring feature always outranks this
document**:

1. **Where features live** — `src/features/`, `app/features/`, `modules/`? Match it.
2. **The HTTP client** — find the project's shared client (often `src/client.ts`,
   `lib/api.ts`, an axios instance, or a `request()` helper). Call it from
   `repository.ts`. **Never** call `fetch`/axios directly and never create a second
   client: the shared one is where auth headers, base URL, and 401 refresh live.
3. **The server-state library** — TanStack Query, SWR, RTK Query, or framework loaders
   and server actions. The three-file split holds regardless; only the hook names change.
4. **Singular or plural** — `component/` vs `components/`, `type.ts` vs `types.ts`.
   Both exist in the wild. Count the neighbours and follow the majority.
5. **Where tests go** — beside the code, or mirrored under a separate test root. If
   mirrored, a feature folder ships only what it ships, and tests import their subject
   through the path alias, never a relative path (a relative path resolves inside the
   test root and will not find the module).
6. **Route wiring** — a thin route file that imports `component/index.tsx`:
   Next App Router `app/<route>/page.tsx`, TanStack Router `routes/<route>.tsx`,
   React Router a lazy element. The route does routing; the feature does everything else.
7. **Global providers** — if the project has a provider registry, note that nesting
   order is usually load-bearing (a socket provider needs auth above it).

## Workflow

1. **Collect inputs.** The feature `<name>` (kebab-case) is required. Infer the rest and
   state your assumptions — do not interrogate.
   - **API paths**: ask for the real backend endpoints. Never invent a REST convention;
     a guessed path is a bug that compiles.
   - **Entities**: if unknown, stub a minimal type and flag it clearly as a guess.
   - **State**: does anything beyond one component need to read it? If not, skip
     `provider/` and `reducer/`.
2. **Check it does not exist.** If `features/<name>/` is there already, stop and ask
   rather than clobbering.
3. **Read one neighbour** of the closest shape and match its idioms — import style,
   error handling, naming, UI library.
4. **Write the tree**, smallest shape that fits.
5. **Wire the route** (thin) and register any global provider at the right depth.
6. **Report** created paths and the manual follow-ups: real endpoints, nav links,
   and every type you guessed.

## Rules worth keeping

These are what make the features feel like one codebase.

- **The three-file API split.** `repository.ts` is pure async functions with no React —
  callable from a test, a script, or a server component. `queries.ts` holds reads,
  `actions.ts` holds writes. Never put `useQuery`/`useMutation` in `repository.ts`;
  that is what welds data access to React and makes it untestable.
- **Query keys are stable arrays**, with parameters appended: `["widgets"]`,
  `["widget", id]`. A mutation invalidates the keys it affects in `onSuccess`, so
  dependent screens refresh without manual refetch calls.
- **`component/index.tsx` is the only entry.** Routes and other features import that;
  they never reach into a feature's internals. This is what lets you restructure a
  feature's insides without touching anything outside it.
- **Cross-feature imports are a smell.** If feature A needs feature B's component,
  the shared piece probably belongs in shared `components/`. Importing B's *types*
  is usually fine; importing its repository usually is not.
- **Shared UI beats a local copy.** Before writing a button, dialog, input, tab, or
  empty state inside a feature, check the shared component library. Accept small
  visual differences — consistency outranks a pixel-exact match. If the gap is a
  missing variant other screens would want too, extend the shared component rather
  than forking it.
- **Types describe the API, not your hopes.** `types.ts` mirrors what the backend
  actually returns, including nullable fields. If the UI wants a different shape,
  that is what `mapper/` is for.

## Templates

Generic; substitute the project's own client, hook library, and import style.

`api/repository.ts`

```ts
import { request } from "@/client";
import { Widget, WidgetListResponse } from "../types";

export const getWidgets = async () => {
  return request<WidgetListResponse>("GET", "/widget/list");
};

export const getWidgetById = async (id: string) => {
  return request<Widget>("GET", `/widget/${id}`);
};

export const updateWidget = async (id: string, name: string) => {
  return request<Widget>("PATCH", `/widget/${id}`, {}, { name });
};
```

`api/queries.ts`

```ts
import { useQuery } from "@tanstack/react-query";
import { getWidgets, getWidgetById } from "./repository";

export const useGetWidgets = () =>
  useQuery({
    queryKey: ["widgets"],
    queryFn: getWidgets,
  });

export const useGetWidgetById = (id: string) =>
  useQuery({
    queryKey: ["widget", id],
    queryFn: () => getWidgetById(id),
    enabled: Boolean(id),
  });
```

`api/actions.ts`

```ts
import { useMutation, useQueryClient } from "@tanstack/react-query";
import { updateWidget } from "./repository";

export const useUpdateWidget = () => {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: ({ id, name }: { id: string; name: string }) =>
      updateWidget(id, name),
    onSuccess: (_data, { id }) => {
      queryClient.invalidateQueries({ queryKey: ["widgets"] });
      queryClient.invalidateQueries({ queryKey: ["widget", id] });
    },
  });
};
```

`types.ts`

```ts
export interface Widget {
  id: string;
  name: string;
  created_at: string;
}

export interface WidgetListResponse {
  data: Widget[];
  total: number;
  page: number;
  page_size: number;
}
```

`component/index.tsx`

```tsx
"use client";

import { useGetWidgets } from "../api/queries";
import { WidgetList } from "./widget-list";

export const Widgets = () => {
  const { data, isLoading, isError } = useGetWidgets();

  if (isLoading) return <WidgetsLoading />;
  if (isError) return <WidgetsError />;

  return <WidgetList widgets={data?.data ?? []} />;
};

export default Widgets;
```

`provider/index.tsx` — only when state is genuinely shared

```tsx
"use client";

import { createContext, useContext, useMemo, useState, ReactNode } from "react";

interface WidgetContextValue {
  selectedId: string | null;
  select: (id: string | null) => void;
}

const WidgetContext = createContext<WidgetContextValue | null>(null);

export const WidgetProvider = ({ children }: { children: ReactNode }) => {
  const [selectedId, setSelectedId] = useState<string | null>(null);
  const value = useMemo(
    () => ({ selectedId, select: setSelectedId }),
    [selectedId]
  );

  return (
    <WidgetContext.Provider value={value}>{children}</WidgetContext.Provider>
  );
};

export const useWidget = () => {
  const context = useContext(WidgetContext);
  if (!context) throw new Error("useWidget must be used within WidgetProvider");
  return context;
};
```

## Anti-patterns

- A `utils.ts` that became a dumping ground — split it or move pieces to shared utils.
- A provider holding server state that the query library already caches.
- `repository.ts` importing React, or a component calling the HTTP client directly.
- A feature folder with one 600-line `index.tsx` and nothing else.
- Barrel files re-exporting everything — they defeat code splitting and invite cycles.
  The only barrel is `component/index.tsx`, and it exports the entry, not the internals.
