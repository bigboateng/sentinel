# Sentinel — Data Pipeline Loop

## The Third Feedback Loop

The inner loop governs individual agent suggestions. The outer loop governs the Sentinel system itself. The data pipeline loop governs the quality, freshness, and completeness of data flowing into Sentinel — the raw material everything else depends on.

Like all Sentinel loops, this is a **negative-feedback controller**: it measures the gap between current data completeness and target coverage, proposes corrections that close the gap, and verifies the gap shrank after implementation. When data quality reaches the setpoint, the loop is quiet.

Without this loop, domain agents can identify that an improvement _should_ happen but can't explain _why the data isn't there_ or _what to change upstream to fix it_.

---

## The Problem

### Schema vs. Reality Gap

Every production database has a gap between what the schema _can_ store and what's actually _populated_. Fields exist in the schema but are null for most rows. Tables are defined but sparse. External data sources return partial results.

This gap matters because Sentinel agents reason on real data. If an agent suggests "personalize notifications by timezone" but the `timezone` field is null for 70% of users, the suggestion is premature. The data pipeline loop detects this gap and proposes how to close it.

### Why the Gap Exists

Typical reasons:

- Fields were added to the schema but the code path that populates them was never completed
- An external API returns the data, but the ingestion pipeline doesn't map it
- The data exists in logs or events but hasn't been aggregated into queryable tables
- A migration added columns but the UI or API never collects the values

---

## The Integration Sentinel

The Integration Sentinel is a **deterministic agent** (no LLM) that monitors data source health.

### What It Checks

| Check                   | What it catches                                                        |
| ----------------------- | ---------------------------------------------------------------------- |
| **Data freshness**      | Has each data source ingested successfully within the expected window? |
| **Credential validity** | Are API credentials configured and working?                            |
| **Error rates**         | Are there repeated ingestion errors (>3 in 24 hours)?                  |
| **Coverage gaps**       | For each entity that should have data, does data actually exist?       |
| **Staleness**           | Is any data source returning the same values it returned days ago?     |

### What It Outputs

- `data_access_gap`: "Analytics data missing for /features/dashboard — no sessions recorded in 14 days"
- `service_setup_gap`: "Error tracking API credentials not configured"
- `data_freshness_gap`: "Package registry ingestion failed for 48+ hours — agents reasoning on stale dependency data"

### Settling Periods

New data sources get a settling period before the Integration Sentinel acts on them. If you just deployed a new analytics pipeline, the sentinel waits (e.g., 14 days) before flagging missing data. This prevents false positives during the initial ramp-up.

---

## How It Connects to the Other Loops

```
┌─────────────────────────────────────────────────────────────┐
│                    DOMAIN AGENTS (Inner Loop)                 │
│                                                              │
│  Agent A ─────────┐                                          │
│  Agent B ─────────┤  "We need field X for this suggestion"  │
│  Agent C ─────────┤  "Field Z has low coverage"              │
│  Agent D ─────────┘                                          │
└──────────────┬───────────────────────────────────────────────┘
               │ demand signal
               ▼
┌─────────────────────────────────────────────────────────────┐
│              INTEGRATION SENTINEL (Pipeline Loop)            │
│                                                              │
│  Compares:                                                   │
│  ┌───────────────────┐  ┌──────────────────┐                 │
│  │ What agents need  │  │ What data exists │                 │
│  └────────┬──────────┘  └────────┬─────────┘                 │
│           ▼                      ▼                           │
│  ┌────────────────────────────────────────┐                  │
│  │ Gap analysis: what's missing or stale? │                  │
│  └────────────────────┬───────────────────┘                  │
│                       ▼                                      │
│  Suggestion: "Fix ingestion X, add field Y, rotate cred Z"  │
└──────────────┬───────────────────────────────────────────────┘
               │ implemented changes
               ▼
┌─────────────────────────────────────────────────────────────┐
│              DATA SOURCES                                     │
│                                                              │
│  Richer, fresher data flows into production DB               │
│  → Domain agents now have data to work with                  │
│  → Back to Inner Loop                                        │
└─────────────────────────────────────────────────────────────┘
```

---

## Why This Loop Exists Separately

| Property            | Inner Loop                            | Outer Loop                      | Data Pipeline Loop                                               |
| ------------------- | ------------------------------------- | ------------------------------- | ---------------------------------------------------------------- |
| **Question**        | Did this suggestion improve a metric? | Is the Sentinel system healthy? | Is the data pipeline producing what agents need?                 |
| **Scope**           | One domain suggestion                 | All agents and data sources     | Data source freshness and completeness                           |
| **Output**          | Domain improvement suggestions        | System improvement suggestions  | Data pipeline fixes, credential rotations, coverage improvements |
| **Feedback signal** | Metric before/after                   | Agent acceptance rate, cost     | Data coverage % before/after fix                                 |

Without this loop:

- Domain agents keep suggesting improvements that can't be validated because the data doesn't exist
- Data source failures go undetected for days or weeks
- Agents reason on stale data and produce blind suggestions
- The team manually discovers data gaps through trial and error

With this loop:

- Data gaps are detected automatically when an agent needs a field
- Source failures are caught within hours, not weeks
- Suggestions include evidence of data availability before recommending changes
- Coverage improvements are tracked and verified over time
