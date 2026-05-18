# Backend

## Blueprint

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

## Permissions, not roles

There is no single `role` per session and no `requireRole` gate. Each user belongs to one or more groups; each group has a list of role names that resolve to a set of `Permission` strings (e.g. `posts:publish`, `users:manage`, `settings:write`).

Permission lookups happen on the request context: `has('<permission>')`, `requirePermission(...)`, `requireAnyPermission(...)`, `hasPermission(...)`.

## Metrics live in dedicated tables

metadata values NEVER embed in domain entities. Always a separate event table keyed by the entity ID and an event kind:
- A pipeline event table for processing metrics.
- A message event table for per-interaction metrics.

Adding a new metered feature → create a parallel event table; do not bolt fields onto the domain entity.

## External-model naming by intent

Constants and wrappers in `lib/` and `constants/models.ts` are named by what they DO in the app, not by the technical model name:

```
✗ GPT_4O_MODEL, gpt4o, WHISPER_MODEL, TTS_MODEL
✓ summaryGenerationModel, titleSuggestionModel, transcribeAudio, synthesizeSpeech
```

The technical model ID lives in `constants/models.ts` and is referenced by the intent-named wrapper. Swapping providers/models changes one constant, not the call sites.

## Backend anti-patterns

1. Direct external SDK imports or binary spawns outside `lib/`.
2. Database access inside `services/`.
3. Logic in `<aggregate>.routes.ts` — routes must stay thin.
4. Importing internal files from another aggregate (skip the `index.ts` barrel).
5. Inferred return type on a service method.
6. Constants (prompts, prices, model IDs, lookup tables) loose in arbitrary files instead of `constants/`.
8. Cost/tokens/duration/model embedded in a domain entity instead of a dedicated event table.

## Backend error handling

- If an aggregate's existing layout deviates from the pattern (missing one of `index.ts` / `routes.ts` / `service.ts` / `schema.ts` / `dto.ts`), bring it into compliance in the same edit rather than perpetuating the deviation.
- If a service file under `services/` has a database import, STOP — relocate the DB call to `core/<aggregate>/` or `query/` before continuing.
- If a route handler grew logic, extract it to the aggregate's service in the same edit.
- If a domain entity gains a metric-shaped field (cost, tokens, duration, model, latency), STOP — add a parallel event table instead.
