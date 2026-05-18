# Frontend

## Blueprint

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

## Rules

Functional components only. **Prop types declared above the component, never inline.**

Routes are gated by a permission-aware wrapper that consumes the **same permission strings used on the backend**. Read groups / permissions / profile-status from the session-info endpoint (commonly `GET /api/me`).

**Design tokens only** — never hardcoded colors, spacing, or radii. Reference the design-system tokens defined by the project.

Every clickable element gets a pointer cursor affordance. Always preserve the focus-visible ring.

When multiple frontends ship from the same monorepo, **each owns its own copy of the component primitives** — accept duplication over a shared UI package, since shared UI packages calcify quickly under multi-app pressure.

## Frontend anti-patterns

1. Inline component prop types.
2. Hardcoded colors / spacing outside design tokens.
3. Forgetting to handle the `FAILED` (or equivalent error) status in the UI.
4. Class components.
5. Shared UI package across multiple frontends in the same monorepo.
