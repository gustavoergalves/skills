---
name: code-conventions
description: "Team code conventions and coding practices for PHP/Laravel: PSR-1/PSR-12, SOLID/KISS/DRY, layered architecture, naming of variables/methods/booleans/classes, arrays, strings, dates, method and class structure, exceptions, conditions, tests, PR size (max 12 changed files) and no unnecessary comments (self-explanatory code). Two modes: IMPLEMENT (write code that follows every rule, as simple and direct as possible) and REVIEW (check a diff/branch/PR/file against the rules and report each violation with the rule number, location and a concrete fix). Use when implementing, refactoring or reviewing PHP code, or when asked whether code follows the conventions."
---

# Code Conventions

Write code that is **efficient, simple and direct**: the least code that solves the task,
readable top to bottom, named so well that it needs no comments.

The complete rule set — every rule, numbered, with bad/good examples — is in
[references/rules.md](references/rules.md). **Read it in full before implementing or reviewing.**
Do not work from memory or from the summary below; the summary is only an index.

## Rule index

| § | Topic | Key rules |
| --- | --- | --- |
| 1 | Principles | PSR-1/PSR-12, SOLID/KISS/DRY, CodeSniffer, namespaces + autoload, PSR libs, param/property/return types MUST |
| 2 | Common | readability > write-ability, no magic methods, no dead/commented code, IDE-refactorable, low cyclomatic complexity, top-to-bottom reading, no code names in DB, no `exit`, data-changing commands confirm + log |
| 3 | Layers | Presentation / Application / Domain / Infrastructure; Domain depends on nothing (declares interfaces); thin Controllers/Commands |
| 4 | Pull requests | few lines, only task-related code, **max 12 changed files** |
| 5 | Variables | camelCase, informative English names, no type in name, one name per entity, property vars include object name, no scalar by-ref, don't reassign params, one declaration per line, no assignment result, no magic numbers, `null` for missing scalar |
| 6 | Booleans | `is`/`has`/`can`, no negative names, no boolean flag params |
| 7 | Arrays | `array_merge` not `+`, `array_key_exists` not `isset` |
| 8 | Strings | single quotes unless interpolation/escapes |
| 9 | Dates | `DateTimeImmutable`, string when scalar, no month/year intervals |
| 10 | Methods | full typing + PHPDoc for arrays/throws, verb names, private by default, no recursion, param order, `?` nullable, one return type, `null` when not found, `$result` in long methods, throw on errors, check → load → work → result, DTO over chain object |
| 11 | Classes | `Trait`/`Interface` suffix, `Abstract` prefix, private by default, public → protected → private order, static via class name, no `use` aliases |
| 12 | Comments | `// hack:` tag, source link for borrowed algorithms, **no unnecessary comments** |
| 13 | Exceptions | wrap third-party exceptions at once, hide messages from users, per-domain exceptions, no context in message |
| 14 | Conditions | boolean-only `if`, strict `===`, no `== true`, positive checks, parentheses with AND+OR, no chained ternaries |
| 15 | Tests | unit + integration tests for the change |

Rules worded *must / forbidden / cannot* are **MUST**; *should / avoid / if possible* are
**SHOULD** (see the modality note at the top of rules.md). Rule 5.2.6 is revoked — never report it.

If the repository has its own instructions (CLAUDE.md, AGENTS.md, phpcs.xml, linters) that
conflict with a rule here, the repository wins; mention the conflict.

## Mode 1 — Implement

1. **Read [references/rules.md](references/rules.md).**
2. **Size the work before coding (4.3).** Estimate which files will change. If it is more than
   12, stop and propose a split before writing code: smaller tasks, or a sequence of PRs that are
   each mergeable on their own (e.g. interface/contract → implementation → wiring → cleanup).
   Ask the user which split to take.
3. **Place each piece in its layer (§3).** Controller/Command: read input, call the domain,
   return. Business rules in Services/Models. Queries and API clients in Infrastructure, behind
   an interface the Domain owns.
4. **Write the minimum.** Only what the task needs (4.1, 4.2, KISS). No speculative parameters,
   abstractions, config, or error handling for impossible cases. Do not touch unrelated code.
5. **Name instead of commenting (12.3).** Before writing a comment, ask: can a better name, an
   extracted method, or a named constant/variable say this? If yes, do that. Only `// hack:`,
   source links, type-carrying PHPDoc and rare *why* comments survive.
6. **Self-check before saying done.** Walk every changed hunk against §1–§15 (types, names,
   booleans, conditions, return types, visibility, member order, exceptions, comments). Run the
   project's CodeSniffer/linter and tests if available. Confirm the changed-file count is ≤ 12.

## Mode 2 — Review

1. **Determine scope.** Default: the current branch vs its base plus uncommitted changes
   (`git merge-base HEAD <base>` then `git diff <merge-base>`). Or what the user names: staged
   changes, a PR number (`gh pr diff <n>`), a commit range, or specific files.
2. **Read [references/rules.md](references/rules.md)**, then the full diff. Open the
   surrounding code when a rule depends on context (layer of the class, whether a method is used
   elsewhere for 10.3/11.4, whether a name matches the entity's name elsewhere for 5.2.4).
3. **Check PR size (4.3).** `git diff --name-only <merge-base> | wc -l` (or `gh pr view <n>
   --json changedFiles`). More than 12 → MUST violation; suggest a concrete split based on the
   files actually changed. Also flag changes unrelated to the task (4.2).
4. **Check every changed line against every section, §1–§15.** Review only code added or
   modified in the scope — pre-existing code that was not touched is out of scope (4.2), unless
   the change makes it worse.
5. **Verify each finding** against the actual code before reporting: quote the line, confirm the
   rule really applies (e.g. 10.2 does not apply to Eloquent relations; 7.2 is about associative
   arrays; 12.3 allows type-carrying PHPDoc). Drop anything you cannot confirm.
6. **Report** in the user's language, using the format below. If there are no violations, say so
   plainly and list what was checked.

### Report format

````markdown
## Code conventions review — <scope>

**PR size:** <N> files changed (limit 12) — OK | VIOLATION
**Result:** <X> MUST · <Y> SHOULD violations

### MUST
1. **[10.8] Return `null` when not found** — `app/Services/OrderService.php:42`
   **Problem:** `findOrder()` returns `false` when the order does not exist.
   **Suggestion:**
   ```php
   public function findOrder(int $orderId): ?Order
   {
       return $this->orderRepository->findById($orderId);
   }
   ```

### SHOULD
1. **[6.3] Boolean flag parameter** — `app/Services/UserService.php:18`
   **Problem:** …
   **Suggestion:** …
````

Rules for the report:
- One entry per violation, grouped MUST then SHOULD, ordered by file and line.
- Always cite the rule number and name, the `path:line`, what is wrong in this code, and a
  concrete corrected snippet (not a generic restatement of the rule).
- Same rule broken many times in one file → one entry listing all lines.
- Never invent rules. Problems that are not in the catalog (bugs, security) may go in a short
  separate "Other observations" section, clearly marked as outside the conventions.
