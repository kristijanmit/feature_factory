---
name: build-with-tests
description: Use this skill when implementing a feature or extending existing behaviour — any task that produces production code (endpoint, service, worker, migration, component). Reads CLAUDE.md and the technical brief first, matches patterns from existing features, writes production code with unit tests alongside it, and runs the project's typecheck, lint, and test commands at the end. Triggers on: "build", "implement", "add", "extend", "ship the feature".
---

# Build a feature with tests

Follow this process for every production code change, start to
finish. Do not skip steps because a change "looks small."

## 1. Read the ground rules before writing code

1. Read `CLAUDE.md` in full: the stack, the architecture rules,
   the "don't do" list, and the typecheck / lint / test commands.
2. Read the technical brief for this feature and stay inside its
   scope. If there is no brief, confirm the scope with the user
   (or the invoking agent) before writing code.
3. If CLAUDE.md and the brief conflict, stop and report the
   conflict instead of picking a side silently.

## 2. Study 2-3 similar existing features

Before writing anything new, find two or three existing features
with the same shape as the one you're building (same layer:
service, route, worker, component) and read them end to end,
including their tests.

Match what you find:

- Reuse existing helpers, services, and templates instead of
  writing new ones. If `services/email.ts` already has
  `sendTemplatedEmail()`, add a template and call it — do not
  write a new mailer.
- Copy the structure of the closest existing feature: file
  location, naming, registration (e.g. how workers are added to
  `workers/index.ts`), and error handling.
- If no similar feature exists, say so explicitly and propose a
  structure before building it.

## 3. Implement in small steps, tests alongside the code

Break the feature into the smallest coherent steps you can.
For each step:

- Write the production code.
- Write a unit test that covers the new behaviour.
- Run that test and confirm it passes before moving on.

Tests are part of the feature, not a follow-up task. This is
normal good engineering, not a strict red-green TDD loop — write
the code and its tests together, in whichever order helps you
think.

- Every new service function, route, and worker gets unit tests
  in the same change.
- Per behaviour, cover at minimum: the success path, a
  validation failure, and one edge case (empty inputs,
  boundaries, retries, deduplication — whichever applies). Add
  any failure paths the brief calls out.
- Base new tests on the test files of the similar features you
  read in step 2: same file location, same setup/teardown
  helpers, same mocking approach.

## 4. Verify before finishing

Run, using the commands from CLAUDE.md:

1. Typecheck
2. Lint
3. The test suite

All three must pass before you report the work as done. If a
failure is pre-existing and unrelated to your change, do not fix
code outside your scope — report it as pre-existing, with the
failing test's name.

Report results plainly: what passed, what failed, exact counts.
Never describe the work as complete if any check failed.

## 5. Return a short summary

Finish with a summary containing:

- Files added / changed.
- Patterns and helpers reused.
- Typecheck / lint / test results (exact counts).
- Any rule you would suggest adding to CLAUDE.md — a convention
  you had to infer from the code that nothing documents today.

## 6. Codebase conventions

<!-- Keep this section current: when a convention changes or a
     new one is established, update it here. If the codebase and
     this list disagree, trust the codebase and fix this list. -->

**Naming**
- Files: kebab-case (`weekly-digest.ts`); test files mirror the
  source file name (`digest.ts` → `digest.test.ts`).
- Functions: verb-first camelCase (`buildWeeklyDigest`,
  `sendTemplatedEmail`).
- Migrations: date-prefixed, snake_case description
  (`20260714_add_digest_optout.ts`).

**Where business logic lives**
- Business logic goes in `services/`. Routes validate input,
  call a service, and shape the response — nothing more.
- Background jobs live in `workers/`, are registered in
  `workers/index.ts`, and must be idempotent. Workers call
  services; they do not contain business logic themselves.

**Error handling**
- Follow the error pattern of the similar features from step 2;
  do not invent a new error type or handling style per feature.
- Failures must be explicit: no silently swallowed exceptions,
  no bare `catch` that only logs.

**Tests**
- Unit tests live next to the code they cover, in a `__tests__/`
  folder (`services/__tests__/digest.test.ts`) — or in `tests/`
  if that is the existing pattern where you're working.
- One test file per source file; group cases with `describe` per
  function.
- Use builders from `test/builders/` for any entity setup; do
  not hand-construct entities inline in tests.
- Mock at the boundary (email delivery, external APIs, clock),
  not internal services.

**Always**
- Tenant isolation: every query and job must respect tenant
  boundaries; call it out in your summary if a feature has none.
- Timezones: handle explicitly; store UTC, convert at the edge.

## Rules

- Do not refactor unrelated code.
- Do not change files outside the agreed scope (the brief, or
  the scope confirmed in step 1).
- Do not add new dependencies without explicit instruction.
- If you cannot make the tests pass without violating one of
  these rules, stop and report the conflict instead of working
  around it.
