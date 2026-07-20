# Frontend

## Blueprint

```
src/
├── lib/                          ◄ ONLY layer that imports external SDKs / clients
│   ├── <api-client>                  HTTP client wrapper (axios + auth interceptor)
│   ├── <auth-client>                 sign-in / session client
│   └── <storage-client>              browser/native storage adapter
│
├── constants/                    ◄ data-only, ZERO logic
│   ├── permissions.ts                SAME permission IDs as the backend
│   ├── routes.ts                     route path constants
│   └── enums.ts                      lookup tables shared across features
│
├── utils/                        ◄ pure functions — no I/O, no domain types
│
├── core/                         ◄ aggregate roots — SAME name as backend core/<aggregate> and DB tables
│   └── <aggregate>/
│       ├── pages/                    ◄ one folder per route/screen
│       │   └── <page>/
│       │       ├── Model.ts          ◄ data hook (TanStack Query) — receives deps, owns cache invalidation
│       │       ├── ViewModel.ts      ◄ presentation hook — derives values, owns navigation/alerts/pickers
│       │       └── View.tsx          ◄ pure render — props = { viewModel }, NO useState/useEffect/fetch
│       │
│       ├── data/                     ◄ pure async network functions (NOT hooks) — one file per endpoint action
│       │   ├── get-<entities>.ts
│       │   ├── create-<entity>.ts
│       │   └── delete-<entity>.ts
│       │
│       ├── components/               ◄ feature components — each in its OWN folder
│       │   └── <component>/
│       │       ├── use-<component>.ts    ◄ SMART hook — all logic (only if the component owns any)
│       │       ├── <component>.tsx       ◄ DUMB render — zero inline logic
│       │       └── index.ts              ◄ exports ONLY the component
│       ├── constants/                ◄ EVERY singleton/constant of THIS aggregate — NEVER declared inside components
│       │   └── index.ts
│       ├── utils/                    ◄ pure feature helpers
│       ├── types/                    ◄ shared entity types for the aggregate
│       │   └── index.ts
│       └── __tests__/                ◄ tests for Model and ViewModel (not View)
│           ├── <page>-model.test.ts
│           └── <page>-viewmodel.test.ts
│
├── components/                   ◄ design-system primitives (DUMB — ZERO logic) — ONE copy per app, NEVER shared across apps
│   └── <primitive>/                  each component is a folder
│       ├── <primitive>.tsx           render + tokens, no hook
│       └── index.ts                  exports ONLY the component
├── hooks/                        ◄ cross-feature reusable hooks
├── providers/                    ◄ context providers (auth, theme, branding)
├── app.tsx                       ◄ composition root: routes + providers + permission gating
└── main.tsx                      ◄ entrypoint
```

The aggregate folder name MUST match the backend `core/<aggregate>/` and the database table prefix. See [database.md](./database.md) for the aggregate-naming rule.

## MVVM layers

Every page in `pages/<page>/` is composed of three files with strict responsibilities. Filenames are PascalCase (exception to the global kebab-case rule).

### Model — `Model.ts`

Hook named `use<Page>Model`. Owns data fetching, mutations, and cache invalidation.

- Receives a typed `deps` parameter (functions from `data/` + IDs/context).
- Uses TanStack Query: `useQuery`, `useMutation`, `useQueryClient`.
- Applies defaults at this layer (e.g. `data === undefined ? [] : data`) — never push optional handling down to ViewModel/View.
- Exposes `perform<Action>` callbacks and flags (`isLoading`, `isCreating`, `isDeleting`).
- Exports `type <Page>Model = ReturnType<typeof use<Page>Model>`.

```ts
type use<Page>ModelDeps = {
  get<Entities>: typeof get<Entities>;
  create<Entity>: typeof create<Entity>;
  <id>: string;
};

export function use<Page>Model(deps: use<Page>ModelDeps) {
  const queryClient = useQueryClient();
  const queryKey = ['<entities>', deps.<id>] as const;

  const { data, isLoading } = useQuery({
    queryKey,
    queryFn: () => deps.get<Entities>(deps.<id>),
    enabled: !!deps.<id>,
  });

  const createMutation = useMutation({
    mutationFn: deps.create<Entity>,
    onSuccess: () => queryClient.invalidateQueries({ queryKey }),
  });

  return {
    <entities>: data === undefined ? [] : data,
    isLoading,
    isCreating: createMutation.isPending,
    performCreate: createMutation.mutateAsync,
  };
}

export type <Page>Model = ReturnType<typeof use<Page>Model>;
```

