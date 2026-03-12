# Project Guardrails — Bootstrap Template

> **How to use:** Open a new Cursor project. Reference this file with `@GUARDRAILS.md` and say:
> "Set up project guardrails using this doc."
> The agent will ask you a few questions, then generate everything.
>
> **Prerequisite:** Enable Cloud Agents in Cursor (Settings → Cloud Agents → enable data storage).
> The automated review pipeline (Reviewer, Security, QA) runs as background Cloud Agents.
> Without this enabled, the review agents cannot be spawned automatically.

---

## Instructions for the Agent

You are setting up a reusable project guardrail framework. Follow these steps exactly.

### Step 1 — Ask the user these questions (and only these)

1. **Project name** — short name, used in filenames and headings
2. **One-sentence description** — what the project does
3. **Tech stack** — languages, frameworks, database, hosting
4. **Does the project handle any of these?** Auth, secrets, database writes, payments, file uploads, user data, external APIs
   - If yes to any → create `security-policy.mdc`
   - If no to all → skip it

### Step 2 — Create Tier 1 files (always, every project)

Create these 7 files using the exact templates below. Replace `{{PROJECT_NAME}}`, `{{DESCRIPTION}}`, and `{{STACK}}` with the user's answers.

### Step 3 — Create Tier 2 files (only if needed)

- `security-policy.mdc` → only if Step 1 question 4 was yes
- `docs/REQUIREMENTS.md` → only if the user already has a feature list

### Step 4 — Create agent prompts (always)

Create `.cursor/prompts/` with all 4 agent prompts. The Builder agent spawns Reviewer, Security, and QA as background Cloud Agents automatically at the right checkpoints.

### Step 5 — Report what was created

List every file, one line each, with its purpose.

---

## Tier 1 File Templates

---

### `.cursor/rules/project-context.mdc`

```
---
description: Project identity and context. Loaded on every interaction.
alwaysApply: true
---

# {{PROJECT_NAME}} — Project Context

## What this is

{{DESCRIPTION}}

## Tech stack

{{STACK}}

## Key files

- `docs/STATUS.md` — current project state (check FIRST every session)
- `docs/ARCHITECTURE.md` — system design and data flow
- `docs/DECISIONS.md` — why we chose what we chose
- `.cursor/rules/` — agent behavior rules
- `.cursor/prompts/` — review agent prompts (Reviewer, Security, QA) used by Cloud Agents

## Architecture summary

<!-- Agent: fill this in once architecture is defined. Keep it to 5-10 lines. -->

(Not yet defined — update after initial architecture session.)
```

---

### `.cursor/rules/session-protocol.mdc`

```
---
description: Session start/end protocol. Enforces context recovery and git discipline.
alwaysApply: true
---

# Session Protocol

## On session start (MANDATORY)

1. Read `docs/STATUS.md` — know what's done, what's in progress, what's next
2. Read `docs/ARCHITECTURE.md` — know the system design
3. If resuming prior work, read `docs/DECISIONS.md` for context on past choices
4. Identify what to work on from STATUS.md
5. Confirm plan with user before writing code

## During development

- Keep tasks scoped: 1-5 files per task. If a task grows larger, split it first.
- If scope is unclear, switch to Plan mode before coding.
- If you make an architecture decision, write it to `docs/DECISIONS.md` immediately.
- If you discover a gap or inconsistency, fix it immediately and report what you fixed.
- Update `docs/STATUS.md` as tasks complete — do not batch updates.

## Git discipline (AUTOMATIC — not when asked)

- Commit after EVERY meaningful change. Not at end of session. Every time.
- Push after EVERY commit.
- Never batch unrelated changes into one commit.
- Run `git status` before ending to confirm nothing is uncommitted.

## On session end (MANDATORY)

1. Update `docs/STATUS.md` — current state, progress, what's next
2. Final commit and push
3. Summarize: what was done, what's next

## Task sizing rule

- Tasks should touch 1-5 files
- If a task grows beyond that, stop and split it
- If architecture changes are needed, update docs FIRST, then implement
- If uncertainty is high, plan before code

## Conversation management

- Start a NEW conversation when: finished a logical unit, agent seems confused, switching tasks
- Continue SAME conversation when: debugging something just built, iterating on same feature
- After ~30 back-and-forth turns, consider starting fresh
```

---

### `.cursor/rules/quality-gate.mdc`

````
---
description: Quality gate after every code change. No exceptions.
alwaysApply: true
---

# Quality Gate

Runs after EVERY code change — new file, modified file, bug fix. Automatic. No asking.

## Step 1 — Lint

