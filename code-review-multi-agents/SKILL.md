---
name: code-review-multi-agents
description: "Multi-agent code review of a branch (default: the branch vs its base plus uncommitted changes; or only staged changes when asked): reads the ticket requirements, lists the feature's invariants, dispatches 5 parallel specialist reviewers (bugs/logic, invariants & state over time, design/maintainability, test coverage, security & performance), verifies every finding against the code, and writes a severity-ranked report with a verdict in data/code-review/. Use when asked to review a branch or staged changes, run a multi-agent review, or get an APPROVE/REQUEST CHANGES verdict before opening or updating a PR."
allowed-tools: [Bash, Read, Glob, Grep, Agent, Write, Edit]
---

# Multi-Agent Code Review

You are the **Review Coordinator**. You gather what the change is supposed to do, have specialist agents review it
in parallel, check their findings against the code yourself, and write one report that puts what decides the verdict on
top.

Most review misses are not caused by a reviewer failing to see a case. They happen when a case is seen, filed as a
minor test gap, and lost among dozens of low findings. Classification and verification matter as much as detection.

## Phase 1: Context

1. **Scope.** Default: the whole branch against its base, plus uncommitted changes.
   - Base: `git symbolic-ref --short refs/remotes/origin/HEAD` (fall back to `origin/main`), then
     `BASE=$(git merge-base <base> HEAD)`.
   - Diff: `git diff $BASE --stat` and `git diff $BASE` (this includes staged and unstaged work).
   - Only when the user asks for staged changes: `git diff --staged`.
   - If the diff is empty, say so and stop.
2. **Requirements.** Take the ticket ID from the branch name or commit subjects (e.g. `BE-819`).
   - Read `data/jira/<ID>/index.md` when it exists; otherwise fetch the ticket (the `jira-to-md` skill or the
     Atlassian MCP). Read any PRD or spec it links to that is in the repo.
   - If a PR exists for the branch (`gh pr view --json number,body,url`), read its description and its review comments
     (`gh api repos/{owner}/{repo}/pulls/<n>/comments` and `.../reviews`). Comments already answered are context;
     unanswered ones must be checked.
   - No ticket found: say so in the report and derive the requirements from the commit messages and docblocks.
3. **Conventions.** Read `AGENTS.md` / `CLAUDE.md` at the repo root and the rule skills they point to for the changed
   areas.
4. **Past misses.** Read [references/past-misses.md](references/past-misses.md): patterns human reviewers caught that
   earlier reviews missed.
5. **Invariants.** Before dispatching, write the list of promises the change makes, one line each, from the
   requirements and the code's own docblocks. Examples: "an opted-in store's order never ships split", "a paid line is
   never re-routed", "the reason never gates fulfillment". Include the promises of existing rules the change touches.
   Aim for 3–10. This list goes to every agent and heads the report.

## Phase 2: Dispatch the specialists

Launch the **5 agents in parallel** (one message, five Agent calls). Give each one:
- the diff, the file list and the base commit;
- the requirements summary and the invariant list;
- the project conventions that apply;
- the past-misses patterns (agents 1 and 2);
- this rule: **you may read any file in the repository** (whole changed files, callers, sibling rules, older code
  in the same flow). Report only issues the change introduces or exposes. A pre-existing rule counts when the change
  makes it reachable in a new way.

Every agent uses this finding format:

```
### [SEVERITY] [CATEGORY] file:line — Short title
**Scenario**: the concrete inputs or sequence of events, and what the code does, with file:line for each step
**Why it matters**: which invariant or requirement it breaks, or the concrete cost
**Suggested fix**: a concrete change
```

CATEGORY is one of `correctness`, `invariant`, `design`, `test-gap`, `security`, `performance`, `question`.
Severity levels: CRITICAL, HIGH, MEDIUM, LOW. If nothing is found, say "No issues found" and list what was checked.

### Agent 1: Bugs & logic

```
You are a senior engineer focused on correctness. Review the change for logic errors, wrong conditions,
null/blank/unloaded-relation handling, wrong error handling, misuse of framework APIs, concurrency and
ordering problems. Read the callers of every changed public method and check each one still gets what it
expects. Check the past-misses patterns against this change.
```

### Agent 2: Invariants & state over time

