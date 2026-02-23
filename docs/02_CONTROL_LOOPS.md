# Sentinel — Control Loop Architecture

## Why Multiple Feedback Loops

A system that produces suggestions but never learns whether those suggestions were good is an open loop — it will drift, produce noise, and lose trust. Sentinel is designed as a closed-loop system with five distinct feedback mechanisms:

1. **Inner loop** — governs each individual agent's suggestions
2. **Outer loop** — governs the Sentinel system itself via the System Architect
3. **Data pipeline loop** — governs data source freshness and completeness (see `03_DATA_PIPELINE.md`)
4. **Integrity loop** — governs structural correctness of critical data paths
5. **Build loop** — governs how Sentinel itself is constructed, version by version (see `04_BUILD_SYSTEM.md`)

---

## Design Philosophy: Aerospace-Grade Negative-Feedback Control

Sentinel borrows its architecture from aerospace flight control systems. The same principles that keep aircraft stable under turbulence keep AI agent systems stable under shifting data, changing user behavior, and imperfect LLM outputs.

### Negative Feedback, Not Positive Feedback

Positive feedback amplifies error — a microphone pointed at a speaker produces a screech that grows until something clips. In an AI agent system, positive feedback means: agent detects a problem → overreacts → creates new problems → detects those → overreacts again → system oscillates.

Sentinel uses **negative-feedback control with stability margins**. Every loop measures the error between actual state and desired setpoint, then applies corrections that _reduce_ the error:

- Agent detects a metric below target → suggests improvement → improvement deployed → metric recovers → agent sees reduced error → produces fewer suggestions for that domain
- Agent detects a data pipeline failure → suggests credential rotation → credential fixed → ingestion recovers → agent confirms resolution and goes quiet

The correction always opposes the error. When the error is gone, the output is quiet. This is the thermostat principle — heat turns on when temperature is below setpoint, turns off when it reaches setpoint.

### Why Closed-Loop Matters

Most AI agent systems are open-loop: generate output, hope it's useful, never check. Open-loop systems drift. In aerospace, an open-loop autopilot flies straight until a gust hits — then it diverges. In AI agents, an open-loop system suggests features nobody wants, repeats itself, wastes money, and gets turned off within a month.

Sentinel treats every agent output as a **control signal** that must be measured against reality. The suggestion is the actuator command. The metric change (or lack thereof) is the sensor reading. The difference between expected and actual outcome is the error signal that drives the next correction cycle.

---

## The Five Loops Mapped to Control Theory

| Loop                   | Control Theory Analog                | Role                                                       | Bandwidth       |
| ---------------------- | ------------------------------------ | ---------------------------------------------------------- | --------------- |
| **Inner loop**         | Rate damper / stability augmentation | Per-suggestion feedback — did this move a metric?          | Days to weeks   |
| **Outer loop**         | Flight envelope protection           | System-level health — are agents producing value?          | Weeks to months |
| **Data pipeline loop** | Sensor calibration and redundancy    | Is the upstream data accurate, fresh, and complete?        | Hours to days   |
| **Integrity loop**     | Structural health monitoring         | Are critical data paths producing correct, traceable data? | Hours to days   |
| **Build loop**         | Adaptive control law updates         | Is the system itself being constructed correctly?          | Per-version     |

The inner loop is the fastest product loop — it catches individual bad suggestions. The outer loop is slower but broader — it catches systemic problems like an agent that's fundamentally misconfigured. The data pipeline loop ensures the "sensors" (production data) are trustworthy. The integrity loop catches structural damage in critical write paths. The build loop modifies the "control laws" (agent code and configuration) themselves.

---

## Stability Constraints

Every negative-feedback system needs constraints to maintain stability margins. Without them, even a correctly-signed feedback loop can overshoot, ring, or diverge.

| Constraint                                              | What It Prevents                        | Aerospace Analog                               |
| ------------------------------------------------------- | --------------------------------------- | ---------------------------------------------- |
| **Cost guardrails** (per-run and monthly budgets)       | Runaway LLM spending                    | Actuator saturation limits                     |
| **Dry-run mode**                                        | Untested agents affecting production    | Simulation before flight                       |
| **Human review**                                        | Bad suggestions reaching implementation | Pilot authority / supervisory override         |
| **One version at a time** (build loop)                  | Parallel changes that conflict          | Rate limiting on control law updates           |
| **Scheduling separation** (different agent frequencies) | Mode coupling between loops             | Bandwidth separation in multi-loop controllers |
| **Max suggestion cap**                                  | Flooding the review queue               | Output saturation limits                       |
| **Settling periods**                                    | Acting on data before it stabilizes     | Sensor settling time                           |

### Mode Coupling Risk

In aerospace, mode coupling occurs when two control loops interact at similar frequencies and amplify each other. In Sentinel, the equivalent risk is:

1. A data pipeline agent suggests changing a data source
2. The change shifts the data distribution
3. A product agent sees the new data and produces very different suggestions
4. Those suggestions get rejected (the data shift was unexpected)
5. The product agent interprets rejections as "those kinds of suggestions are bad" and over-corrects