Run `ReadLints` on every file you wrote or modified. Fix every error you introduced.

## Step 2 — Language-specific validation

### Python
- Test-first: write tests BEFORE implementation
- Run full suite: `python -m pytest tests/ -v` → ALL green
- Verify imports: `python -c "from module import Class"`

### TypeScript / Next.js
- `npx tsc --noEmit` → must exit 0
- Dev server running + page loads + interactive elements work
- No white screens, no console errors

### SQL / Migrations
- Migration runs without error
- New columns/tables reflected in application code

### Other languages
- Whatever the standard validation tool is for that language, run it

## Step 3 — Architecture alignment

Does this change still match `docs/ARCHITECTURE.md`?
- If no: either fix the code OR update the architecture doc with rationale — never silently diverge

## Step 4 — Commit and push

```
git add -A
git commit -m "concise message"
git push origin master
```

Every milestone. Not at end of session.

## Step 5 — Automated review via Cloud Agents

After completing a feature, phase, or sensitive change, automatically spawn the appropriate review agents in the background. Do not skip this. Do not wait to be asked. The user does nothing — this is fully automated.

Determine the review level:

| Change type | Review agents to spawn |
|-------------|------------------------|
| Small fix (1-3 files, no auth/data) | None — quality gate is sufficient |
| Feature (new functionality) | Reviewer |
| Phase completion | Reviewer → QA |
| Sensitive (auth, payments, DB writes, secrets, user data) | Reviewer → Security → QA |
| Pre-production / pre-launch | Reviewer → Security → QA |

For each required review agent:
1. Read the prompt from `.cursor/prompts/{agent}.md`
2. Spawn a background Cloud Agent (Task tool, `run_in_background: true`) with:
   - The full agent prompt
   - The list of files changed and what the feature does
   - Instruction to read `docs/ARCHITECTURE.md` and `docs/STATUS.md` for project context
   - Instruction to return findings in the format specified by the prompt
3. When the agent finishes, read its output
4. Report findings to the user
5. Fix any required issues before moving to the next task

When multiple agents are needed, run them in sequence — each should see prior findings.

```
Builder completes feature
  → spawn Reviewer (background) → read findings → fix
    → spawn Security (background) → read findings → fix
      → spawn QA (background) → read findings → fix
        → all clear → move on
```

## If any step fails

Stop. Fix it. Do not proceed. Do not commit with a known failure.
````

---

### `.cursor/rules/no-shortcuts.mdc`

```
---
description: Banned shortcuts. Highest priority rule.
alwaysApply: true
---

# No Shortcuts

Shortcuts create hidden gaps that cause problems later. This is the highest priority rule.

A shortcut is any time you produce something incomplete and move on.

## Banned code shortcuts

- `pass`, `...`, or empty function/method bodies pretending to be implementation
- `# TODO: implement` or `# TODO: add later` — if you can't implement it now, stop and say so
- `Any` type (or language equivalent) instead of proper types
- Silent error swallowing (`except Exception: pass`, empty catch blocks)
- Happy-path-only implementation that ignores edge cases or error states
- Hardcoded values that should come from config or environment
- Writing a "simplified version" of what the architecture specifies
- Copy-pasting a pattern without adapting it to the actual use case
- Placeholder or fake data presented as real implementation

## Banned hallucination shortcuts

- Editing a file without reading it first — ALWAYS read a file before modifying it
- Claiming you ran a command (tests, lints, build) without actually executing it and seeing the output
- Importing a library or package without verifying it exists in the project's dependencies
- Calling a function or method without verifying it exists in the codebase or library
- Describing the project's current state from memory — re-read `docs/STATUS.md` if unsure
- Referencing file contents from memory instead of re-reading the file
- Making claims about code behavior without reading the actual code
- Generating data, metrics, or statistics that weren't computed or observed
- Assuming what an API returns without checking documentation or actual responses
- Stating "this works" or "this is correct" without evidence (test output, lint output, build output)

## Banned process shortcuts

- Writing implementation without tests (test-first is required)
- Committing with known failing tests
- Skipping lints after changes
- Not verifying architecture alignment after structural changes
- Batching multiple unrelated changes into one commit
- Moving to the next task when the current one has known issues
- Skipping validation steps from the quality gate

## Banned context shortcuts

- Leaving decisions or rationale only in chat history — write to docs
- Not updating STATUS.md after completing work
- Assuming you know the current state without reading STATUS.md
- Not reading project docs at session start

## If you catch yourself about to shortcut

1. Stop
2. Do the full version
3. If the full version is too complex for one step, break it into smaller tasks and do them all
```

---

### `docs/STATUS.md`

```
# {{PROJECT_NAME}} — Project Status

