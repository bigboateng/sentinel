# Sentinel — What It Is

## Overview

Sentinel is a system of AI agents that continuously analyze your production systems and surface the highest-leverage improvements your team would never find manually.

Every software product has more opportunities to improve than any team can find on their own. Not bugs or outages — those get caught. The strategic improvements that compound over time: the notification timing that would double engagement, the onboarding step where users silently drop off, the data field that's populated but never surfaced in the UI, the dependency that drifted three major versions behind.

Sentinel finds these by reading your systems through configurable **lenses** — each lens is an agent with a specific perspective on the same underlying data.

---

## Isolation from Product Code

Sentinel is **completely isolated** from your product codebase. It does not live inside your app, your API, or your frontend. It is a separate system that:

- **Reads** from your production database (read-only queries)
- **Reads** from your code repository (static analysis, schema inspection)
- **Reads** from external data sources (analytics, error tracking, package registries)
- **Writes** only to its own tables (suggestions, agent runs, config)

Your product team never has to touch Sentinel code. Your product code never imports from Sentinel. The only connection is the read-only data access layer.

```
┌─────────────────────────────────────────────────┐
│                YOUR PRODUCT                      │
│  App  │  API  │  Database  │  Analytics  │  CI  │
└───────┬─────────────────────────────────────────┘
        │ read-only queries
        ▼
┌─────────────────────────────────────────────────┐
│                SENTINEL (isolated)                │
│                                                   │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────┐ │
│  │ Data queries │  │ Agents      │  │ Review   │ │
│  │ (sensors)    │  │ (lenses)    │  │ pipeline │ │
│  └──────┬──────┘  └──────┬──────┘  └────┬─────┘ │
│         │                │               │       │
│         ▼                ▼               ▼       │
│  ┌─────────────────────────────────────────────┐ │
│  │         Sentinel database (own tables)       │ │
│  │  suggestions │ agent_runs │ config │ health  │ │
│  └─────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────┘
```

This isolation means:

- **No risk to production.** Sentinel can't break your app because it never writes to your app's data.
- **No coupling.** Product code changes don't affect Sentinel. Sentinel changes don't affect the product.
- **Easy to add, easy to remove.** Sentinel is a bolt-on system. You can add it to any existing codebase.

---

## The Core Architecture

### Lenses, Not Dashboards

A Sentinel lens is an agent with a perspective. The same database, the same codebase, the same logs — but different agents surface different insights:

- An **engineering lens** finds N+1 queries, dead code paths, error patterns in audit logs, dependency drift
- A **product lens** finds funnel drop-offs, adoption gaps, activation blockers, churn signals
- A **design lens** finds rage clicks, navigation dead-ends, abandoned flows, UX friction
- A **data quality lens** finds schema fields that are populated but unused, or needed but empty

Adding a new perspective is just adding a new agent. The LLM is what lets you define a lens in plain language instead of writing custom analytics code. You describe what to look for, the agent does the SQL + code reading + reasoning.

### The Core Loop

Every agent follows the same cycle:

```
[Schedule trigger] → [Query production data] → [Analyze with LLM or rules]
        → [Output: structured suggestion] → [Store in Sentinel DB]
                → [Human reviews] → [Accept / Reject with reason]
                        → [If implemented: measure metric]
                                → [Agent sees updated data on next run]
```

### Safety Boundary

Sentinel suggests. Humans decide. Humans build.

Agents never modify the product, push code, or change production data. Human review is always in the loop. Every suggestion is a structured, reviewable, trackable unit of work — not an automated action.

---

## The 5-PR Base System

The base Sentinel infrastructure takes **5 PRs** to stand up. After that, adding agents is incremental — each new agent is a single PR following the canonical pipeline pattern.

