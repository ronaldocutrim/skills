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

## Sub-guides

Load the file that matches the touched area alongside this one:

- [Backend](./backend.md) — layer blueprint, aggregate layout, permissions, metrics, external-model naming, anti-patterns, error handling.
- [Frontend](./frontend.md) — feature blueprint, component rules, design tokens, permission-gated routes, anti-patterns.
- [Tests](./tests.md) — AAA format, isolated database.

---

## Shared conventions (backend + frontend + tests)

- **Strict type-checking** with index access treated as possibly-undefined.
- **No abbreviations** — `viewModel`, not `vm`.
- Identifiers always **English**. Comments and user-facing error messages may use the project's primary spoken language.
- File names in **kebab-case**.
- **Path-alias imports** (`@/`) for anything outside the current folder. Each workspace has `paths: { "@/*": ["./src/*"] }`. Relative paths (`./foo`, `../foo`) only for siblings in the same folder. When editing a file with non-conforming imports (`../../foo`, `../../../foo`), upgrade them to `@/foo` in the same edit — applies to new code AND any file opened for edit.
- **No `??`** (nullish coalescing). Fix the type or move the default to the source layer.
- Permission IDs and the role→permission mapping live in `constants/permissions.ts` and `constants/roles.ts`. Adding a new permission means editing those constants, not introducing a new gate type.

---

## Process before declaring a task done

1. Identify each touched area and activate the matching tech skill.
2. Apply the rules above plus the rules in the relevant sub-guide(s).
3. Run the project's typecheck, lint, and build commands. Fix root causes — never workaround.
4. Use the project's package manager for new dependencies. Never edit `package.json` manually. Always stable versions (no preview/beta/alpha/rc/nightly).