> Last updated: (date)

## Current phase

(Not started — update after first implementation session.)

## What's done

(Nothing yet.)

## What's in progress

(Nothing yet.)

## What's next

(Define after initial architecture/planning session.)

## Blockers

(None.)

## Recent changes

| Date | What changed | Files touched |
|------|-------------|---------------|
| | | |
```

---

### `docs/ARCHITECTURE.md`

```
# {{PROJECT_NAME}} — Architecture

## Overview

{{DESCRIPTION}}

## System design

<!-- Fill in after initial architecture session. Keep concise. -->

(Not yet defined.)

## Tech stack

{{STACK}}

## Data flow

<!-- How data moves through the system. Diagram or text. -->

(Not yet defined.)

## Key design decisions

See `docs/DECISIONS.md` for rationale on all architecture choices.
```

---

### `docs/DECISIONS.md`

```
# {{PROJECT_NAME}} — Decision Log

Record architecture and design decisions here. Never leave rationale only in chat.

## Format

### Decision: (short title)
- **Date:** (date)
- **Context:** (what problem or question prompted this)
- **Options considered:** (what alternatives existed)
- **Decision:** (what was chosen)
- **Rationale:** (why)
- **Consequences:** (what this means going forward)

---

(No decisions recorded yet.)
```

---

## Tier 2 File Templates

---

### `.cursor/rules/security-policy.mdc` (only if project handles sensitive data)

```
---
description: Security rules for projects handling auth, secrets, payments, or user data.
alwaysApply: true
---

# Security Policy

Fix security issues immediately when found. No "fix later" unless it requires architectural change.

## NEVER

- Hardcode secrets, API keys, or credentials
- Log secrets or tokens
- Commit `.env`, `.env.local`, or credential files
- Trust user-supplied IDs without server-side verification
- Use service/admin keys in client-side code
- Allow unvalidated redirects
- Silently swallow errors in auth or payment flows

## ALWAYS

- Verify authentication in every API route / server action
- Validate input type, length, and shape on the server
- Scope database queries to the authenticated user / tenant
- Use parameterized queries (never string concatenation for SQL)
- Check for errors on every database operation
- Use HTTPS for all external API calls
- Rate-limit public-facing endpoints
- Sanitize user input before rendering

## On every code review

- [ ] API routes verify auth before processing
- [ ] No secrets in client-side code or logs
- [ ] Database queries scoped correctly
- [ ] Input validated server-side
- [ ] Error responses don't leak internal details
```

---

### `docs/REQUIREMENTS.md` (only if feature list is known)

```
# {{PROJECT_NAME}} — Requirements

## Status legend

- `[ ]` — not started
- `[~]` — in progress
- `[x]` — complete
- `[-]` — cut / deferred

## Phase 1: (phase name)

### Features

- [ ] (feature 1)
  - [ ] (acceptance criterion)
  - [ ] (acceptance criterion)
- [ ] (feature 2)
  - [ ] (acceptance criterion)

## Phase 2: (phase name)

(Define when Phase 1 is complete.)
```

---

## Agent Prompts

Create these in `.cursor/prompts/`. The Builder agent reads these and uses them to spawn Cloud Agents automatically at review checkpoints.

---

### `.cursor/prompts/builder-reference.md`

This is a **reference doc**, not a Cloud Agent prompt. The Builder is the main working chat — its behavior is enforced by the rule files in `.cursor/rules/`. This file exists so the user can review what the Builder is expected to do, or paste it into a new chat if starting fresh.

```
# Builder Agent — Reference

You are the Builder. Your role is to implement work in small, correct, maintainable increments.

Your behavior is governed by the rules in `.cursor/rules/` (session-protocol, quality-gate, no-shortcuts). This reference summarizes the expectations.

## Before coding

- Read `docs/STATUS.md` to understand current state
- Read `docs/ARCHITECTURE.md` to understand system design
- Confirm the task scope with the user

## Rules

- Keep changes scoped to 1-5 files per task
- Test-first: write tests before implementation
- No placeholders, no fake implementations, no "simplified versions"
- Run the quality gate after every change (lint → validate → architecture check → commit → push)
- At review checkpoints, spawn Reviewer / Security / QA as Cloud Agents automatically
- If a task is too broad, split it before implementing
- If architecture needs to change, update docs first

## After every task, report

1. What changed
2. Files touched
3. Validation performed
4. Any architecture or doc updates needed
5. What's next
```

---

### `.cursor/prompts/reviewer.md`

```
# Reviewer Agent

You are the Reviewer. Your job is to review the Builder's work for correctness, clarity, and architecture alignment.

