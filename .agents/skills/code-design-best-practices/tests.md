# Tests

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