### ViewModel — `ViewModel.ts`

Hook named `use<Page>ViewModel`. Owns presentation logic and screen-bound side-effects.

- Receives `params` with `{ model: <Page>Model, ...routeParams }`.
- Derives display values (formatted strings, computed labels, header subtitles).
- Owns side-effects tied to the screen: navigation (`router.push`/`router.back`), alerts, OS pickers, clipboard.
- Exposes `handle<Event>` callbacks consumed by the View.
- Exports `type <Page>ViewModel = ReturnType<typeof use<Page>ViewModel>`.

### View — `View.tsx`

Functional component named `<Page>View`. Pure render.

- `type Props = { viewModel: <Page>ViewModel }`, parameter named `props`.
- NO `useState`, NO `useEffect`, NO network calls, NO business logic.
- Reads everything from `props.viewModel`.

### Data — `data/<action>.ts`

Pure async function — NOT a hook.

- Wraps the HTTP client; instruments Sentry on 5xx/network errors; rethrows so the Model can surface `isError`.
- One file per endpoint action: `get-expenses.ts`, `create-expense.ts`, `delete-expense.ts`.
- Injected into Model as a dep — Model NEVER imports from `data/` directly. This makes Models trivial to test with stubs.

## Rules

### Required libraries

These three are the default stack for every frontend in this codebase — do not reach for alternatives (SWR, Formik, manual `fetch` + `useState`, hand-rolled validators) without an explicit reason.

- **TanStack Query** — the only way to manage server state. Queries and mutations live in `Model.ts`. NO `useEffect` + `useState` pairs to fetch data.
- **React Hook Form** — the only way to manage form state (values, dirty/touched, submission, field-level errors). `useForm` lives in `ViewModel.ts` (or in a leaf form component when the form is fully self-contained). NO ad-hoc `useState` per field.
- **Zod** — the only way to declare input validation. Declare the schema once, derive the TS type with `z.infer<typeof schema>`, and wire it into React Hook Form via `zodResolver`. Reuse the same schema for the payload type passed to `data/<action>.ts` when the shapes line up.

```ts
// ViewModel.ts
const <action>Schema = z.object({ ... });
type <Action>FormData = z.infer<typeof <action>Schema>;

const form = useForm<<Action>FormData>({
  resolver: zodResolver(<action>Schema),
  defaultValues: { ... },
});

const handleSubmit = form.handleSubmit(async (values) => {
  await params.model.perform<Action>(values);
});
```

### Components and naming

Functional components only — no class components, no `React.FC`. **Prop and parameter types declared above the function, never inline in the signature.** Mandatory naming:
- Components: `type Props` — parameter is named `props`.
- Hooks and functions: `type <functionName>Params` — parameter is named `params`. For Model deps, use `type use<Page>ModelDeps`.

```ts
// correct — component
type Props = { viewModel: <Page>ViewModel };
export function <Page>View(props: Props) { ... }

// correct — hook
type useDespesasViewModelParams = { model: DespesasModel; freightId: string };
export function useDespesasViewModel(params: useDespesasViewModelParams) { ... }

// wrong
export function <Page>View({ viewModel }: { viewModel: <Page>ViewModel }) { ... }
```

Always handle **loading, error, and empty** states — the Model surfaces the flags, the ViewModel decides what to expose, the View renders. Prefer **composition** (compound components) over many boolean props.

Routes are gated by a permission-aware wrapper that consumes the **same permission strings used on the backend**. Read groups / permissions / profile-status from the session-info endpoint (commonly `GET /api/me`).

**Design tokens only** — never hardcoded colors, spacing, or radii. Reference the design-system tokens defined by the project.

Every clickable element gets a pointer cursor affordance. Always preserve the focus-visible ring.

When multiple frontends ship from the same monorepo, **each owns its own copy of the component primitives** — accept duplication over a shared UI package, since shared UI packages calcify quickly under multi-app pressure.

### Smart and dumb components

Logic and rendering never live in the same file. A component's **smart** half is always a **hook**; its **dumb** half is the render. This is the MVVM split (`Model`/`ViewModel` = smart, `View` = dumb) applied at every level, including single low-level components.

