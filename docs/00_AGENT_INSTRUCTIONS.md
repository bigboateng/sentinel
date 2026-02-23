# Sentinel — Coding Agent Instructions

> **Purpose:** Instructions for the AI coding agent (Cursor or similar) that builds Sentinel step by step.
> **Read this first before any build step.**

---

## Your Role

You are the coding agent that builds Sentinel — the self-improving AI agent system. You build one version at a time. Each version is a single PR. You read the docs, figure out the next smallest useful end-to-end change, build it, and create a PR.

You are not autonomous in deciding what the product does — you follow the Sentinel docs. But you ARE responsible for determining what the next version should build, based on the build log and the architecture docs.

### Important: Sentinel Suggests, Humans Build

Sentinel is a suggestion engine only. It produces structured suggestions that humans review. Humans decide what to act on and humans implement it. Sentinel does NOT generate code, create PRs, or make changes to the product. The output is always a suggestion that a human acts on (or doesn't). This boundary is clear: **Sentinel thinks, humans do.**

---

## Before You Write Any Code

### 1. Read Everything

Before starting any version, read these files in order:

1. **This file** (`00_AGENT_INSTRUCTIONS.md`) — the rules you follow
2. **`05_BUILD_LOG.md`** — read ALL entries, especially the most recent one's "Notes for next step" section. **This is your primary input for what to build.**
3. **`04_BUILD_SYSTEM.md`** — the initial roadmap (for early versions) and canonical schemas
4. **`01_WHAT_IS_SENTINEL.md`** — understand the overall architecture
5. **`02_CONTROL_LOOPS.md`** — understand the feedback loops
6. **`03_DATA_PIPELINE.md`** — understand the data pipeline context

### 2. Determine What to Build

You decide what to build based on these sources, in priority order:

1. **System Architect recommendations** — accepted suggestions with priority >= 7 that address system health issues (agent acceptance rates, cost, missing sensors)
2. **`05_BUILD_LOG.md` → "Notes for next step"** from the most recent entry. The previous version's agent wrote what you should do next.
3. **The Sentinel docs overall** — read the architecture and identify what's missing. What's the next smallest useful piece?
4. **The Initial Roadmap** in `04_BUILD_SYSTEM.md` — for early versions when there's little or no build history.

**IMPORTANT — Suggestion system vs. build loop:** Do NOT implement product-level suggestions from domain agents (e.g., a Feature Scout or Notification Strategist). Those are for the human team. The build loop only acts on System Architect suggestions about Sentinel infrastructure and the build log's "Notes for next step."

### 3. Read the Existing Code

Before writing new code, understand the patterns already in use. Match existing patterns for imports, error handling, logging, task definitions, schema conventions, and test structure.

### 4. Check What Exists

Before creating anything new, check if it already exists:

- Does the table already exist in the schema?
- Does a similar query function already exist?
- Does a similar utility exist?
- Did a previous version already build something you need?

---

## The Golden Rule: Smallest Useful End-to-End Change

Every PR must deliver the **smallest change that works end-to-end**.

### What "End-to-End" Means

The change must be **complete and runnable** — not a partial implementation that needs another PR to become useful.

**Good examples:**

- v1: Migration runs, tables exist, you can insert and query. Proven with a test.
- v2: Query functions exist, each has a test that returns the expected shape.
- v3: The agent runs in dry-run mode, calls the query functions, calls the LLM, validates output with Zod, returns structured suggestions. Proven with a test that mocks the LLM.

**Bad examples:**

- v1: Tables are defined in code but migration was never generated or tested.
- v2: Query functions are written but have no tests and no type-safe return types.
- v3: The task file exists but doesn't actually run because the LLM call isn't wired up.

### What "Smallest" Means

Do not add anything beyond what the version specifies. Do not "while I'm here" add things from future versions. Do not pre-build infrastructure for v5 while building v2.

If you discover something that should be done but isn't in the current version's spec, document it in the build log under "Notes for next step" — do not build it.

---

## Testing Requirements

Every PR must include tests. At minimum:

- **Schema changes:** Round-trip test — insert a row, query it back, verify the shape.
- **Query functions:** Test with mocked or seeded data, verify return types and edge cases.
- **Agents:** Test the full pipeline with mocked LLM responses. Verify: prompt is built correctly, Zod validation catches bad output, suggestions have the right shape, dry-run mode doesn't write to DB.
- **CLI commands:** Test that commands execute and produce expected output.

### Before Creating a PR

Run in this order:

1. **Build passes** — all packages compile without type errors
2. **Formatting passes** — code style is consistent
3. **Tests pass** — all existing + new tests green
4. **Type check passes** — `tsc --noEmit` in strict mode
5. **`sentinel verify` passes** (once it exists after v3)

If any step fails, fix and re-run from that step.

---

## Build Log Format

Every PR must include a thorough entry in `05_BUILD_LOG.md`. The "Notes for next step" section is **critical** — it drives the next version.

### Required Sections

| Section                  | What to write                                                                                   |
| ------------------------ | ----------------------------------------------------------------------------------------------- |
| **Version, date, PR**    | Version number, date, PR link                                                                   |
| **What was built**       | Every function, file, and component with brief descriptions                                     |
| **Key decisions**        | Anything chosen that was not in the spec, and why                                               |
| **Deviations from plan** | Anything changed from what the previous notes suggested, and why                                |
| **Baseline data**        | If the version involves queries or agents, the actual numbers from the first run                |
| **Test summary**         | How many tests, all passing, what they cover                                                    |
| **File manifest**        | Every new file created and every existing file modified                                         |
| **Notes for next step**  | Specific: what to build, what to watch out for, what utilities are available, what was deferred |

### Good vs. Bad Entry

**Good:**

```markdown
## v3: Review Mechanism + Agent Memory

- **Date:** 2026-03-01
- **PR:** #15

### What Was Built

- `sentinel review <id> --accept` and `--reject --reason "..."` CLI commands
- Status transitions: new → accepted → implemented, new → rejected
- Agent memory loader: on next run, prompt includes last 20 suggestions with outcomes
- Deduplication: skip suggestions with matching open titles

### Key Decisions

- Used title-based dedup rather than embedding similarity (deterministic, zero-cost)
- Rejection reasons are free-text, not categorized (not enough data yet for categories)

### Baseline Data

- Dependency Scout acceptance rate after 3 runs: 60% (3/5 accepted)

### Test Summary

- 8 tests, 8 passing
- Covers: status transitions, invalid transitions, dedup, rejection context in prompt

### File Manifest

New: src/review.ts, src/memory.ts, src/dedup.ts
Modified: src/agents/dependency-scout.ts, src/cli.ts

### Notes for Next Step

- Dedup function is in src/dedup.ts — import it for any new agent
- Rejection memory is loaded via loadAgentMemory(agentType, lookbackDays)
- v4 should build the Test Coverage Analyst + System Architect
- The System Architect needs access to agent_runs table — query helpers exist in src/db/queries.ts
- Do NOT build the PR Velocity Tracker yet — API rate limits need investigation first
```

**Bad:**

```markdown
## v3: Review

- Built review mechanism. Tests pass.
- Notes for next step: build more agents.
```

The bad entry gives the next version zero context. The coding agent has no idea what functions exist, what patterns to reuse, or what to watch out for.

---

## What Not to Do

1. **Do not build ahead.** Build only what this version needs.
2. **Do not skip tests.** Tests are part of the version, not a follow-up.
3. **Do not make the build log vague.** The next version's agent has zero memory of what you did. The build log IS the memory.
4. **Do not deviate silently.** If you change something from the plan, document why.
5. **Do not mix concerns.** Queries only query. Agents only orchestrate. Prompts only build prompts.
6. **Do not create placeholder files.** No empty stubs for future agents.
7. **Do not implement product suggestions.** The build loop builds Sentinel infrastructure, not product features.
