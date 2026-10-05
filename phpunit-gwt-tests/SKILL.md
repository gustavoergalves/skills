---
name: phpunit-gwt-tests
description: Structure PHPUnit test method bodies (this repo's tests, tests/Feature or tests/Unit) as Given/When/Then steps extracted into well-named private given*/when*/then* methods. Use when writing a new test method or reviewing/refactoring an existing one for readability. Does not cover naming of test methods, RefreshDatabase, or fixtures — those already live in CLAUDE.md's Testing section.
---

# PHPUnit Given-When-Then Structure

Given-When-Then (GWT, from BDD) is the same shape as Arrange-Act-Assert: **Given** sets up
preconditions and fixtures, **When** performs the single action under test, **Then** asserts
the outcome. Every test method body is made only of calls to private `given*`, `when*` and
`then*` methods, in that order, so the test reads as a specification and a reviewer can scan
it without reading setup or assertion details. **Do not use `// Given`, `// When`, `// Then`
comments** — the method names carry the structure.

This skill governs body structure only. Everything else — test method names,
`RefreshDatabase`, `actingAs`, feature tests never mocking the service layer, UUID literals —
is already required by `CLAUDE.md`'s Testing section; do not restate it here, follow it as-is.

## Procedure

1. Write the test body as a sequence of step calls — one or more `given*`, exactly one
   `when*`, one or more `then*`:
   ```php
   public function testExposesOrderAndLineItemFulfillmentDates(): void
   {
       $store = $this->givenActiveStore();
       $this->givenFulfilledOrder(
           $store,
           Carbon::parse('2026-05-15 12:00:00'),
           orderOverrides: ['order_fulfillment_date' => '2026-05-10 09:00:00'],
       );

       $this->whenToolIsCalled([
           'store_ids' => [$store->id],
           'date_from' => '2026-05-01',
           'date_to' => '2026-05-31',
           'date_basis' => 'fulfillment_date',
       ]);

       $this->thenFulfillmentDatesAre('2026-05-10 09:00:00', '2026-05-15 12:00:00');
   }
   ```
   Separate the Given, When and Then groups with one blank line. Nothing else goes in the
   body — no inline factories, assertions, or `$this->service->...` calls.
2. **Name steps after domain intent, not mechanics.** `givenActiveStore()`, not
   `givenStoreFactoryCreate()`; `thenFulfillmentDatesAre(...)`, not `thenAssertJsonPath(...)`.
   Prefix is mandatory (`given` / `when` / `then`), followed by a phrase that reads as a
   sentence: `givenFulfilledOrder`, `whenToolIsCalled`, `thenResponseIsForbidden`,
   `thenNoJobIsDispatched`.
3. **Keep the values that matter visible.** Pass the inputs and expected values the test is
   about as arguments (the dates and filters above); hide incidental setup (unrelated
   factory attributes, auth boilerplate, HTTP fakes) inside the step. Use named arguments
   when a literal's meaning isn't obvious.
4. **Pass state explicitly.**
   - `given*` steps return the fixtures later steps need (`$store = $this->givenActiveStore()`).
   - The `when*` step stores the result of the action in a private property of the test class
     (e.g. `private TestResponse $response;`, `private array $result;`) that `then*` steps
     read. Don't return it into a local just to pass it back to `then*`.
   - `then*` steps return `void` and may hold several related assertions.
5. **Exactly one `when*` per test** — the one behavior under test. Prior state-changing calls
   needed to reach the starting state (even to the same service method) are `given*` steps,
   e.g. `givenProductAlreadyFavourited($lot)` before `whenBellIsRung($lot)`.
6. **Exception tests**: put `expectException`/`expectExceptionMessage` in a `then*` step and
   call it *before* the `when*` step — PHPUnit requires the expectation to be registered
   first, and the name still declares the expected outcome:
   ```php
   public function testListThrowsOwnershipExceptionWhenCustomerNotLinked(): void
   {
       $this->thenOwnershipExceptionIsThrown();

       $this->whenFavouritesAreListedFor(self::UNKNOWN_SUB);
   }
   ```
   This is the only allowed reordering.
7. A test with no meaningful precondition simply has no `given*` call — don't add an empty
   one or a placeholder comment.
8. **Where the step methods live**: private methods in the same test class, below all test
   methods, grouped Given → When → Then. Reuse a step across tests in the class instead of
   writing near-duplicates; parameterise it rather than adding `givenActiveStoreWithX`,
   `givenActiveStoreWithY`. If steps are needed across several test classes, move them to a
   trait (`Tests\Concerns\...`); fixture *data* still belongs in `Tests\TestData` classes per
   `references/testing-guidelines.md`.
9. **Data-provider tests** pass the provider's parameters straight into the step calls; the
   provider itself is the parameterised Given.
10. **Refactoring an existing test** (with GWT comments or none): extract each section into
    step methods, delete the `// Given`/`// When`/`// Then` comments, and fix any arranging
    that leaked into the act (rule 5) at the same time. Don't change what the test verifies.

Done when every test method in the file being written or touched contains only `given*`,
`when*`, `then*` calls (exactly one `when*`), no GWT comments remain, step names read as
domain sentences, and `make pint` / `make test-filter f=<Name>` still pass.

## References

- For multi-step Given chains, mock/spy collaborators (the rare case in this repo, since
  feature tests must not mock the service layer per CLAUDE.md), HTTP tests, data providers,
  and full classes showing where step methods go, see `references/examples.md`.
- For general testing guidelines (Unit/Feature/Integration namespaces, mandatory feature
  tests, `Tests\TestData` fixture classes, Mockery argument verification, teardown), see
  `references/testing-guidelines.md`. Where it conflicts with `CLAUDE.md`,
  `CLAUDE.md` wins.