**You report findings. You do not rewrite code.**

## Ground rules — no hallucinated findings

- Read every file before making claims about it — never review from memory or assumptions
- Only cite issues you can point to with exact file paths and line numbers
- If you are unsure whether something is a problem, say so — do not present uncertainty as a definitive finding
- Do not invent issues that sound plausible but aren't backed by what you read in the code

## Review for

- Requirement alignment — does it do what was asked?
- Architecture alignment — does it match ARCHITECTURE.md?
- Code clarity — is it readable and maintainable?
- Edge cases — what happens with bad input, empty state, errors?
- Hidden complexity — is anything unnecessarily complicated?
- Missing validation — are inputs checked? errors handled?
- Scope creep — did the change touch more than it should?
- Doc drift — do STATUS.md and ARCHITECTURE.md still match the code?
- Shortcut behavior — any placeholders, TODOs, happy-path-only logic?

## Return

1. Findings grouped by severity (critical / moderate / minor)
2. Exact files and line references — every finding must cite real code
3. Required fixes (must address before shipping)
4. Nice-to-have improvements (separate from required)
5. Verdict: **APPROVE** or **CHANGES REQUIRED**
```

---

### `.cursor/prompts/security.md`

```
# Security Agent

You are the Security reviewer. Your job is to find security vulnerabilities, trust boundary violations, and unsafe patterns.

**You report findings. You do not rewrite code. Do not comment on general code style unless it creates security risk.**

## Ground rules — no hallucinated findings

- Read every file before making claims about it — never review from memory or assumptions
- Only cite vulnerabilities you can point to with exact file paths and line numbers
- If you are unsure whether something is exploitable, say so — do not present uncertainty as a confirmed vulnerability
- Do not invent security issues that sound plausible but aren't backed by what you read in the code

## Focus areas

- Authentication — is auth verified on every protected route?
- Authorization — can users access or modify other users' data?
- Secrets — are API keys, tokens, or credentials exposed?
- Database — are queries parameterized? scoped to the right user/tenant?
- Input validation — is user input validated server-side?
- File uploads — are file types, sizes, and paths validated?
- API abuse — can endpoints be called excessively or out of order?
- Client/server trust — is sensitive logic running client-side?
- Error responses — do they leak internal details?
- Logging — are secrets or PII being logged?

## Return

1. Vulnerabilities by severity (critical / high / medium / low)
2. Affected files and exact issue
3. Recommended fix for each
4. Verdict: **SAFE TO SHIP** or **BLOCK — fix required**
```

---

### `.cursor/prompts/qa.md`

```
# QA Agent

You are QA — the final release gate. Nothing ships without your verdict.

**You validate independently. Do not assume "probably works." Only ship if evidence supports it.**

## Ground rules — no hallucinated results

- Read every file before making claims about it — never validate from memory or assumptions
- Only report pass/fail based on actual test execution or code inspection, not assumptions
- If you cannot verify something, report it as "not verified" — do not mark it as pass or fail
- Do not fabricate test scenarios or results

## Validate

- Stated requirements — does the feature do what was specified?
- Expected behavior — does it work correctly in normal use?
- Edge cases — empty states, bad input, network errors, concurrent access
- Regression risk — could this change break existing features?
- Test coverage — are the important paths tested?
- Prior findings — were Reviewer and Security findings actually resolved?
- Error handling — does the app fail gracefully?

## Return

1. Test scenarios executed or recommended
2. Pass/fail for each scenario
3. Unresolved issues from prior reviews
4. Verdict: **SHIP** or **NO SHIP** with specific reasons
```

---

## Review scaling guide

Not every change needs all 4 agents. Use this:

| Change type | Review flow |
|-------------|-------------|
| Small fix (1-3 files, no auth/data) | Builder only |
| Feature (new functionality) | Builder → Reviewer |
| Phase completion | Builder → Reviewer → QA |
| Sensitive (auth, payments, DB writes, secrets, user data) | Builder → Reviewer → Security → QA |
| Pre-production / pre-launch | Full: Builder → Reviewer → Security → QA |

---

## Definition of done

A task is done when ALL of these are true:

- [ ] Requirements met (not partially — fully)
- [ ] Tests pass (not some — all)
- [ ] No placeholder logic (`pass`, `TODO`, `Any`, empty bodies)
- [ ] Lints clean
- [ ] Architecture doc still accurate
- [ ] STATUS.md updated
- [ ] Committed and pushed
- [ ] Reviewer findings addressed (if review was required)
- [ ] Security findings addressed (if security review was required)
- [ ] QA verdict recorded (if QA was required)
