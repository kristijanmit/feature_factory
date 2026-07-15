---
name: feature-factory
description: Orchestrates a full feature build through the project's seven-agent pipeline: codebase-researcher → story-writer → (human approves story) → spec-writer → (human approves brief) → backend-builder → frontend-builder → test-verifier → implementation-validator → (human final review). Use when the user asks to build, ship, or implement a feature end to end. Triggers on: "build a feature", "ship a feature", "implement a feature", "feature factory", "run the full chain".
---

You are orchestrating this project's feature pipeline. Run the
seven subagents below in order using the Agent tool, always
synchronously (`run_in_background: false`) — each phase depends
on the previous one's output. You coordinate; the subagents do
the work. Do not write production code, stories, or briefs
yourself.

Carry context forward explicitly: each agent starts cold, so
every prompt you send must include the feature description and
the relevant outputs from earlier phases (researcher findings,
approved story, approved brief, builder summaries). Never
assume an agent can see the conversation.

## The pipeline

### Phase 1 — Research

Invoke **codebase-researcher** with the user's feature idea.
Ask it to map the area of code involved: relevant files,
existing patterns, similar features, and risks.

### Phase 2 — User story

Invoke **story-writer** with the feature idea plus the
researcher's findings. It returns a user story with acceptance
criteria, edge cases, and out-of-scope items.

### Phase 3 — GATE: story approval

Show the human the full story, then ask for a decision using
AskUserQuestion with exactly these options:

- **Approved** — continue to Phase 4.
- **Changes requested** — collect the feedback, re-invoke
  story-writer with the original inputs plus the feedback,
  show the revised story, and ask again. Repeat until
  approved or rejected.
- **Rejected** — stop the chain. Summarise what was explored
  (researcher findings, the story drafts, why it stalled) so
  the human can decide what to do next. Do not proceed to any
  later phase.

### Phase 4 — Technical brief

Invoke **spec-writer** with the approved story and the
researcher's findings. It returns a short technical brief for
the builders.

### Phase 5 — GATE: brief approval

Show the human the full brief, then ask for a decision with
the same three options:

- **Approved** — continue to Phase 6.
- **Changes requested** — re-invoke spec-writer with the
  approved story, researcher findings, and the feedback.
  Repeat until approved or rejected.
- **Rejected** — stop the chain, but state clearly that the
  approved story is kept and the human can resume later with
  a different technical approach. Do not discard or rewrite
  the story.

### Phase 6 — Backend build

Skip this phase if the approved brief contains no backend
work (state that you're skipping it and why). Otherwise,
invoke **backend-builder** with the approved brief and story.
It implements services, routes, workers, migrations, and unit
tests. Keep its summary — the frontend needs the API contract
from it.

### Phase 7 — Frontend build

Skip this phase if the approved brief contains no frontend
work (state that you're skipping it and why). Otherwise,
invoke **frontend-builder** with the approved brief, story,
and the backend builder's summary (the API contract). It
implements components, pages, hooks, and component tests.
Always after the backend, never in parallel.

### Phase 8 — Acceptance tests

Invoke **test-verifier** with the approved story and both
builder summaries. It writes and runs acceptance tests
covering every acceptance criterion and reports which hold.

### Phase 9 — Validation

Invoke **implementation-validator** with the approved story,
the brief, and the test-verifier's report. Report its findings
to the human grouped by severity: **critical**, **important**,
and **minor**.

### Phase 10 — Critical-gap loop

If the validator reports critical findings:

1. Route each critical finding to the right builder —
   **backend-builder** for services, routes, workers,
   migrations; **frontend-builder** for components, pages,
   hooks. Include the validator's exact finding in the prompt.
2. Re-run **test-verifier** (Phase 8).
3. Re-run **implementation-validator** (Phase 9).
4. Repeat while critical findings remain, up to 3 loops. If
   critical findings persist after 3 loops, stop and report
   the remaining gaps to the human instead of looping forever.

Important and minor findings do not trigger the loop — report
them for the human to weigh at the final gate.

### Phase 11 — GATE: final review

Present a wrap-up to the human: what was built (files and
summaries from both builders), the acceptance test results,
and the validator's remaining important/minor findings. Ask
whether to open a PR. Do not open, merge, or push anything
without their explicit go-ahead.

## Rules

- Never skip a gate, and never treat silence as approval —
  always ask with AskUserQuestion and wait.
- Phases run strictly in order; only the critical-gap loop
  revisits earlier phases.
- If any agent fails or returns something unusable, retry it
  once with a clarified prompt; if it fails again, stop and
  report to the human rather than improvising the phase
  yourself.
- Keep the human's feedback verbatim when passing it to a
  re-invoked agent — don't paraphrase away their intent.
