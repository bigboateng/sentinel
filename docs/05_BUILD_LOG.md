# Sentinel — Build Log

> **This file is the memory of the system.** Every version built adds an entry here. The "Notes for next step" section of the most recent entry is the primary input for determining what to build next. If this file is empty, start with v1 from the Initial Roadmap in `04_BUILD_SYSTEM.md`.

---

## Entry Format

Every entry must include all of these sections. The coding agent reads this file before building anything — vague entries mean the next version is built blind.

```markdown
## vN: <Short Title>

- **Date:** YYYY-MM-DD
- **PR:** #<number>

### What Was Built

Bullet list of every function, file, and component. Brief descriptions of what each does.

### Key Decisions

Anything chosen that wasn't in the spec, and why. Trade-offs made.

### Deviations from Plan

Anything changed from what the previous version's "Notes for next step" suggested, and why.

### Baseline Data

If the version involves queries or agents, the actual numbers from the first run.
(e.g., "Dependency Scout found 4 outdated packages on first run, 2 with critical CVEs")

### Test Summary

How many tests, all passing, what they cover.

### File Manifest

New: list of new files
Modified: list of modified files

### Notes for Next Step

SPECIFIC guidance for the next version's coding agent:

- What to build next
- What files/functions already exist that should be reused
- What patterns to follow
- What was deferred and why
- What to watch out for
```

---

## Example Entry

```markdown
## v1: Foundation — Schema, CLI, Seed

- **Date:** 2026-02-20
- **PR:** #1

### What Was Built

- Database schema with Drizzle ORM: `suggestions`, `agent_runs`, `agent_config` tables
- Migration generated and tested (SQLite)
- CLI skeleton with Commander.js: `sentinel next`, `sentinel status`, `sentinel seed`
- `sentinel seed` populates 90 days of realistic historical data (configurable with --days)
- `sentinel status` prints agent count, suggestion counts by status, and cost summary
- `sentinel next` reads build log + architecture docs → outputs constrained prompt for v2

### Key Decisions

- SQLite over Postgres for zero-infrastructure setup. Can migrate to Postgres later if needed.
- Drizzle ORM for type-safe queries and simple migration management.
- Commander.js for CLI (lightweight, widely used).
- Seed data includes 3 fake agent types with mixed suggestion statuses to test queries.

### Deviations from Plan

- None. This is v1, following the Initial Roadmap.

### Baseline Data

- Seed produces: 45 suggestions (18 accepted, 9 rejected, 12 implemented, 6 new)
- Seed produces: 30 agent_run entries across 3 agent types
- `sentinel status` correctly reads and displays all counts

### Test Summary

- 12 tests, 12 passing
- Covers: table creation, insert/query round-trip for all 3 tables, seed data counts,
  CLI command execution (status output format, seed idempotency)

### File Manifest

New: src/db/schema.ts, src/db/queries.ts, src/db/migrate.ts, src/cli/index.ts,
src/cli/next.ts, src/cli/status.ts, src/cli/seed.ts,
src/seed/generate.ts, tests/db.test.ts, tests/cli.test.ts,
drizzle.config.ts, package.json, tsconfig.json
Modified: (none — first version)

### Notes for Next Step

- v2 should build the first domain agent following the canonical 11-step pipeline
- Query functions are in src/db/queries.ts — add new ones there, following the existing pattern
- The suggestion insert function is `insertSuggestion(data)` — it handles ID generation
- The run logger is `logAgentRun(data)` — call it at the end of every agent run
- Agent config for the new agent type needs to be seeded (add to src/seed/generate.ts)
- The --simulate flag needs to be wired into the CLI run command (scaffold exists but isn't connected)
- Fixtures should go in fixtures/scenarios/<scenario-name>.json
- Do NOT build the review mechanism yet — that's v3
```

---

## Build History

_No versions built yet. Start with v1 from the Initial Roadmap in `04_BUILD_SYSTEM.md`._