**Mitigation:** Scheduling separation. The data pipeline loop and the product suggestion loop run at different frequencies and on different schedules. When a data change lands, there's a settling period before product agents act on the new data.

### State Propagation

The "Notes for next step" mechanism in the build loop is a **discrete-time state propagator**. Each version passes its terminal state (what was built, what was learned, what's next) as initial conditions to the next version. No context is lost between iterations.

---

## Inner Loop — Per-Agent Feedback

Each agent operates its own feedback cycle:

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  Agent Runs  │────▶│  Suggestion  │────▶│  Human       │
│  (scheduled) │     │  Generated   │     │  Review      │
└──────────────┘     └──────────────┘     └──────┬───────┘
       ▲                                         │
       │                                         ▼
       │                                  ┌──────────────┐
       │                                  │ Implemented? │
       │                                  │ Metric       │
       │                                  │ tracked      │
       │                                  └──────┬───────┘
       │                                         │
       │              ┌──────────────┐           │
       └──────────────│ Agent sees   │◀──────────┘
                      │ updated data │
                      │ on next run  │
                      └──────────────┘
```

### What the inner loop tracks

| Field                        | Purpose                                       |
| ---------------------------- | --------------------------------------------- |
| `suggestion.status`          | accepted / rejected / implemented / ignored   |
| `suggestion.rejectionReason` | Why it was rejected (feeds back to agent)     |
| `suggestion.targetMetric`    | What metric the suggestion aimed to improve   |
| `suggestion.metricBefore`    | Metric value at time of suggestion            |
| `suggestion.metricAfter`     | Metric value 7/14/30 days post-implementation |

### What it catches

- Suggestions that were already implemented (avoids duplicates via dedup)
- Whether a recommended change actually moved a metric
- Rejection patterns the agent should learn from
- Shifts in data that make previous assumptions stale

---

## Outer Loop — System-Level Feedback (The System Architect)

The System Architect is a dedicated meta-agent that monitors the health of all other agents.

```
┌─────────────────────────────────────────────────────────┐
│                 Inner Loop Agents                         │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────────┐  │
│  │ Agent A  │ │ Agent B  │ │ Agent C  │ │ Agent D    │  │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └─────┬──────┘  │
│       ▼             ▼            ▼              ▼         │
│  ┌────────────────────────────────────────────────────┐   │
│  │         Agent Run Logs + Suggestions               │   │
│  └──────────────────────┬─────────────────────────────┘   │
└─────────────────────────┼─────────────────────────────────┘
                          ▼
              ┌───────────────────────┐
              │   System Architect    │
              │                       │
              │   Monitors:           │
              │   - Acceptance rates  │
              │   - Cost per agent    │
              │   - Data source gaps  │
              │   - Missing sensors   │
              │   - Agent errors      │
              └───────────┬───────────┘
                          ▼
              ┌───────────────────────┐
              │   System Suggestions  │
              │                       │
              │   - Prompt rework     │
              │   - New agent types   │
              │   - Data source fixes │
              │   - Cost optimization │
              └───────────────────────┘
```

### What the System Architect outputs

| Type                  | Example                                                                                                        |
| --------------------- | -------------------------------------------------------------------------------------------------------------- |
| **Prompt rework**     | "Agent A has 20% acceptance rate. Common rejection: 'not actionable.' Add implementation specifics to prompt." |
| **New agent**         | "No agent monitors test flakiness. CI data shows 12% flaky rate. Propose: Test Stability Monitor."             |
| **Data source fix**   | "Analytics API credentials expired. Ingestion has failed for 48 hours."                                        |
| **Cost optimization** | "Agent C uses 8K tokens/run but produces 1 suggestion. Reduce context to top-20 items."                        |
| **Agent retirement**  | "Agent D has 0% acceptance rate over 8 weeks. Disable and investigate."                                        |

---

## How the Loops Interact

| Property            | Inner Loop                         | Outer Loop                                  |
| ------------------- | ---------------------------------- | ------------------------------------------- |
| **Frequency**       | Every agent run                    | Weekly or bi-weekly                         |
| **Scope**           | One agent's suggestions            | All agents, all data sources                |
| **Output**          | Domain suggestions                 | System improvement suggestions              |
| **Feedback signal** | Did this suggestion move a metric? | Is this agent producing useful suggestions? |
| **Timescale**       | Days to weeks                      | Weeks to months                             |

The outer loop prevents Sentinel from becoming a static tool that degrades over time. As the product evolves, the System Architect ensures Sentinel evolves with it.

---

## Failure Modes

### Without the inner loop

- Agents repeat suggestions that were already implemented
- No way to know if a suggestion actually helped
- Team loses trust because suggestions feel random
- Agents can't learn from rejection patterns

### Without the outer loop

- A broken data source silently degrades suggestion quality for weeks
- Costs creep up with no visibility
- The system works well initially but slowly becomes irrelevant
- No one notices when a new agent type would unlock significant value

### Without either loop

Open-loop system. Suggestions are fire-and-forget. No learning, no adaptation, no quality signal. Expensive, noisy, and quickly abandoned.
