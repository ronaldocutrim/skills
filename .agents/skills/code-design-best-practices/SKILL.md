---
name: code-design-best-practices
description: Stack-agnostic code design and testing conventions, split into backend, frontend, database, and shared sections. Covers backend layer boundaries (lib/constants/utils/core/services/query/middleware), aggregate root layout (routes/service/schema/dto/index), DTO three-namespace pattern (DB/Input/Output), thin routes, pure services that never touch the database, permissions-over-roles RBAC, metrics in dedicated event tables, external-model wrappers named by intent; frontend MVVM conventions (Model/ViewModel/View per page, data/ layer of pure async functions, deps injection into Model), functional components with prop types above the component (type Props / type <fnName>Params naming), design-token-only styling, permission-gated routes, one UI primitive copy per app, loading/error/empty state handling, composition over boolean props; database conventions covering aggregate-name alignment across backend/frontend/db, primary keys, audit fields, indexing, money, soft delete, migrations; shared rules for kebab-case filenames (with MVVM PascalCase exception), path-alias imports, no nullish coalescing, no abbreviations, English identifiers, async/await over .then(); test conventions with when/then it() naming, makeSut factories, AAA blocks with stage comments and isolated database. Activates when authoring or refactoring any code — new aggregates, routes, services, schemas, DTOs, components, hooks, tests, or migrations.
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

## Before creating a new aggregate — ASK

When the work involves creating a new aggregate (backend, frontend, or both), the agent **MUST ask the user for the aggregate name first** — a single **kebab-case English noun** (e.g. `freight`, `invoice`, `subscription`).

The confirmed name is reused **verbatim** across three layers:

- Backend: `src/core/<aggregate>/` and file prefixes `<aggregate>.routes.ts` / `.service.ts` / `.schema.ts` / `.dto.ts`.
- Frontend: `src/core/<aggregate>/`.
- Database: tables prefixed by `<aggregate>_` (or schema `<aggregate>`).

Never create files for an unconfirmed name. Never let the name drift across layers. See [database.md](./database.md) for the full alignment rule.

## Sub-guides

Load the file that matches the touched area alongside this one:

- [Backend](./backend.md) — layer blueprint, aggregate layout, permissions, metrics, external-model naming, anti-patterns, error handling.
- [Frontend](./frontend.md) — MVVM blueprint (Model/ViewModel/View + data layer), component rules, design tokens, permission-gated routes, anti-patterns.
- [Database](./database.md) — aggregate-name alignment, column conventions, indexing, money/dates, soft delete, migrations.
- [Tests](./tests.md) — `when/then` naming, `makeSut` factory, AAA with stage comments, isolated database.

---

## Shared conventions (backend + frontend + database + tests)

- **Strict type-checking** with index access treated as possibly-undefined.
- **No abbreviations** — `viewModel`, not `vm`. Examples: `response` (not `res`), `model` (not `mdl`).
- Identifiers always **English** — even when the user request is in another language, translate before applying. Applies to variables, parameters, functions, types, schema fields, table columns, UI labels, object keys, and file names. Comments and user-facing error messages may use the project's primary spoken language.
- File names in **kebab-case** (e.g. `my-component.tsx`). **Exception**: the three MVVM layer files inside `core/<aggregate>/pages/<page>/` use PascalCase — `Model.ts`, `ViewModel.ts`, `View.tsx`.
- **Path-alias imports** (`@/`) for anything outside the current folder. Each workspace has `paths: { "@/*": ["./src/*"] }`. Relative paths (`./foo`, `../foo`) only for siblings in the same folder. When editing a file with non-conforming imports (`../../foo`, `../../../foo`), upgrade them to `@/foo` in the same edit — applies to new code AND any file opened for edit.
- **No `??`** (nullish coalescing) — code smell. Frequent use signals that the type is wrong or that the default belongs in the source layer, not the call site.
  - Field always exists? Remove the nullability from the type.
  - Env-var default? Validate at startup and fail fast — don't spread fallbacks.
  - `Record<Enum, X>` lookup? Make it exhaustive; no default needed.
  - Optional backend field? Normalize once in the Model, not in every View/ViewModel.
  - Possibly-absent array? Guarantee `[]` in the Data or Model layer, not in the consumer.
  - Rare exceptions: legitimate, immutable contract from an external library — document with a comment explaining why.
- **`async`/`await` only** — never chain `.then()` on a returned `Promise`. Async functions carry the explicit `async` flag. Functions that don't return a value use `await` without `return`, never `.then(() => undefined)`.
  ```ts
  // correct
  export async function <action>(data: <Action>FormData): Promise<<Action>Response> {
    const response = await api.post('/<route>', data);
    return response.data;
  }

  // wrong
  export function <action>(data: <Action>FormData): Promise<<Action>Response> {
    return api.post('/<route>', data).then((response) => response.data);
  }
  ```
- Permission IDs and the role→permission mapping live in `constants/permissions.ts` and `constants/roles.ts`. Adding a new permission means editing those constants, not introducing a new gate type.

---

## Process before declaring a task done

1. Identify each touched area and activate the matching tech skill.
2. Apply the rules above plus the rules in the relevant sub-guide(s).
3. Run the project's typecheck, lint, and build commands. Fix root causes — never workaround.
4. Use the project's package manager for new dependencies. Never edit `package.json` manually. Always stable versions (no preview/beta/alpha/rc/nightly).