- **Smart** = hooks. At page level: `Model.ts` + `ViewModel.ts`. At component level: a colocated `use<Component>` hook. Hooks own data fetching, mutations, derivations, navigation, and every side-effect. **This is the ONLY place logic lives.**
- **Dumb** = the render files (`View.tsx`, feature components, design-system primitives). The component **body carries ZERO logic** — it either takes pure props or calls its own single `use<Component>()` hook, then returns markup. No inline queries, mutations, navigation, or domain formatting.

Two tiers of components below the page:

1. **Design-system primitives** (`src/components/`) — the lowest tier, shared across the app. **Zero logic, zero domain knowledge, never a hook.** Props → markup + design tokens, nothing else. NO TanStack Query, NO domain types, NO navigation, NO formatting of domain values, NO `import` from `core/`. A primitive may hold *purely visual* self-contained state (a `<Popover>` open flag, hover, controlled-input focus) — never business/domain state.
2. **Feature components** (`core/<aggregate>/components/`) — reused across the aggregate's pages. **Default to dumb**: receive data as props already derived by the page's ViewModel. When a component is **genuinely self-contained and reused across pages** (a picker that loads its own options, a notifications bell), it owns its logic in a colocated `use<Component>` hook — never inline in the component body.

Prefer lifting data to the page's `Model` and passing props down. Give a component its own `use<Component>` hook **only** when it is self-contained and reused across pages — otherwise you fragment the fetch and lose the Model's centralized cache orchestration. A component body with a `useQuery`, a `router.push`, or a `formatCurrency` inside it is the most common violation: move it into the hook (or hoist it to the page's Model/ViewModel).

### Component folder layout

**Every component is a folder**, never a loose file — primitives and feature components alike.

```
<component-name>/
├── use-<component-name>.ts   ◄ SMART — all logic (queries, mutations, derivations, side-effects). Present ONLY when the component owns logic.
├── <component-name>.tsx      ◄ DUMB — calls its own use-hook (or takes pure props) and renders. ZERO inline logic.
└── index.ts                  ◄ re-exports ONLY the component — NEVER the hook
```

- The render file `<component-name>.tsx` exports `<ComponentName>` (PascalCase). If the component owns logic, its body calls `use<ComponentName>()` **once** and renders the result — nothing else.
- The hook file `use-<component-name>.ts` exports `use<ComponentName>` and holds every query, mutation, derivation, and side-effect. It receives its `data/` functions as injected deps (same rule as `Model` — testable with stubs).
- `index.ts` exports **only the component**. The hook stays **private to the folder** — consumers import `<ComponentName>` and see a dumb, props-only API. Never re-export the hook.
- Files are kebab-case (`freight-picker.tsx`, `use-freight-picker.ts`); the component is PascalCase, the hook is camelCase. This is NOT the MVVM PascalCase exception — component folders keep kebab-case filenames.
- A pure leaf with no logic (a primitive, a props-only feature component) **skips the hook file** — just `<component-name>.tsx` + `index.ts`.

```ts
// core/freight/components/freight-picker/use-freight-picker.ts — SMART (private to the folder)
type useFreightPickerParams = { getFreights: typeof getFreights };

export function useFreightPicker(params: useFreightPickerParams) {
  const { data, isLoading } = useQuery({ queryKey: ['freights'], queryFn: params.getFreights });
  return { options: data === undefined ? [] : data.map(toOption), isLoading };
}
```

```tsx
// core/freight/components/freight-picker/freight-picker.tsx — DUMB, zero inline logic
type Props = { value: string; onChange: (value: string) => void };

export function FreightPicker(props: Props) {
  const { options, isLoading } = useFreightPicker({ getFreights });
  if (isLoading) return <Skeleton />;
  return <Select options={options} value={props.value} onChange={props.onChange} />;
}
```

```ts
// core/freight/components/freight-picker/index.ts — exports ONLY the component
export { FreightPicker } from './freight-picker';
```

### Loading patterns

Loading is **derived state, never `useState`**. The flags originate in the **Model** (from TanStack Query), the **ViewModel** turns them into a single status, and the **View** renders the matching shape with dumb primitives.

**Flags come from the Model:**
- `isLoading` — first fetch, no data yet.
- `isFetching` — background refetch with data already on screen.
- `isCreating` / `isUpdating` / `isDeleting` — from `mutation.isPending`.

**Pick the shape by context:**
- **Initial load** (no data yet) → a **skeleton** that mirrors the final layout. Not a centered spinner for content regions.
- **Mutation in flight** → **disable the trigger** and show an inline pending affordance (button spinner + disabled label). The rest of the screen stays interactive. Prevents double-submit.
- **Background refetch** (data already visible) → a **subtle** indicator (top progress bar, reduced opacity). Never replace visible data with a skeleton.

**Resolve states in a fixed precedence** — error → initial-loading → empty → content. Derive one status in the ViewModel; branch once in the View.

```ts
// ViewModel.ts — collapse raw flags into a single status; never leak them to the View
type <Page>Status = 'error' | 'loading' | 'empty' | 'ready';

function resolveStatus(params: use<Page>ViewModelParams): <Page>Status {
  if (params.model.isError) return 'error';
  if (params.model.isLoading) return 'loading';
  if (params.model.<entities>.length === 0) return 'empty';
  return 'ready';
}
```

```tsx
// View.tsx — one branch, a dumb primitive for each state
export function <Page>View(props: Props) {
  const { status } = props.viewModel;
  if (status === 'error') return <ErrorState onRetry={props.viewModel.handleRetry} />;
  if (status === 'loading') return <<Page>Skeleton />;
  if (status === 'empty') return <EmptyState />;
  return <<Page>Content viewModel={props.viewModel} />;
}
```

- **Skeletons are dumb primitives** — one `<Skeleton>` in `src/components/`, composed into feature-level skeletons. Token-driven, zero logic.
- **Mutation triggers** read the matching flag: `disabled={props.viewModel.isSubmitting}` plus a pending label — the flag comes from the Model, surfaced by the ViewModel.

### Aggregate constants

Every singleton and constant that belongs to an aggregate lives in `core/<aggregate>/constants/` — and **nowhere else**. This is the single source of truth for anything fixed and shared inside the aggregate: default values, option/lookup tables, query-key prefixes, status maps, magic numbers/strings, config objects, and any module-level singleton (a configured client instance, a cache, a formatter).

- **NEVER declare constants inside a component** (`View.tsx`, feature components) or inside `Model.ts` / `ViewModel.ts`. A literal that only one component uses still moves to `constants/` the moment it carries meaning (an option list, a limit, a label map) — components only consume constants, never define them.
- Inline literals are allowed only when they are intrinsic to the JSX and carry no domain meaning (e.g. `flex={1}`, a single icon size). The moment a value names a rule, a default, or a set of options, it belongs in `constants/`.
- The folder is **data-only, ZERO logic** — same rule as the root `constants/`. Derivations go to `utils/`.
- Cross-feature constants belong in the **root** `src/constants/`; aggregate-only constants stay in `core/<aggregate>/constants/`. Never reach into another aggregate's `constants/`.

## Frontend anti-patterns

1. Inline component prop types.
2. Hardcoded colors / spacing outside design tokens.
3. Forgetting to handle loading / error / empty states.
4. Class components.
5. Shared UI package across multiple frontends in the same monorepo.
6. `Model.ts` importing directly from `data/` (must come through `deps`).
7. `useState` / `useEffect` / network calls inside `View.tsx`.
8. Business logic or formatting inside `View.tsx`.
9. Navigation, alerts, or OS pickers inside `Model.ts` (those belong in `ViewModel.ts`).
10. Optional handling (`?? []`, `?? ''`) leaking into `ViewModel` or `View` — defaults belong in `Model`.
11. Async server state managed with `useState` + `useEffect` instead of TanStack Query.
12. Form state managed with one `useState` per field instead of React Hook Form.
13. Hand-written validation (regex, if-blocks, manual error messages) instead of a Zod schema fed into `zodResolver`.
14. Constants or singletons declared inside components / `Model.ts` / `ViewModel.ts` instead of `core/<aggregate>/constants/`.
15. Logic (data fetching, mutations, domain formatting, navigation, side-effects) inside a design-system primitive or any UI component — it belongs in `Model.ts` / `ViewModel.ts`.
16. Design-system primitive that imports from `core/`, holds domain state, or knows domain types.
17. Loading tracked with `useState` instead of the Model's TanStack Query flags.
18. Centered spinner where a skeleton belongs, or replacing already-visible data with a full skeleton during a background refetch.
19. Mutation trigger without a disabled / pending state (double-submit risk).
20. Component defined as a loose file instead of its own folder with an `index.ts`.
21. `index.ts` re-exporting the `use<Component>` hook — the smart half must stay private to the folder.
22. Logic inline in a component body instead of its colocated `use<Component>` hook.