| PR                               | What it delivers                                                                               | What works after this PR                                                |
| -------------------------------- | ---------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------- |
| **v1: Foundation**               | Database tables, CLI skeleton (`sentinel next`, `sentinel status`, `sentinel seed`), seed data | You can store and query suggestions. `sentinel next` outputs v2 prompt. |
| **v2: First Agent**              | One agent following the canonical pipeline, dry-run mode, `--simulate` mode                    | `sentinel run --simulate degrading` produces real suggestions.          |
| **v3: Review + Memory + Verify** | Accept/reject, rejection learning, dedup, `sentinel verify` quality gate                       | Feedback loop works. System enforces its own build quality.             |
| **v4: System Architect**         | Meta-agent that monitors all other agents, computes acceptance rates, flags underperformers    | Outer loop works. System monitors itself.                               |
| **v5: Integration Sentinel**     | Data source health monitoring, data freshness checks                                           | Data pipeline loop works. All five loops operational.                   |

### After v5: Agents Forever

The base system is complete. From here, you just keep adding agents:

- Each new agent is a single PR
- Each follows the canonical 11-step pipeline (see `04_BUILD_SYSTEM.md`)
- `sentinel next` determines what to build next based on the build log + System Architect recommendations
- `sentinel verify` validates every PR before merge
- The System Architect monitors each new agent's health automatically

**There is no final version.** The system builds itself indefinitely. As your product evolves, new lenses become valuable. The System Architect identifies gaps and recommends new agents. The build log propagates context forward. Each version builds on everything before it.

---

## Extensibility — It Pairs with a Coding Tool

The architecture docs are the system's source of truth. A coding agent (Cursor, Copilot, or similar) that reads them understands: the canonical pipeline, the feedback loops, the schemas, and the safety constraints. This means extending Sentinel is as simple as describing what you want in plan mode.

Because every agent follows the same 11-step pipeline and the same control-theory structure, new capabilities can be added to a single agent or applied across all of them:

- **New output channels** — "When a priority >= 8 suggestion is created, post it to Slack." One integration point, every agent benefits.
- **Frontend surfaces** — "Build a dashboard that reads the suggestions table and shows pending items by agent type." The schema is already defined; the UI just reads it.
- **Richer data capture** — "Add a `userSegment` field to suggestions so agents can break down metrics by cohort." One schema migration, all agents can populate it.
- **New agent capabilities** — "Give the Dependency Scout access to CI logs so it can correlate outdated packages with build failures." One new query function, wired into one agent's `queryData` step.
- **Cross-agent coordination** — "When the Integration Sentinel detects a data source failure, suppress all agents that depend on that source until it recovers." One new check in the pipeline's step 2, applied globally.

None of these require rearchitecting the system. The coding agent reads the docs, identifies where the change fits in the canonical pipeline, and builds it — following the same patterns, the same schemas, the same testing strategy.

```
You (plan mode):  "Add Slack notifications for high-priority suggestions"

Coding agent:
  1. Reads 04_BUILD_SYSTEM.md → finds canonical pipeline step 10 (store suggestions)
  2. Reads schema → suggestions table has `priority` field
  3. Adds a post-store hook: if priority >= 8, call Slack webhook
  4. Adds test: mock webhook, verify it fires for priority 8, doesn't fire for priority 5
  5. Updates build log with what was built and why

One PR. Every agent that produces a high-priority suggestion now notifies Slack.
```

The control-theory design is what makes this safe. Cost guardrails prevent new capabilities from blowing budgets. Dry-run mode lets you test changes without side effects. The System Architect monitors whether the new capability is actually useful. The build log ensures the next version knows what changed.

---

## Where It Lives

Sentinel lives in its own package or directory, separate from your product code:

```
your-project/
├── apps/                    # Your product
├── packages/                # Your shared libraries
├── sentinel/                # Sentinel (isolated)
│   ├── docs/                # Architecture, build log, agent instructions
│   ├── src/
│   │   ├── db/              # Sentinel's own schema + queries
│   │   ├── agents/          # Agent implementations
│   │   ├── review/          # Accept/reject/memory
│   │   └── cli/             # sentinel next, verify, run, status
│   └── tests/
└── ...
```

Or as a standalone project entirely:

```
sentinel/
├── docs/
├── src/
├── fixtures/
├── package.json
└── ...
```

The key constraint: Sentinel has **read-only access** to your product's data sources and **write access only to its own tables**. It does not import from your product code. It does not export to your product code.