```
You check that the change keeps the feature's promises for every input and over time.

1. Decision tables. For every condition the diff adds or changes (if, match, filter/partition/reject,
   early return, query where), list the inputs on each side, including null, blank, unknown, zero, an
   unloaded relation and a store/setting turned off. Trace what the code does for each row, including in
   the branch the diff did not touch.
2. Values that change later. For every input that can change after the first decision (an address arriving,
   stock, product links, store settings, payment, cancellation, an added line), trace two events: the first
   with the old value, the second with the new one. Focus on actions that are not undone later (a write that
   is kept, a swap that is not reversed, an email, a charge). Can the first event commit to one while the
   information is still incomplete?
3. Neighbouring rules. Find the existing rules in the same flow (sibling services, flags, enums, older tickets
   named in docblocks) and work out how the new rule combines with each: which runs first, which one wins,
   whether one undoes or blocks the other.
4. Check every invariant against the rows and sequences above.

Report a case only when it breaks an invariant or the requirements (CATEGORY invariant), or when the
requirements say nothing and the outcome would surprise the product owner (CATEGORY question). Each
report gives the concrete sequence of events and the final outcome.
```

### Agent 3: Design & maintainability

```
You review long-term maintainability against the project's conventions: breaking changes to public
interfaces, mechanisms or states the requirements do not need, duplication of an existing helper,
names that mislead, and inconsistency with the surrounding code. Do not suggest style changes the
project's conventions contradict.
```

### Agent 4: Test coverage

```
You review test adequacy: changed behaviour without a test, untested edge cases, tests that do not
assert what they claim, weakened assertions.

For every untested case, first answer: what does the code do in that case today, and is that what the
requirements and invariants want? If the outcome is wrong, report it as CATEGORY correctness or
invariant with the scenario, not as a test gap. Report a test-gap only when the current behaviour is
right and just unprotected.
```

### Agent 5: Security & performance

```
You review security (injection, authorization, data exposure, input validation at trust boundaries) and
performance (N+1 queries, unbounded loads, queries without a usable index, work repeated in loops) of
the change. Report only issues with a concrete trigger in this code. Say "No issues found" otherwise.
```

## Phase 3: Verify and classify

Before you write anything, go through the findings yourself:

1. **Reclassify by outcome.** A finding from any agent whose scenario ends in an outcome that breaks an invariant or
   requirement is a `correctness`/`invariant` finding, **at least HIGH**, whatever category or severity the agent gave
   it. This includes test gaps, design notes and questions.
2. **Verify** every CRITICAL, HIGH and MEDIUM finding by tracing its scenario through the code, file by file. When a
   test is cheap to write, run one that reproduces it. Mark each finding `CONFIRMED`, or drop it with one line saying
   why. Do not keep unverified findings above LOW.
3. **Deduplicate.** Merge findings about the same cause and note which agents raised them.
4. **Weigh materiality.** Downgrade findings whose trigger cannot happen in practice (say what rules it out) or whose
   cost is negligible. Keep a finding only if it would change what the author does.

## Phase 4: Report

Write `data/code-review/YYYY-MM-DD-HHMM.md` (local time; create `data/code-review/` if needed). Write the report in the
language the user is using.

```markdown
# Code Review Report

**Date**: YYYY-MM-DD
**Scope**: <branch> vs <base> (<base sha>), N files (+a / -d)[, plus uncommitted changes]
**Requirements**: <ticket ID and title, or "no ticket found">
**Verdict**: APPROVE | REQUEST CHANGES | NEEDS DISCUSSION — one sentence saying why

## Invariants

| # | Invariant | Holds? | Evidence |
|---|-----------|--------|----------|
| 1 | ... | ✅ / ❌ (finding id) / ❓ (unverified, why) | file:line or test name |

## Open PR comments

(only when the PR has review comments: each one, whether the code answers it, evidence)

## Findings that decide the verdict

(CRITICAL and HIGH, CONFIRMED, each with its scenario)

## Other findings

(MEDIUM in full; LOW as one line each)

## Questions for the product owner

## Dropped findings

(one line each: what was raised and why it does not hold)
```

Verdict:
- Any ❌ invariant, or any CRITICAL → REQUEST CHANGES.
- Any HIGH → REQUEST CHANGES unless the author accepted the trade-off explicitly in the ticket or PR.
- Open product questions whose answer changes the code → NEEDS DISCUSSION.
- Otherwise → APPROVE (with the MEDIUM/LOW as suggestions).

Then print the verdict, the invariant table, the deciding findings and the report path.

## Phase 5: Learn from human reviews

When a human reviewer later raises something this review missed or underrated, add an entry to
[references/past-misses.md](references/past-misses.md): the pattern in general terms, how to spot it, and why the
review missed it. Do not record the specific bug.
