---
name: phpunit-gwt-tests
description: Structure PHPUnit test method bodies (this repo's tests, tests/Feature or tests/Unit) as Given/When/Then blocks. Use when writing a new test method or reviewing/refactoring an existing one for readability. Does not cover naming, RefreshDatabase, or fixtures — those already live in CLAUDE.md's Testing section.
---

# PHPUnit Given-When-Then Structure

Given-When-Then (GWT, from BDD) is the same shape as Arrange-Act-Assert: **Given** sets up
preconditions and fixtures, **When** performs the single action under test, **Then** asserts
the outcome. Every test method body follows this shape, marked with three comments, so a
reviewer can scan a test in three lines without reading the assertions in detail.

This skill governs body structure only. Everything else — `test_snake_case` names,
`RefreshDatabase`, `actingAs`, feature tests never mocking the service layer, UUID literals —
is already required by `CLAUDE.md`'s Testing section; do not restate it here, follow it as-is.

## Procedure

1. Write (or split) the test body into exactly three commented sections, in order:
   ```php
   public function test_ringing_the_bell_on_a_lot_creates_the_favourite(): void
   {
       // Given
       $lot = $this->eligibleLot();

       // When
       $result = $this->service->setNotifyForCustomer(self::LINKED_SUB, 99, $lot->rambase_id, $this->tenantId('skanfil'), true);

       // Then
       $this->assertTrue($result['created']);
       $this->assertDatabaseHas('watchlist_items', [
           'auction_object_id' => $lot->id,
           'notify_enabled' => true,
       ]);
   }
   ```
2. Keep **When** to a single call — the one behavior under test. If arranging needs more than
   one call (factories, prior state-changing calls to reach the starting state), all of that
   stays under **Given**, even when it invokes the same service method the test exercises later.
3. Exception and domain-error tests still get all three comments even though `expectException`
   runs before the act. Put `expectException`/`expectExceptionMessage` under **Then** — despite
   its position in the code, it declares the expected outcome — and the call that triggers it
   under **When**:
   ```php
   public function test_list_throws_ownership_exception_when_customer_not_linked(): void
   {
       // Given
       // (no linked customer — nothing to arrange)

       // Then
       $this->expectException(WatchlistOwnershipException::class);

       // When
       $this->service->listForCustomer(self::UNKNOWN_SUB, 99, $this->tenantId('skanfil'), []);
   }
   ```
   Keep a `// Given` comment even with nothing to arrange — say so in one line — so the
   three-part shape stays scannable.
4. A test with no meaningful precondition (a pure function, a static factory) still keeps all
   three comments; an empty **Given** is a signal the case is trivial, not a reason to drop it.
5. Data-provider-driven tests keep the same three sections in the test method; the provider
   itself is the parameterised "Given" and needs no internal comments.
6. When reviewing an existing test that lacks the structure, add the three comments without
   reordering existing arrange/act/assert code unless the code itself violates rule 2 (extra
   arranging leaking into the act line) — fix that at the same time.

Done when every test method in the file being written or touched has `// Given`, `// When`,
`// Then` comments in that order, `// When` wraps a single action, and `make pint` /
`make test-filter f=<Name>` still pass.

## References

- For mock/collaborator-heavy tests (the rare case in this repo, since feature tests must not
  mock the service layer per CLAUDE.md) and multi-step Given chains, see
  `references/examples.md`.
