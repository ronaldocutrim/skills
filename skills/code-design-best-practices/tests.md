# Tests

## Test file location — ALWAYS in `__tests__/` inside the aggregate

Every test file lives in `__tests__/` at the **root of the aggregate it tests**, never co-located with the source file, never at a parent or shared level.

- Backend: `src/core/<aggregate>/__tests__/<file>.test.ts`
- Frontend: `src/core/<aggregate>/__tests__/<page>-model.test.ts` and `__tests__/<page>-viewmodel.test.ts`

```
src/core/freight/
├── pages/
│   └── despesas/
│       ├── Model.ts
│       ├── ViewModel.ts
│       └── View.tsx
├── data/
├── components/
└── __tests__/                         ◄ ALL tests for this aggregate live here
    ├── despesas-model.test.ts
    ├── despesas-viewmodel.test.ts
    └── ...
```

Rules:
- One `__tests__/` per aggregate. NEVER create a `__tests__/` inside `pages/<page>/`, `components/`, `data/`, or any sub-folder.
- File name mirrors the SUT: `<page>-model.test.ts` for `pages/<page>/Model.ts`, `<page>-viewmodel.test.ts` for `pages/<page>/ViewModel.ts`, `<file>.test.ts` for any other module.
- When creating or modifying a Model/ViewModel/service, the corresponding test file in `__tests__/` is created or updated in the **same edit**.
- View files (`View.tsx`) are NOT unit-tested — they are pure render and have no testable logic. The ViewModel is the boundary.

## `it()` naming — When/Then

Describe each test with `when <condition>, then <expected result>`:

```ts
it('when email is invalid, then should return validation error')
it('when user is not authenticated, then should redirect to login')
it('when submit succeeds, then should navigate to home')
```

## `makeSut` — mandatory setup

Every test file declares a `makeSut` factory that instantiates the System Under Test with its stubs/fakes. **Never instantiate the SUT inline inside an `it`.**

```ts
const makeSut = () => {
  const fakeData = { <action>: jest.fn().mockResolvedValue(<stub>) }
  const sut = use<Feature>Model(fakeData)
  return { sut, fakeData }
}

it('when <condition>, then <expected result>', () => {
  const { sut, fakeData } = makeSut()
  // ...
})
```

## AAA — Arrange / Act / Assert

Each `it` body has exactly three sections separated by a single blank line. Each block is labeled with a stage comment: `// <stage> — <short description>` (stage in English; description may use the project's primary spoken language).

```ts
it('when email is empty, then should disable submit button', () => {
  // arrange — prepara o SUT e o email vazio
  const { sut } = makeSut()
  const email = ''

  // act — define o email no model
  sut.setEmail(email)

  // assert — botão de submit fica desabilitado
  expect(sut.isSubmitDisabled).toBe(true)
})
```

- **Arrange**: prepares data, stubs, and initial state.
- **Act**: executes the action under test (a single call).
- **Assert**: verifies the expected outcome.
- Never mix sections — each one in its own block separated by a blank line.

Service tests run against an isolated database, never the dev DB.
