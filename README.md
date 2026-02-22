# Sentinel

**Rocket science so you can vibe code.**

AI lets you build fast. The hard part isn't building — it's knowing what to build, what's breaking, and what you're missing. Your codebase, your database, and your infrastructure already have every signal you need. Nobody has time to look at all of it. Sentinel does.

You describe a goal. Sentinel builds the agent, picks the data sources, writes the queries, tracks outcomes, and goes quiet when things are healthy. You don't think about which tables or which metrics. Your coding agent already sees your schema, your domain logic, and the Sentinel architecture in one context. It figures out the rest.

Under the hood, Sentinel is powered by a set of control theory equations — the same math that keeps rockets on course. Agents self-correct, learn from your feedback, and shut up when there's nothing to say. As long as you keep the original Sentinel docs in your repo, the system's capabilities upgrade themselves the moment you build your first agent. Use plan mode, reference the Sentinel docs and your codebase, and it extracts the correct solution.

Accept a suggestion and the agent measures whether it worked. Reject it and the agent remembers why. Ignore it and the agent takes the hint. Every interaction sharpens the system. Over time, your own prompts get better — you start thinking about what to measure, what data to capture, how to design for observability. That wasn't your idea. It was engineered in. The logic behind Sentinel is Sentinel. The same observe-reason-suggest loop that powers your agents is what built this system. **Sentinel builds Sentinel.**

---

## See It Work

```bash
cd examples/taskflow && npm install
npx tsx bin/sentinel.ts run --agent onboarding-analyst --dry-run
```

```
Running onboarding-analyst...
  [sensors] Querying live data from users + events tables...
  [sensors] Found 200 users, 136 activated (68.0%)
  [budget] Estimated ~207 input tokens, ~500 output tokens
  [budget] Estimated cost: $0.0081 (limit: $0.1000)
  [llm] Calling claude-sonnet-4-20250514 via generateObject()...
  [llm] Response received — 4 suggestions
  [llm] Actual tokens: 1302 (prompt: 771, completion: 531)
  [llm] Actual cost: $0.0103
  Status: completed
  Suggestions: 4

  Suggestions:
    [medium] Reduce email verification drop-off
      Implement automated email resend functionality and improve email
      deliverability to reduce the 15% drop-off at email verification step
    [high] Streamline workspace creation process
      Simplify workspace setup with templates, reduce required fields, and add
      progress indicators to decrease the 20% drop-off
    [high] Redesign team invitation flow
      Make team invitation optional or defer it to later in the onboarding
      process, as the 64% drop-off is severely impacting activation
    [high] Improve overall activation rate
      Focus on the critical funnel steps (workspace creation and team invitation)
      to increase overall activation from 68% to the target of 85%
```

4 prioritized suggestions with evidence, from live data, for ~$0.01. No API key? Use `--simulate degrading` to run with pre-computed scenarios.

The CLI is a demo. Agents return structured data — plug them into dashboards, Slack, GitHub Actions, or whatever fits.

---

## What You Get

- **Describe a goal, get an agent.** "Keep onboarding above 85%." Your coding agent reads the Sentinel spec + your schema and builds the whole thing.
- **Learns from every interaction.** Accept, reject with a reason, or ignore. The agent adapts. Two teams with identical data get different suggestions because their feedback is different.
- **Goes quiet when healthy.** A well-tuned agent has nothing to say. That silence is the signal.
- **Pennies per run.** ~$0.01 per agent execution. Budget guardrails prevent runaway costs.
- **Survives rewrites.** Some agents watch code. Others watch outcomes — API responses, metrics, business results. Those survive refactors, team handoffs, and complete rewrites. The code changes. The goal doesn't.
- **Scales to anything.** Weekend project or 50-service platform — same pattern, same feedback loop.
- **Not just code.** Suggestions might need a designer, a PM, or a team conversation. The agent surfaces evidence. What happens next is your call.

---

## How It Works

Sentinel lives in your repo. It pairs with your coding agent — that's the key design choice.

Your coding agent sees the Sentinel architecture, your codebase, and your data in one context. When you say *"Build a Sentinel agent that watches [X],"* it already knows your tables, your domain logic, and the patterns from existing agents. The agent writes itself.

```bash
npx tsx bin/sentinel.ts next     # Prints the next build prompt
npx tsx bin/sentinel.ts status   # Shows DB state and agent stats
npm test                         # 43 tests, all passing
```

No monitoring server. No agent runtime. No separate database. Agents are code in your project. They query your database, run on your infrastructure, hook into whatever scheduling you already have.

> **Deep dive:** See [`docs/WHY_SENTINEL.md`](docs/WHY_SENTINEL.md) for use cases, architecture details, and advanced patterns.

---

## Get Started

```bash
git clone <repo-url>
cd sentinel/examples/taskflow
npm install
npx tsx bin/sentinel.ts run --simulate degrading --dry-run
```

**Working example:** [`examples/taskflow/`](examples/taskflow/) — project management SaaS with realistic seed data and a fully wired Onboarding Analyst agent. See the [TaskFlow README](examples/taskflow/README.md).

**Architecture:** [`docs/01_ARCHITECTURE.md`](docs/01_ARCHITECTURE.md) — control theory foundation, canonical pipeline, memory, cost guardrails.

---

*This system is its own first suggestion. Sentinel builds Sentinel.*
