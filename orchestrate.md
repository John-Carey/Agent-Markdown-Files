---
name: orchestrate
description: Orchestrate a full feature development pipeline by chaining specialized agents (planner → architect → implementation → code-reviewer → database-reviewer → doc-updater) with markdown handoff documents between stages. Use this skill whenever the user requests a new feature, a significant refactor, or any multi-step development task — especially if they say "orchestrate", "run the pipeline", "full workflow", "build this feature end to end", or ask for planning, implementation, and review together. Also use when resuming a previously started pipeline from an existing handoff document.
---

# Orchestrate

Coordinate the specialized agents in this repo as a sequential pipeline, passing context between stages via **handoff documents**. Each stage completes its work, writes a handoff file for the next stage, and the pipeline continues until the feature is planned, designed, implemented, reviewed, and documented.

## The Pipeline

```
planner → architect → [implementation] → code-reviewer → database-reviewer → doc-updater
```

| Stage | Agent | Role | Handoff produced |
|-------|-------|------|------------------|
| 1 | `planner` | Break the request into a phased, actionable implementation plan | `01-plan.md` |
| 2 | `architect` | Design the system changes, document trade-offs and decisions | `02-architecture.md` |
| 3 | *(you — the coding agent)* | Implement the code following the plan and architecture | `03-implementation.md` |
| 4 | `code-reviewer` | Review all changes for quality, security, and maintainability | `04-code-review.md` |
| 5 | `database-reviewer` | Review schema/query changes for performance and security | `05-database-review.md` |
| 6 | `doc-updater` | Update codemaps, READMEs, and guides to match the new code | `06-docs.md` (final summary) |

**Stage 3 is not a custom agent.** The coding agent running this skill (Claude Code, Codex, Cursor, etc.) performs the implementation itself, using the plan and architecture handoffs as its instructions.

## Handoff Documents

Handoffs are how context flows through the pipeline without re-explaining the task at every stage. They live in a per-feature directory:

```
.pipeline/<feature-slug>/
├── 00-request.md          # Original user request + success criteria
├── 01-plan.md             # planner → architect
├── 02-architecture.md     # architect → implementation
├── 03-implementation.md   # implementation → code-reviewer
├── 04-code-review.md      # code-reviewer → database-reviewer
├── 05-database-review.md  # database-reviewer → doc-updater
└── 06-docs.md             # doc-updater → final summary for the user
```

Use a short kebab-case slug derived from the feature name (e.g., `.pipeline/semantic-search/`). Add `.pipeline/` to `.gitignore` unless the user wants the audit trail committed.

### Handoff Format

Every handoff document MUST follow this template:

```markdown
# Handoff: [Stage Name] → [Next Stage Name]

**Feature:** [feature name]
**Stage:** [N of 6]
**Author:** [agent name]
**Date:** YYYY-MM-DD
**Status:** COMPLETE | COMPLETE_WITH_WARNINGS | BLOCKED

## Task Summary
[2-4 sentences: what this feature is and what the pipeline is trying to achieve.
Carry this forward from the previous handoff, updating only if scope changed.]

## Work Completed This Stage
[What this agent actually did — files read, plans made, code written,
issues found. Be specific: file paths, function names, decisions.]

## Decisions Made
[Numbered list. Each decision includes the WHY, not just the what.
Carry forward unresolved decisions from earlier stages.]

1. **[Decision]** — [Rationale]

## Instructions for Next Stage
[Explicit, actionable directives for the next agent. What to focus on,
what to skip, what order to work in. This section is the next agent's
primary input — write it as if the next agent knows nothing else.]

## Open Questions / Risks
[Anything unresolved, assumptions made, or risks the next stages should
watch for. Write "None" if empty — never omit the section.]

## Artifacts
[Files created or modified this stage, with paths. For review stages,
include verdicts (approve/warn/block) and issue counts by severity.]
```

## Orchestration Workflow

### Step 0 — Initialize

1. Confirm the feature request with the user and capture success criteria.
2. Create `.pipeline/<feature-slug>/00-request.md` containing the verbatim request, success criteria, and any constraints the user stated.
3. Determine which stages apply (see **Skipping Stages** below) and tell the user the planned pipeline before starting.

