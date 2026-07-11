# AI Coding Agents

A collection of agent instruction files that I currently use with my coding agents. Each file defines a specialized AI agent: a planner, an architect, reviewers, and a documentation specialist. These instruction files can be dropped into most agentic LLM tools (Claude Code, Codex, Cursor, and others). In short, they work to give your coding assistant a focused role with clear responsibilities, workflow and quality standards.

These are living documents. They reflect how I actually work today and will evolve as my workflows do.

## How They Work

There are two ways to use these agents:

**1. Standalone**: Each agent is fully self-contained. Invoke any one of them individually when you need its specialty: ask the `planner` to break down a feature, run the `code-reviewer` after a change, or bring in the `database-reviewer` before shipping a migration.

**2. Orchestrated (sequential pipeline)**: The agents are designed to work together as a chain, passing context between stages using handoff documents. The **[orchestrate](orchestrate/SKILL.md)** skill coordinates this flow:

```
planner → architect → agentic coding tool (Claude Code, Codex, Cursor, etc.) → code-reviewer → database-reviewer → doc-updater
```

Each agent completes its stage, writes a handoff document summarizing decisions and open items, and the next agent picks up from there. This keeps context tight, avoids re-explaining the task at every stage, and produces a trail of how a feature went from idea to documented code.

## The Agents

### 🗺️ planner
An expert planning specialist for complex features and refactors. It analyzes requirements, reviews the existing codebase, and produces a detailed, phased implementation plan. The agent is equipped with specific file paths, step-by-step actions, dependencies, risk ratings, and mitigations. Plans are structured to enable incremental, testable implementation, and the agent explicitly checks for edge cases, error scenarios, and common code smells before work begins.

### 🏛️ architect
A senior software architect focused on system design, scalability, and technical decision-making. It reviews current architecture, gathers functional and non-functional requirements, and proposes designs with documented trade-off analysis (pros, cons, alternatives, rationale). Includes guidance on common frontend/backend/data patterns, a full system design checklist, architectural anti-patterns to avoid, and a template for Architecture Decision Records (ADRs).

### 🔨 agentic coding (your tool of choice)
This stage isn't a custom agent, it's whatever agentic coding tool you already use (Claude Code, Codex, Cursor, Windsurf, etc.). The coding agent takes the handoff documents from the planner and architect and implements the actual code, following the phased plan and design decisions laid out in the earlier stages. Once implementation is done, the review agents take over.

### 🔍 code-reviewer
An expert code review specialist that runs immediately after code is written or modified. It diffs recent changes and reviews them against a prioritized checklist: critical security issues (hardcoded secrets, SQL injection, XSS, auth bypasses), code quality (function/file size, nesting depth, error handling, test coverage), performance (algorithm complexity, N+1 queries, unnecessary re-renders), and best practices. Feedback is organized by severity with concrete fix examples, and changes get a clear approve/warn/block verdict.

### 🗄️ database-reviewer
A PostgreSQL specialist for query optimization, schema design, security, and performance, incorporating patterns from Supabase's postgres-best-practices. It covers index strategy (composite, covering, partial, GIN/BRIN), proper data type selection, Row Level Security for multi-tenant data, connection pooling, deadlock prevention, and diagnostics with `EXPLAIN ANALYZE` and `pg_stat_statements` — plus a pre-approval checklist and a catalog of query, schema, security, and connection anti-patterns to flag.

### 📚 doc-updater
A documentation and codemap specialist that keeps docs in sync with the actual codebase. It generates architectural codemaps (`docs/CODEMAPS/*`) from repository structure using AST analysis (ts-morph, TypeScript Compiler API, madge), refreshes READMEs and guides from source, and validates that every file path, link, and code example in the docs actually works. Built on the principle that documentation which doesn't match reality is worse than no documentation.

### 🎯 orchestrate (skill)
A skill that chains the stages into a sequential pipeline. Each stage writes a markdown **handoff document**: task summary, decisions made (with rationale), and explicit instructions for the next stage, all into a per-feature `.pipeline/<feature-slug>/` directory, so context flows from planner through doc-updater without re-explaining the task. The skill also handles the practical mechanics: review verdicts with teeth (a ❌ Block loops back to implementation, capped at 3 fix cycles before escalating to you), skipping stages that don't apply (e.g., no database-reviewer if nothing touches the DB), and resuming an interrupted pipeline from its existing handoffs. See [orchestrate/SKILL.md](orchestrate/SKILL.md) for the full workflow and handoff template.

## Getting Started

The agent files use a common format: YAML frontmatter (name, description, tools, model) followed by markdown instructions. Most agentic coding tools accept this format directly or with minor tweaks.

### Claude Code

Copy the files into your agents directory:

```bash
# Project-level (shared with your team via git)
mkdir -p .claude/agents
cp *.md .claude/agents/

# Or user-level (available in all your projects)
mkdir -p ~/.claude/agents
cp *.md ~/.claude/agents/
```

Claude Code will auto-invoke agents based on their `description` field (note the `PROACTIVELY` hints), or you can invoke one explicitly:

```
> Use the code-reviewer agent to review my latest changes
```

### OpenAI Codex CLI

Codex uses `AGENTS.md` for instructions. Either paste an agent's content into your project's `AGENTS.md`, or keep the files in a folder (e.g. `agents/`) and reference them:

```markdown
<!-- AGENTS.md -->
When planning features, follow the instructions in agents/planner.md.
When reviewing code, follow the instructions in agents/code-reviewer.md.
```

### Cursor / Windsurf / other IDE agents

Add the agent content as project rules:

- **Cursor**: place files in `.cursor/rules/` (or paste into Settings → Rules), optionally converting the frontmatter to rule metadata
- **Windsurf**: add to `.windsurf/rules/` or your global rules

### ChatGPT / Claude.ai / any chat LLM

No setup needed — paste the contents of an agent file at the start of a conversation (or into a Project's custom instructions / a custom GPT's system prompt), then give it your task. The agent instructions work as a system prompt for any capable model.

### General approach for any tool

1. Strip or adapt the YAML frontmatter if your tool doesn't support it (the `tools` and `model` fields are Claude Code-specific).
2. Provide the markdown body as the agent's system prompt or rules file.
3. Give the agent read access to your codebase (and shell access for the reviewer/doc agents that run `git diff`, `psql`, or analysis scripts).
4. Invoke the right agent at the right stage of your workflow — or wait for the orchestrate skill to do it for you.

## Repo Structure

```
.
├── README.md
├── planner.md            # Feature & refactor planning
├── architect.md          # System design & trade-off analysis
├── code-reviewer.md      # Quality & security review
├── database-reviewer.md  # PostgreSQL optimization & security
├── doc-updater.md        # Codemaps & documentation sync
└── orchestrate/
    └── SKILL.md          # Sequential pipeline skill (handoff-based orchestration)
```

## License & Credits

Feel free to use, adapt, and remix these for your own workflows.

The `database-reviewer` agent adapts patterns from [Supabase Agent Skills](https://github.com/supabase/agent-skills) under the MIT license.
