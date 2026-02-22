# Why Sentinel

Deep dive into what Sentinel does, how it works, and where it's going. For the quick version, see the [README](../README.md).

---

## The Problem

AI lets you build fast. So you build a lot. Every system you ship produces signals — audit logs, API responses, user events, error traces, metrics. Those signals contain everything you need to know: where users struggle, where errors come from, what's breaking silently, what features nobody uses.

But nobody looks at it systematically. The engineering effort goes to building, not observing. You ship a feature, move on, and the data it produces sits there. Capabilities go uncapitalized. Problems go unnoticed until they're crises. Low-hanging fruit stays on the tree.

Sentinel solves this by shifting observation work to agents that run continuously, cost pennies, and learn from your feedback.

---

## How Agents Work

### You spec goals, not data sources

You say: *"Build a Sentinel agent that keeps onboarding activation above 85%."*

You don't say which tables, which columns, which queries, or which metrics. Your coding agent already has your schema, your domain logic, and the Sentinel architecture in context. It picks the data sources, writes the sensor queries, defines the setpoints, and builds the full agent pipeline. You review it and run it.

This means the barrier to a new agent is: **can you describe the goal in a sentence?** If yes, the data is almost always already in your system.

### The control theory engine

Each agent is a negative-feedback controller — the same math that keeps rockets on course:

| Control theory | Sentinel mapping |
|---|---|
| **Setpoint** | Your goal (e.g., activation above 85%) |
| **Sensor** | SQL queries against your database |
| **Error signal** | Gap between current state and setpoint |
| **Actuator** | Structured suggestions to the review queue |
| **Feedback** | Human review + metric measurement after implementation |
| **Quiet condition** | Error is zero — agent emits nothing. System is healthy. |

The quiet condition is what makes this sustainable. A well-tuned agent goes silent when there's nothing to say. That silence is the signal that things are working.

### The feedback loop

Every interaction teaches the agent:

- **Accept** — the agent measures whether the metric moved. It learns what kind of suggestions land.
- **Reject: "too large"** — the agent learns to break suggestions down or filter for smaller wins.
- **Reject: "not a priority"** — the agent learns your threshold for what matters.
- **Reject: "already tried this"** — the agent remembers and stops suggesting it.
- **Ignore** — signals low confidence. The agent weighs that pattern over time.

Two teams with identical data get different suggestions because their feedback history is different. The agents aren't generic — they're personalized through use.

### Self-correcting at every level

Sentinel doesn't just correct what it monitors. It corrects itself:

- **Per-agent:** each agent tracks accept/reject rates and metric outcomes
- **System-wide:** the System Architect agent monitors other agents — are they useful? too expensive? producing suggestions that keep getting rejected?
- **Build loop:** `sentinel next` is itself an agent — it observes the build state, reasons about what's missing, and produces the next step

The logic behind Sentinel is Sentinel. The same observe-reason-suggest loop powers the agents and built the system.

---

## Two Modes of Observation

### Code-aware

The agent reads your codebase. It knows your schema, your queries, your domain logic. It connects data patterns to the code that produces them.

Instead of "reduce onboarding drop-off," the agent points to the exact 3-step flow in `VerifyEmail.tsx` and the API call in `onboardingService.ts` that could be simplified — based on the 34% drop-off data from your events table.

Good for: engineering insights, error detection, architecture suggestions, refactoring recommendations.

### Code-surviving

The agent ignores the code entirely. It watches outputs — API responses, metrics, business outcomes. The code underneath can be completely rewritten, refactored, or replaced. The agent still works because it monitors the result, not the implementation.

Good for: vibe-coded subsystems, third-party integrations, long-lived goals across rewrites, team handoffs where institutional knowledge is lost.

You can mix both. An onboarding agent might be code-aware (it knows the signup flow). A revenue agent might be code-surviving (it watches conversion rates regardless of how checkout is implemented).

---

## Use Cases

### Insight Engine

Surface product and engineering improvements from your existing data. Onboarding drop-off, feature adoption gaps, engagement trends. Described as a goal, built in minutes, runs for pennies.

*"Build a Sentinel agent that monitors user activation through the onboarding funnel."*

The agent reads your users and events tables, computes drop-off at each step, calls an LLM to analyze patterns, and produces prioritized suggestions with evidence. See [`examples/taskflow/`](../examples/taskflow/) for a working implementation.

