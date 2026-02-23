# Sentinel — Build System

## Overview

Sentinel builds itself. Not automatically — a human drives each version. But the system provides the structure, the next-step reasoning, and the verification so that each version is correctly scoped, correctly built, and correctly tested.

The build system is the fifth feedback loop. Its inputs are the build log (state from every prior version), the architecture docs (the target), and optionally the System Architect's recommendations (what the system itself identified as needed). Its output is a constrained prompt that tells the coding agent exactly what to build next.

---

## CLI Commands

### `sentinel next`

Reads the build log, architecture docs, and (if it exists) the System Architect's recommendations. Outputs the next version's constrained prompt — what to build, what files exist, what patterns to follow, what NOT to build.

```bash
sentinel next
# Output:
# === SENTINEL BUILD v6 ===
# Based on: build log v5, System Architect recommendation SA-12
#
# BUILD THIS:
# - Test Coverage Analyst agent following canonical pipeline in src/agents/
# - Reads test coverage data from CI (query function: getCITestCoverage)
# - Outputs suggestions of type "test_gap"
#
# CONSTRAINTS:
# - Follow pattern in src/agents/dependency-scout.ts
# - Reuse dedup from src/dedup.ts
# - Reuse memory loader from src/memory.ts
# - Do NOT build PR Velocity Tracker (deferred, needs API rate limit investigation)
#
# TESTING:
# - Mock LLM, verify Zod schema, test dry-run, test dedup
#
# WHEN DONE:
# - Update 05_BUILD_LOG.md with full entry
# - Run sentinel verify
# - Create PR
```

### `sentinel verify`

Validates the current version against quality criteria before PR creation:

```bash
sentinel verify
# ✓ All tests pass
# ✓ Type check passes
# ✓ Build log updated with v6 entry
# ✓ Build log has "Notes for next step" section
# ✓ No files outside sentinel/ directory were modified
# ✗ Missing test for dry-run mode
# RESULT: FAIL — fix the above before creating PR
```

### `sentinel run`

Runs a specific agent (or all agents) against production data or simulated data:

```bash
sentinel run dependency-scout                    # production data
sentinel run dependency-scout --dry-run          # production data, no DB writes
sentinel run dependency-scout --simulate degrading  # synthetic scenario
sentinel run --all --dry-run                     # all agents, no writes
```

### `sentinel review`

Review suggestions produced by agents:

```bash
sentinel review                         # list pending suggestions
sentinel review <id> --accept           # mark as accepted
sentinel review <id> --reject --reason "not actionable — needs specific file paths"
```

### `sentinel seed`

Populate realistic historical data for the demo or a fresh install:

```bash
sentinel seed                   # default: 90 days of realistic data
sentinel seed --days 30         # shorter history
```

### `sentinel status`

Dashboard showing system health:

```bash
sentinel status
# Agents: 4 active, 0 errored
# Suggestions: 12 pending, 8 accepted, 3 rejected, 5 implemented
# Acceptance rate: 72%
# Cost (this month): $4.20 / $50.00 budget
# Data sources: 3/3 healthy
# Last System Architect run: 2 days ago — 1 recommendation pending
```

---

## Initial Roadmap (5 Versions)

| Version | Scope                                                                                  | Delivers                                                                                                            |
| ------- | -------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| **v1**  | Foundation — schema, migrations, CLI skeleton, seed data                               | Tables exist. `sentinel seed` populates history. `sentinel next` outputs v2 prompt. `sentinel status` shows counts. |
| **v2**  | First agent — canonical pipeline, one domain agent, `--simulate` mode                  | `sentinel run <agent> --simulate degrading` produces structured suggestions. Dry-run works.                         |
| **v3**  | Review + memory + verify — accept/reject, dedup, rejection learning, `sentinel verify` | Feedback loop closes. Build quality is enforced. Agent sees previous outcomes.                                      |
| **v4**  | System Architect — meta-agent monitoring agent health                                  | Outer loop works. Acceptance rates, cost tracking, agent health scoring. Recommends prompt rework or new agents.    |
| **v5**  | Integration Sentinel — data source monitoring                                          | Data pipeline loop works. Freshness checks, credential validation, coverage gap detection. All 5 loops operational. |

After v5, every new version is a new agent or system improvement, driven by the build log and System Architect.

---

## Canonical Agent Pipeline (11 Steps)

Every agent follows this pipeline. No exceptions. This consistency is what makes new agents cheap to build and easy to verify.

| Step                         | What happens                                                                                     |
| ---------------------------- | ------------------------------------------------------------------------------------------------ |
| 1. **Schedule trigger**      | Cron, manual, or CLI invocation                                                                  |
| 2. **Load configuration**    | Read agent config (lookback window, thresholds, LLM model)                                       |
| 3. **Query production data** | Call read-only query functions against production DB                                             |
| 4. **Load agent memory**     | Fetch last N suggestions with outcomes (accepted, rejected + reason, implemented + metric delta) |
| 5. **Build prompt**          | Assemble system prompt + data context + memory into LLM prompt                                   |
| 6. **Call LLM**              | Send prompt, receive structured response                                                         |
| 7. **Validate output**       | Parse with Zod schema — reject malformed responses                                               |
| 8. **Deduplicate**           | Check for open suggestions with matching titles — skip duplicates                                |
| 9. **Dry-run gate**          | If `--dry-run`, log and return without writing. If `--simulate`, use fixture data.               |
| 10. **Store suggestions**    | Write validated, deduplicated suggestions to Sentinel DB                                         |
| 11. **Log run metadata**     | Record: timestamp, token count, cost, suggestion count, errors                                   |

