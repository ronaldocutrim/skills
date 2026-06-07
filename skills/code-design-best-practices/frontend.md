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
│       ├── components/               ◄ feature-scoped components reused across pages of THIS aggregate
│       ├── constants/                ◄ EVERY singleton/constant of THIS aggregate — NEVER declared inside components
│       │   └── index.ts
│       ├── utils/                    ◄ pure feature helpers
│       ├── types/                    ◄ shared entity types for the aggregate
│       │   └── index.ts
│       └── __tests__/                ◄ tests for Model and ViewModel (not View)
│           ├── <page>-model.test.ts
│           └── <page>-viewmodel.test.ts
│
├── components/                   ◄ design-system primitives — ONE copy per app, NEVER shared across apps
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