### Step 1 — Planner

Invoke the `planner` agent with `00-request.md` as input. The planner analyzes the codebase and produces its implementation plan **plus** the handoff `01-plan.md`. The plan itself (phases, steps, risks) goes in the handoff's Work Completed / Artifacts sections or as an attached section.

### Step 2 — Architect

Invoke the `architect` agent with `01-plan.md` as input. The architect reviews the plan against the existing architecture, makes design decisions with documented trade-offs (ADR-style where significant), and writes `02-architecture.md`. If the architect's design contradicts the plan, it must flag the conflict in **Open Questions** and propose a resolution — do not silently diverge.

### Step 3 — Implementation (the coding agent)

Read `01-plan.md` and `02-architecture.md`. Implement the code:

- Follow the plan's phase order; complete and verify each phase before starting the next.
- Honor the architecture's decisions; if reality forces a deviation, record it as a new decision with rationale.
- Run existing tests and add new ones per the plan.
- Write `03-implementation.md` when done, listing every file created/modified and any deviations from plan or architecture.

### Step 4 — Code Review

Invoke the `code-reviewer` agent with `03-implementation.md` as input. It diffs the changes, reviews against its checklist, and writes `04-code-review.md` including a verdict:

- ✅ **Approve** — continue to Step 5.
- ⚠️ **Warning** — continue, but carry warnings forward in the handoff.
- ❌ **Block** — return to Step 3. Fix the critical issues, update `03-implementation.md` (append a revision section, don't overwrite history), and re-run review. Max 3 fix cycles; if still blocked, stop and escalate to the user.

### Step 5 — Database Review

Invoke the `database-reviewer` agent with `04-code-review.md` as input **only if** the changes touch schemas, migrations, queries, or data access patterns. Same verdict/loop-back rules as Step 4. Writes `05-database-review.md`.

### Step 6 — Documentation

Invoke the `doc-updater` agent with the full handoff chain as input. It regenerates affected codemaps, updates READMEs/guides, verifies links and examples, and writes `06-docs.md` as the **final pipeline summary**: what was built, key decisions, review outcomes, and any remaining follow-ups.

### Step 7 — Report

Present the user a concise completion summary drawn from `06-docs.md`: feature delivered, files changed, review verdicts, open follow-ups. Link to the `.pipeline/<feature-slug>/` directory for the full audit trail.

## Skipping Stages

Not every task needs the full chain. Adapt, and record skipped stages in the handoffs:

- **No database changes** → skip Step 5.
- **Trivial/small change** (single file, no design impact) → planner may be skipped; start at implementation with `00-request.md` as input.
- **Docs-only or review-only requests** → invoke just the relevant agent, still writing a handoff for the audit trail.
- **User asks for plan only** → stop after Step 1 or 2 and present the handoff.

When in doubt, ask the user which stages they want rather than guessing.

## Resuming a Pipeline

If a `.pipeline/<feature-slug>/` directory already exists:

1. Read all handoffs in order to reconstruct state.
2. The highest-numbered handoff with `Status: COMPLETE` tells you the last finished stage; resume at the next one.
3. If the last handoff is `BLOCKED`, resolve its Open Questions (with the user if needed) before continuing.

## Rules

1. **Never skip the handoff.** A stage is not complete until its handoff document exists. The handoff IS the deliverable interface between stages.
2. **Handoffs are append-friendly, not rewrite-friendly.** On fix cycles, append revision sections rather than erasing prior content — the audit trail matters.
3. **Carry context forward.** Each handoff repeats the Task Summary and carries unresolved Decisions and Open Questions forward so no stage needs to read the entire chain (though it may).
4. **Escalate, don't spin.** Conflicts between stages, repeated review blocks, or ambiguous requirements go to the user after at most 3 attempts.
5. **One feature, one directory.** Parallel features get separate `.pipeline/` directories; never interleave handoffs.
6. **Reviews have teeth.** A ❌ Block verdict always loops back to implementation. Never proceed past a block without fixes or explicit user override (record the override in the handoff).
