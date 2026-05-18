---
name: code-design-best-practices
description: Stack-agnostic code design and testing conventions, split into backend, frontend, and shared sections. Covers backend layer boundaries (lib/constants/utils/core/services/query/middleware), aggregate root layout (routes/service/schema/dto/index), DTO three-namespace pattern (DB/Input/Output), thin routes, pure services that never touch the database, permissions-over-roles RBAC, metrics in dedicated event tables, external-model wrappers named by intent; frontend conventions for functional components with prop types above the component, design-token-only styling, permission-gated routes, one UI primitive copy per app; shared rules for kebab-case filenames, path-alias imports, no nullish coalescing, no abbreviations, English identifiers; AAA test format with descriptive labels and isolated database. Activates when authoring or refactoring any code — new aggregates, routes, services, schemas, DTOs, components, hooks, tests, or migrations.
---

# Code Design Best Practices

Stack-agnostic conventions for code structure and tests. Apply these rules on every edit — independent of language, framework, ORM, UI library, or package manager. Load tool-specific skills alongside this one whenever the touched area is covered by one.

## When to load this skill

- Adding or modifying a route, service, schema, DTO, or aggregate.
- Creating or editing a component, hook, page, or form.
- Writing or updating tests (unit, integration, end-to-end).
- Authoring a database schema or migration.
- Reviewing a diff for project consistency.

Skip when the task is environment setup, deployment, or infra wiring.

---

## Shared conventions (backend + frontend)

- **Strict type-checking** with index access treated as possibly-undefined.
- **No abbreviations** — `viewModel`, not `vm`.
- Identifiers always **English**. Comments and user-facing error messages may use the project's primary spoken language.
- File names in **kebab-case**.
- **Path-alias imports** (`@/`) for anything outside the current folder. Each workspace has `paths: { "@/*": ["./src/*"] }`. Relative paths (`./foo`, `../foo`) only for siblings in the same folder. When editing a file with non-conforming imports (`../../foo`, `../../../foo`), upgrade them to `@/foo` in the same edit — applies to new code AND any file opened for edit.
- **No `??`** (nullish coalescing). Fix the type or move the default to the source layer.
- Permission IDs and the role→permission mapping live in `constants/permissions.ts` and `constants/roles.ts`. Adding a new permission means editing those constants, not introducing a new gate type.

---

## Backend

### Blueprint

```
src/
├── lib/                          ◄ ONLY layer that imports external SDKs or spawns binaries
│   ├── <db-client>                   singleton ORM / database client
│   ├── <auth-server>                 session resolver / auth server
│   ├── <ai-client>                   AI SDK wrapper
│   └── <storage-client>              object / file storage
│
├── constants/                    ◄ data-only, ZERO logic
│   ├── prompts.ts                    system prompts (never inline in services/routes)
│   ├── models.ts                     technical model IDs (referenced by intent-named wrappers)
│   ├── permissions.ts                permission ID catalog
│   └── roles.ts                      role → permission[] mapping
│
├── utils/                        ◄ pure functions — no I/O, no domain types
│
├── core/                         ◄ aggregate roots — the ONLY layer with database access
│   └── <aggregate>/
│       ├── index.ts                  barrel — only re-exports <aggregate>.routes
│       ├── <aggregate>.routes.ts     THIN: parse → call service → return envelope
│       ├── <aggregate>.service.ts    logic + DB access; every fn has EXPLICIT return type
│       ├── <aggregate>.schema.ts     input validation schemas
│       └── <aggregate>.dto.ts        three namespaces, NO comments:
│                                       ├ <Aggregate>DB      ORM select/include shapes + payload types
│                                       ├ <Aggregate>Input   request types (derived from schema)
│                                       └ <Aggregate>Output  service return types
│
├── services/                     ◄ pure domain orchestration — NEVER imports DB client
├── query/                        ◄ DB queries that legitimately span aggregates
├── middleware/                   ◄ error-handler, auth, scope, permission gates
├── app.ts                        ◄ composition root
└── server.ts                     ◄ entrypoint
```

Cross-aggregate access goes through `core/<aggregate>/index.ts` only — never reach into another aggregate's internal files.

### Permissions, not roles

There is no single `role` per session and no `requireRole` gate. Each user belongs to one or more groups; each group has a list of role names that resolve to a set of `Permission` strings (e.g. `posts:publish`, `users:manage`, `settings:write`).

Permission lookups happen on the request context: `has('<permission>')`, `requirePermission(...)`, `requireAnyPermission(...)`, `hasPermission(...)`.

### Metrics live in dedicated tables

metadata values NEVER embed in domain entities. Always a separate event table keyed by the entity ID and an event kind:
- A pipeline event table for processing metrics.
- A message event table for per-interaction metrics.

