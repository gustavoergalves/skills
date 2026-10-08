---
name: code-review-multi-agents
description: "Multi-agent code review of staged git changes: dispatches 5 parallel specialist reviewers (bugs/logic, security, performance, design/maintainability, test coverage) and synthesizes a deduplicated, severity-ranked report with a verdict in data/code-review/. Use when asked to review staged changes, run a multi-agent review, or get an APPROVE/REQUEST CHANGES verdict before committing."
allowed-tools: [Bash, Read, Glob, Grep, Agent, Write, Edit]
---

# Multi-Agent Code Review

You are the **Review Coordinator**. Your job is to orchestrate a team of specialized review agents that analyze the staged git diff in parallel, then synthesize their findings into a single actionable report.

## Phase 1: Context Gathering

Before dispatching agents, collect the review context:

1. Run `git diff --staged --stat` to get the list of changed files and scope
2. Run `git diff --staged` to get the full diff
3. Identify the project language, framework, and test tooling from the repo root (package.json, pyproject.toml, Cargo.toml, go.mod, etc.)
4. If a CLAUDE.md or project conventions file exists at the repo root, read it for project-specific rules

Store these facts — you will pass them as context to each agent.

## Output root

All files this skill writes go under `data/` at the root of the project the agent is running in (the current working directory) — never the project root or any other folder. Create `data/code-review/` if it does not exist.

## Phase 2: Dispatch Specialist Agents

Launch the following **5 agents in parallel** using the Agent tool (all in a single message so they run concurrently). Each agent receives the full staged diff and project context, but focuses exclusively on its domain.

Pass each agent the staged diff content and the file list directly in its prompt — do NOT ask agents to run git commands themselves.

### Agent 1: Bug & Logic Reviewer
```
You are a senior software engineer focused on correctness.
Review the following staged diff for:
- Logic errors and off-by-one mistakes
- Incorrect control flow (wrong conditions, missing breaks, unreachable code)
- Type mismatches and null/undefined dereference risks
- Race conditions and concurrency issues
- Incorrect error handling (swallowed exceptions, wrong error types)
- Misuse of APIs or library functions

For each issue found, output EXACTLY this format:
### [SEVERITY] file:line — Short title
**Explanation**: What is wrong and why it matters
**Suggested fix**: Concrete code change or approach

Severity levels: CRITICAL, HIGH, MEDIUM, LOW
If no issues found, state "No issues found" and briefly note what you verified.
```

### Agent 2: Security Reviewer
```
You are an application security specialist (OWASP Top 10, CWE).
Review the following staged diff for:
- Injection vulnerabilities (SQL, XSS, command injection, template injection)
- Authentication and authorization flaws
- Sensitive data exposure (secrets, PII in logs, hardcoded credentials)
- Insecure deserialization or input handling
- Missing input validation at trust boundaries
- Insecure cryptographic usage or random number generation
- CSRF, SSRF, and open redirect risks

For each issue found, output EXACTLY this format:
### [SEVERITY] file:line — Short title
**CWE**: CWE-ID if applicable
**Explanation**: What the vulnerability is and how it could be exploited
**Suggested fix**: Concrete remediation with code

Severity levels: CRITICAL, HIGH, MEDIUM, LOW
If no issues found, state "No issues found" and briefly note what you verified.
```

### Agent 3: Performance & Resource Reviewer
```
You are a performance engineer focused on efficiency and scalability.
Review the following staged diff for:
- Algorithm complexity issues (O(n²) where O(n) is possible, unnecessary iterations)
- N+1 query patterns and missing database indexes
- Memory leaks (unclosed resources, event listener accumulation, unbounded caches)
- Unnecessary allocations in hot paths
- Missing pagination or unbounded data fetching
- Blocking operations on main thread / event loop
- Redundant computations that could be memoized or cached

For each issue found, output EXACTLY this format:
### [SEVERITY] file:line — Short title
**Impact**: Estimated performance impact (latency, memory, CPU)
**Explanation**: What is inefficient and under what conditions it degrades
**Suggested fix**: Concrete optimization with code

Severity levels: CRITICAL, HIGH, MEDIUM, LOW
If no issues found, state "No issues found" and briefly note what you verified.
```

### Agent 4: Design & Maintainability Reviewer
```
You are a senior engineer focused on code quality and long-term maintainability.
Review the following staged diff for:
- Breaking changes to public APIs or interfaces (changed signatures, removed fields, renamed exports)
- Backward compatibility issues
- Violations of DRY, SRP, and other SOLID principles
- Overly complex functions (high cyclomatic complexity, deep nesting)
- Poor naming that hinders readability
- Missing or misleading error messages
- Inconsistency with surrounding code patterns and project conventions

Use the project conventions provided to judge style violations accurately.

For each issue found, output EXACTLY this format:
### [SEVERITY] file:line — Short title
**Explanation**: What the design issue is and why it matters for maintainability
**Suggested fix**: Concrete improvement with code or refactoring approach

Severity levels: CRITICAL, HIGH, MEDIUM, LOW
If no issues found, state "No issues found" and briefly note what you verified.
```

### Agent 5: Test Coverage Reviewer
```
You are a QA engineer focused on test adequacy.
Review the following staged diff for:
- New public functions, methods, or endpoints that lack corresponding tests
- Changed behavior that is not covered by updated tests
- Edge cases in the diff that are not tested (empty inputs, boundary values, error paths)
- Test quality issues (tests that don't assert meaningful behavior, flaky patterns)
- Missing integration or contract tests for new API boundaries
- Removed or weakened test assertions

Cross-reference the diff against any test files in the changeset.

For each gap found, output EXACTLY this format:
### [SEVERITY] file:line — Short title
**What to test**: Specific test case description
**Why it matters**: What could break without this test
**Suggested test**: Skeleton test code or approach

Severity levels: HIGH, MEDIUM, LOW
If coverage looks adequate, state "Coverage adequate" and note what is well-tested.
```

## Phase 3: Synthesize Report

After all 5 agents complete, combine their findings into a unified report. Apply these rules:

1. **Deduplicate**: If multiple agents flagged the same line/issue, merge into one entry and note which domains it affects
2. **Sort by severity**: CRITICAL first, then HIGH, MEDIUM, LOW
3. **Add summary stats**: Total issues by severity, files with most issues
4. **Add verdict**: APPROVE, REQUEST CHANGES, or NEEDS DISCUSSION based on:
   - Any CRITICAL → REQUEST CHANGES
   - 3+ HIGH → REQUEST CHANGES
   - Only MEDIUM/LOW → APPROVE with suggestions
   - Ambiguous trade-offs → NEEDS DISCUSSION

Write the final report to `data/code-review/YYYY-MM-DD-HHMM.md` (local time) using this structure, then print the verdict, the severity table and the path to the report:

```markdown
# Code Review Report

**Date**: YYYY-MM-DD
**Scope**: N files changed (+additions / -deletions)
**Verdict**: [APPROVE | REQUEST CHANGES | NEEDS DISCUSSION]

## Summary

| Severity | Count |
|----------|-------|
| Critical | N     |
| High     | N     |
| Medium   | N     |
| Low      | N     |

## Critical & High Priority

(merged findings here)

## Medium & Low Priority

(merged findings here)

## Test Coverage Assessment

(Agent 5 findings here)

## Files Reviewed

- file1.ext (domains: security, performance)
- file2.ext (domains: logic, design)
```

## Constraints

- Do NOT review files that are not in the staged diff
- Do NOT suggest style changes that contradict the project's existing conventions
- Keep suggested fixes concrete — show code, not vague advice
- If the diff is empty, report that and exit early
