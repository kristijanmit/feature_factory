# Feature Factory Template

A [Claude Code](https://claude.com/claude-code) project template that turns feature requests into shipped, tested, validated code through a multi-agent pipeline with human approval gates.

This repo contains **no application code** — it is the `.claude/` scaffolding you fork (or copy into an existing app) so that Claude Code can build features in your codebase using a repeatable factory process: research → story → spec → build → test → validate, with a human sign-off at every decision point.

## What's in the box

```
.claude/
├── settings.json                  # Wires the pre-commit safety hook
├── hooks/
│   └── pre-commit.sh              # Blocks commits containing sensitive files
├── skills/
│   ├── feature-factory/SKILL.md   # The orchestrator — runs the whole pipeline
│   └── build-with-tests/SKILL.md  # Shared build conventions used by all builder agents
└── agents/
    ├── codebase-researcher.md     # Phase 1 — maps the relevant code (read-only, Haiku)
    ├── story-writer.md            # Phase 2 — writes the user story (read-only)
    ├── spec-writer.md             # Phase 4 — writes the technical brief (read-only)
    ├── backend-builder.md         # Phase 6 — services, routes, workers, migrations + unit tests
    ├── frontend-builder.md        # Phase 7 — components, pages, hooks + component tests
    ├── test-verifier.md           # Phase 8 — acceptance tests against the story
    ├── implementation-validator.md# Phase 9 — gap report vs. story & brief (read-only)
    └── pr-reviewer.md             # On demand — reviews PRs against the project checklist
```

## How the pieces connect

### The pipeline (feature-factory skill)

The `feature-factory` skill is the orchestrator. When you ask Claude Code to *"build a feature"* / *"ship a feature"* / *"run the full chain"*, it invokes the seven pipeline agents **in order, synchronously**, carrying each phase's output forward into the next agent's prompt (agents start cold — they never see your conversation):

```
feature idea
    │
    ▼
[1] codebase-researcher ── maps files, patterns, similar features, risks
    │
    ▼
[2] story-writer ────────── user story + acceptance criteria + edge cases
    │
    ▼
[3] 🚦 HUMAN GATE ───────── approve / request changes / reject the story
    │
    ▼
[4] spec-writer ─────────── technical brief (data model, API, frontend, tests, files)
    │
    ▼
[5] 🚦 HUMAN GATE ───────── approve / request changes / reject the brief
    │
    ▼
[6] backend-builder ─────── services, routes, workers, migrations + unit tests
    │                        (its summary becomes the API contract)
    ▼
[7] frontend-builder ────── components, pages, hooks + tests, consuming that contract
    │
    ▼
[8] test-verifier ───────── acceptance tests for every criterion in the story
    │
    ▼
[9] implementation-validator ── gaps vs. story & brief, grouped by severity
    │
    ▼
[10] critical-gap loop ──── critical findings route back to the right builder,
    │                       then re-verify + re-validate (max 3 loops)
    ▼
[11] 🚦 HUMAN GATE ──────── final review; PR is opened only with your explicit OK
```

Key design rules baked into the pipeline:

- **Three human gates.** The story, the brief, and the final wrap-up all require your explicit approval via an interactive question. Silence is never treated as approval, and a rejected story or brief stops the chain cleanly.
- **Backend before frontend, never parallel.** The backend builder's summary documents the real API contract; the frontend builder consumes it verbatim and reports mismatches instead of patching around them.
- **Builders are fenced.** `backend-builder` never touches components, pages, or hooks; `frontend-builder` never touches services, routes, workers, or migrations.
- **Verifiers never fix.** `test-verifier` edits only test files; `implementation-validator` and `pr-reviewer` edit nothing at all. Failures are routed back to the builders, capped at 3 fix loops before escalating to you.

### The build-with-tests skill

`backend-builder`, `frontend-builder`, and `test-verifier` all load `.claude/skills/build-with-tests/SKILL.md` before writing code. It enforces the process every change follows:

1. Read `CLAUDE.md` and the technical brief first.
2. Study 2–3 similar existing features and match their patterns (reuse helpers, don't reinvent).
3. Implement in small steps with unit tests alongside the code — success path, a validation failure, and one edge case per behaviour, minimum.
4. Run typecheck, lint, and the test suite before reporting done.
5. Return a summary of files changed, patterns reused, and exact check results.

It also carries a **Codebase conventions** section (naming, where business logic lives, error handling, test layout, tenant isolation, timezone rules) that you are expected to edit to match your project — see below.

### The commit safety hook

`.claude/settings.json` registers a `PreToolUse` hook: every time Claude runs a `git commit`, `.claude/hooks/pre-commit.sh` first checks the staged files and **blocks the commit** if any match `.env`, `*.key`, `*.pem`, `secrets.json`, or `creds.md`. Extend the grep pattern in that script for anything else you never want committed.

### CLAUDE.md — the missing piece you must supply

Nearly every agent's first instruction is *"Read CLAUDE.md"*: the spec-writer bases the brief on it, the builders get their typecheck/lint/test commands from it, and the validator and PR reviewer judge pattern consistency against it. **This template ships without one** — creating it is your first post-fork task.

## After forking this repo

1. **Bring the code and the config together.** Either build your application inside this repo, or copy the `.claude/` directory into an existing project. The `.claude/` directory must sit at the repo root of the codebase the agents will work on.

2. **Create `CLAUDE.md` at the repo root.** Run `/init` in Claude Code, or write it by hand. The pipeline depends on it containing at minimum:
   - The stack (language, framework, database, test runner).
   - The exact **typecheck, lint, and test commands** — builders run these verbatim.
   - Architecture rules (where business logic lives, folder layout).
   - A "don't do" list (forbidden dependencies, patterns to avoid).

3. **Update the conventions in `build-with-tests`.** The *Codebase conventions* section of `.claude/skills/build-with-tests/SKILL.md` ships with TypeScript-flavoured defaults (kebab-case files, `services/` + `workers/` layout, `__tests__/` folders, `test/builders/` for entity setup). Rewrite it to match your actual stack and layout — the skill itself says to trust the codebase and fix the list when they disagree.

4. **Adjust the builder boundaries if your layout differs.** The agent definitions in `.claude/agents/` assume a services/routes/workers backend and a React components/pages/hooks frontend. If your project is structured differently (a monorepo, a non-React frontend, backend-only), edit `backend-builder.md` and `frontend-builder.md` so their "only edit these files" rules match reality. The orchestrator already skips the backend or frontend phase when the brief has no work for it.

5. **Extend the pre-commit hook if needed.** Add your own sensitive-file patterns to `.claude/hooks/pre-commit.sh`, and keep it executable (`chmod +x`) if your fork loses the permission bit.

6. **Tune models and tools (optional).** Each agent's frontmatter pins its model (`haiku` for cheap research, `sonnet` for the rest) and its tool allowlist. Adjust to taste — e.g. a stronger model for the spec-writer on a complex codebase.

## How to use it day to day

**Build a feature end to end** — in a Claude Code session, ask:

> Build a feature: users can opt out of the weekly digest email.

or invoke the skill directly with `/feature-factory`. Claude orchestrates the pipeline above; your job is to review at the three gates:

- **Gate 1 — story:** does the user story capture what you actually want? Acceptance criteria become the tests, so fix them here, cheaply.
- **Gate 2 — brief:** is the technical approach right? Rejecting the brief keeps the approved story so you can retry with a different approach.
- **Gate 3 — final review:** you see what was built, the acceptance test results, and any remaining important/minor findings, then decide whether a PR gets opened. Nothing is pushed or merged without your explicit go-ahead.

**Use agents individually** — every agent also works standalone:

> Use the codebase-researcher to map how invoice creation works today.
> Use the pr-reviewer to review PR #142.

`pr-reviewer` is deliberately outside the pipeline: run it on any open PR (including ones written by humans) and it reports critical/important/minor findings with file-and-line citations against the checklist — scope, tests, security and tenant safety, architecture, documentation. It never edits, merges, or approves anything.

**Small changes** don't need the factory. For a one-file tweak, just ask Claude Code directly — the `build-with-tests` skill still kicks in for anything that produces production code, so tests and checks still happen.

## Agent reference

| Agent | Phase | Writes | Model | Tools |
|---|---|---|---|---|
| `codebase-researcher` | 1 | nothing (report only) | haiku | Read, Grep, Glob |
| `story-writer` | 2 | nothing (story only) | sonnet | Read |
| `spec-writer` | 4 | nothing (brief only) | sonnet | Read, Grep, Glob |
| `backend-builder` | 6 | backend code + unit tests | sonnet | Read, Edit, Write, Bash |
| `frontend-builder` | 7 | frontend code + tests | sonnet | Read, Edit, Write, Bash |
| `test-verifier` | 8 | acceptance tests only | sonnet | Read, Edit, Write, Bash |
| `implementation-validator` | 9 | nothing (gap report) | sonnet | Read, Grep, Glob |
| `pr-reviewer` | on demand | nothing (review report) | sonnet | Read, Grep, Glob, Bash (read-only git) |