Adding a new metered feature → create a parallel event table; do not bolt fields onto the domain entity.

### External-model naming by intent

Constants and wrappers in `lib/` and `constants/models.ts` are named by what they DO in the app, not by the technical model name:

```
✗ GPT_4O_MODEL, gpt4o, WHISPER_MODEL, TTS_MODEL
✓ summaryGenerationModel, titleSuggestionModel, transcribeAudio, synthesizeSpeech
```

The technical model ID lives in `constants/models.ts` and is referenced by the intent-named wrapper. Swapping providers/models changes one constant, not the call sites.

### Backend anti-patterns

1. Direct external SDK imports or binary spawns outside `lib/`.
2. Database access inside `services/`.
3. Logic in `<aggregate>.routes.ts` — routes must stay thin.
4. Importing internal files from another aggregate (skip the `index.ts` barrel).
5. Inferred return type on a service method.
6. Constants (prompts, prices, model IDs, lookup tables) loose in arbitrary files instead of `constants/`.
8. Cost/tokens/duration/model embedded in a domain entity instead of a dedicated event table.

### Backend error handling

- If an aggregate's existing layout deviates from the pattern (missing one of `index.ts` / `routes.ts` / `service.ts` / `schema.ts` / `dto.ts`), bring it into compliance in the same edit rather than perpetuating the deviation.
- If a service file under `services/` has a database import, STOP — relocate the DB call to `core/<aggregate>/` or `query/` before continuing.
- If a route handler grew logic, extract it to the aggregate's service in the same edit.
- If a domain entity gains a metric-shaped field (cost, tokens, duration, model, latency), STOP — add a parallel event table instead.

---

## Frontend

### Blueprint

```
src/
├── lib/                          ◄ ONLY layer that imports external SDKs / clients
│   ├── <api-client>                  HTTP client wrapper
│   ├── <auth-client>                 sign-in / session client
│   └── <storage-client>              browser storage adapter
│
├── constants/                    ◄ data-only, ZERO logic
│   ├── permissions.ts                SAME permission IDs as the backend
│   ├── routes.ts                     route path constants
│   └── enums.ts                      lookup tables shared across features
│
├── utils/                        ◄ pure functions — no I/O, no domain types
│
├── features/                     ◄ feature modules — the ONLY layer that owns data fetching
│   └── <feature>/
│       ├── index.ts                  barrel — only public exports
│       ├── <feature>.routes.tsx      page/screen components — THIN: read params → call hooks → render
│       ├── <feature>.hooks.ts        queries + mutations + cache invalidation
│       ├── <feature>.schema.ts       form-input validation schemas
│       └── <feature>.dto.ts          request / response types
│
├── components/                   ◄ design-system primitives — ONE copy per app, NEVER shared across apps
├── hooks/                        ◄ cross-feature reusable hooks
├── providers/                    ◄ context providers (auth, theme, branding)
├── app.tsx                       ◄ composition root: routes + providers + permission gating
└── main.tsx                      ◄ entrypoint
```

### Rules

Functional components only. **Prop types declared above the component, never inline.**

Routes are gated by a permission-aware wrapper that consumes the **same permission strings used on the backend**. Read groups / permissions / profile-status from the session-info endpoint (commonly `GET /api/me`).

**Design tokens only** — never hardcoded colors, spacing, or radii. Reference the design-system tokens defined by the project.

Every clickable element gets a pointer cursor affordance. Always preserve the focus-visible ring.

When multiple frontends ship from the same monorepo, **each owns its own copy of the component primitives** — accept duplication over a shared UI package, since shared UI packages calcify quickly under multi-app pressure.

### Frontend anti-patterns

1. Inline component prop types.
2. Hardcoded colors / spacing outside design tokens.
3. Forgetting to handle the `FAILED` (or equivalent error) status in the UI.
4. Class components.
5. Shared UI package across multiple frontends in the same monorepo.

---

## Tests

Unit, integration, and end-to-end tests follow AAA with descriptive labels and a single blank line between blocks. Format: `// <stage> — <short description>` (stage in English; description may use the project's primary spoken language).

```ts
// arrange — sign in as user and load the home page
...arrange code...

// act — click the "Discard" button on the first draft
...act code...

// assert — draft count decreases by 1
...assert code...
```

Service tests run against an isolated database, never the dev DB.

---

## Process before declaring a task done

1. Identify each touched area and activate the matching tech skill.
2. Apply the rules above.
3. Run the project's typecheck, lint, and build commands. Fix root causes — never workaround.
4. Use the project's package manager for new dependencies. Never edit `package.json` manually. Always stable versions (no preview/beta/alpha/rc/nightly).