---

## Canonical Schemas

### `suggestions` table

| Column               | Type                   | Purpose                                          |
| -------------------- | ---------------------- | ------------------------------------------------ |
| `id`                 | UUID                   | Primary key                                      |
| `agentType`          | string                 | Which agent produced this                        |
| `title`              | string                 | Short summary (also used for dedup)              |
| `description`        | text                   | Full explanation with evidence                   |
| `priority`           | integer (1-10)         | Agent-assigned priority                          |
| `effort`             | enum (low/medium/high) | Estimated implementation effort                  |
| `status`             | enum                   | new → accepted → implemented, new → rejected     |
| `rejectionReason`    | text                   | Why it was rejected (feeds back to agent memory) |
| `targetMetric`       | string                 | What metric this aims to improve                 |
| `metricBefore`       | float                  | Metric value at suggestion time                  |
| `metricAfter`        | float                  | Metric value post-implementation                 |
| `implementationHint` | text                   | How to implement (code paths, files, approach)   |
| `dataEvidence`       | JSON                   | Raw data that supports the suggestion            |
| `createdAt`          | timestamp              | When the agent produced it                       |
| `reviewedAt`         | timestamp              | When a human reviewed it                         |
| `implementedAt`      | timestamp              | When the change was deployed                     |

### `agent_runs` table

| Column                | Type      | Purpose                                 |
| --------------------- | --------- | --------------------------------------- |
| `id`                  | UUID      | Primary key                             |
| `agentType`           | string    | Which agent ran                         |
| `startedAt`           | timestamp | Run start                               |
| `completedAt`         | timestamp | Run end                                 |
| `status`              | enum      | success / error / dry_run               |
| `tokenCount`          | integer   | Total tokens consumed                   |
| `estimatedCost`       | float     | Cost in dollars                         |
| `suggestionsProduced` | integer   | How many suggestions this run generated |
| `errorMessage`        | text      | If status is error, what went wrong     |
| `isDryRun`            | boolean   | Whether this was a dry run              |
| `isSimulated`         | boolean   | Whether this used fixture data          |

### `agent_config` table

| Column                 | Type    | Purpose                                |
| ---------------------- | ------- | -------------------------------------- |
| `agentType`            | string  | Primary key                            |
| `enabled`              | boolean | Is this agent active?                  |
| `schedule`             | string  | Cron expression                        |
| `lookbackDays`         | integer | How far back to query data             |
| `maxSuggestionsPerRun` | integer | Cap on suggestions per execution       |
| `costBudgetPerRun`     | float   | Max cost per run in dollars            |
| `costBudgetMonthly`    | float   | Max monthly cost in dollars            |
| `llmModel`             | string  | Which model to use                     |
| `settlingDays`         | integer | Days to wait before acting on new data |

---

## Agent Registry Template

When adding a new agent, create a file like `src/agents/<agent-name>.ts` with this structure:

```typescript
import { defineAgent } from "../agent-framework";
import { loadAgentMemory } from "../memory";
import { dedup } from "../dedup";
import { SuggestionSchema } from "../schemas";

export const myAgent = defineAgent({
  type: "my-agent-type",
  displayName: "My Agent Display Name",

  async queryData(config) {
    // Step 3: read-only queries against production data
  },

  buildPrompt(data, memory) {
    // Step 5: assemble the LLM prompt
  },

  outputSchema: SuggestionSchema,

  async run(options) {
    const config = await loadConfig(this.type);
    const data = await this.queryData(config);
    const memory = await loadAgentMemory(this.type, config.lookbackDays);
    const prompt = this.buildPrompt(data, memory);

    if (options.dryRun) {
      return { prompt, suggestions: [] };
    }

    const response = await callLLM(prompt, config);
    const validated = this.outputSchema.parse(response);
    const unique = await dedup(this.type, validated);

    await storeSuggestions(unique);
    await logRun(this.type, {
      tokenCount: response.usage,
      suggestions: unique.length,
    });

    return { suggestions: unique };
  },
});
```

---

## Cost Guardrails

| Guard              | Scope                  | Default    | Behavior when exceeded                                          |
| ------------------ | ---------------------- | ---------- | --------------------------------------------------------------- |
| **Per-run budget** | Single agent execution | $2.00      | Agent aborts, logs error, run marked as `error`                 |
| **Monthly budget** | All agents combined    | $50.00     | All agents skip execution, `sentinel status` shows budget alert |
| **Token cap**      | Single LLM call        | 32K tokens | Prompt is truncated to fit (most recent data first)             |

Cost is tracked per run in the `agent_runs` table. `sentinel status` shows month-to-date cost against the monthly budget.

---

## Testing Strategy

| What                   | How                                                                  | When                |
| ---------------------- | -------------------------------------------------------------------- | ------------------- |
| **Schema migrations**  | Insert → query → verify shape                                        | Every schema change |
| **Query functions**    | Seed data → call function → verify output                            | Every new query     |
| **Agent pipeline**     | Mock LLM → run agent → verify Zod parse + dedup + suggestion shape   | Every new agent     |
| **Dry-run mode**       | Run agent with `--dry-run` → verify no DB writes                     | Every agent         |
| **Simulate mode**      | Run with `--simulate <scenario>` → verify suggestions match scenario | Every agent         |
| **CLI commands**       | Run command → verify output format                                   | Every CLI change    |
| **Cost guardrails**    | Mock expensive response → verify agent aborts                        | v2+                 |
| **Build verification** | Run `sentinel verify` → verify all checks pass                       | Every PR            |