### Domain Watchdogs

Complex domain logic — state machines, data transformations, lifecycle transitions — where bugs are subtle and expensive.

Example: an order system with states `pending → authorized → captured → refunded`. The agent watches the audit table, detects invalid transitions (e.g., `refunded` without ever being `captured`), and surfaces them with the exact row, timestamp, and the code path that produced the bad state.

*"Build a Sentinel agent that watches the order_audit_log for invalid state transitions based on the allowed lifecycle in OrderStateMachine.ts."*

No separate error-tracking tool. The audit data was already there. The agent just reads it.

### Autonomous Subsystems

Entire subsystems where the internals don't matter but the outcomes do. Give Sentinel a goal, API keys, and constraints. It builds the subsystem, maintains it, and adjusts based on metrics.

Example: a content SEO pipeline. Decide on a templating system, build the renderer, publish content, measure conversions. The internal code can be vibe coded. Sentinel watches the metrics — search rankings, click-through rates, conversion. When performance drops, it suggests adjustments. When it's healthy, it goes quiet.

*"Build a Sentinel subsystem that reliably converts high-intent Google searches. Target: 5% conversion from organic."*

You care about the numbers. Sentinel owns the internals.

### Drift Detection

Guard what's already working. You fix onboarding drop-off from 34% to 15%. The agent shifts from advisor to guardian — silent until metrics regress. If drop-off creeps back to 20%, it fires with evidence of what changed.

This is the "keep an eye on it" mode. The agent behavior is fundamentally different from suggesting improvements — it's protecting a known-good state.

### Cross-System Correlation

Connect signals across systems that humans would never manually cross-reference.

Example: deployment frequency goes up → test flakiness goes up → customer-reported bugs go up two weeks later. No single dashboard shows this chain. An agent that reads your CI logs, error tracking, and support tickets sees the full picture.

---

## Why It Lives in Your Repo

Sentinel is not a standalone tool. It's designed to live inside your coding agent's context window. That's the architectural choice that makes everything else work.

Your coding agent needs three things to build a useful agent:

1. **The Sentinel spec** — the architecture, the pipeline patterns, the schema conventions
2. **Your codebase** — your tables, your models, your domain logic, your existing queries
3. **Your intent** — "watch for X"

All three are in context at the same time. That's why the agent can pick the right tables, write the right queries, and build something that works against your actual system. A standalone tool would need you to configure all of that manually.

By being in-repo:
- New agents follow patterns from existing ones automatically
- Sensor queries reference your actual schema, not an abstraction
- Prompts use your domain concepts because the coding agent read your codebase
- The system upgrades itself — keep the docs, build a new agent, and the control theory architecture improves with each one

---

## The Compound Effect

After using Sentinel for a while, something shifts. Your prompts get sharper. You start thinking about what data to capture, what to measure, how to design for observability. Your database schemas get more intentional. Your systems get built to be watched — before you even set up the agent.

That wasn't your idea. It was engineered in. The feedback loop doesn't just improve the agents. It improves how you think about systems.

Sentinel built that instinct. **Sentinel builds Sentinel.**

---

## Not Just Code

The suggestions agents produce aren't limited to code changes. Some need:

- A designer to rethink a user flow
- A PM to reprioritize the roadmap
- A team conversation about trade-offs
- A product decision that no amount of code can resolve

The agent surfaces the evidence and the location. What happens next is a human decision. Engineering, design, and product can each define their own agents on the same data — same system, different lenses, different insights.

---

## Architecture Reference

For the full technical specification:

| Document | What it covers |
|---|---|
| [`01_ARCHITECTURE.md`](01_ARCHITECTURE.md) | Control theory foundation, canonical 11-step pipeline, agent memory, cost guardrails |
| [`02_BUILD_PLAN.md`](02_BUILD_PLAN.md) | Build sequence, DB schema, CLI commands, agent types, simulation mode |
| [`03_BUILD_LOG.md`](03_BUILD_LOG.md) | Version-by-version state propagation — grows with each build |

**Tech:** TypeScript, SQLite (better-sqlite3 + Drizzle ORM), Vercel AI SDK + Anthropic, Zod, Commander, Vitest
