# Hierarchy Design — Structuring Teams That Scale Without Chaos

> *Flat teams feel efficient until agent #6. Then coordination collapses, nobody knows who's doing what, and the human becomes the accidental manager.*

A single agent is straightforward. Two or three agents can self-coordinate with shared task files and some messaging. Beyond that, you need structure. This chapter covers the hierarchy and team patterns that keep multi-agent operations running cleanly.

---

## Pentagon's Organizational Building Blocks

Pentagon gives you two grouping mechanisms:

### Pods (Spatial)
Colored rectangular regions on the canvas — like rugs on a floor. Agents placed within a pod's boundaries automatically belong to that pod. Pods have shared task files, MEMORY.md, and DISCUSSION.md.

**Best for:** Grouping agents that work on the same project or codebase.

### Teams (Logical)
Sidebar-only groupings created manually. Agents are assigned to teams explicitly, regardless of canvas position. Teams also have shared task files, MEMORY.md, and DISCUSSION.md.

**Best for:** Cross-cutting concerns that span multiple projects. A "Review Team" might include agents from frontend, backend, and infrastructure pods.

| | Pods | Teams |
|---|---|---|
| Visibility | Canvas (spatial) | Sidebar (logical) |
| Membership | Automatic (position-based) | Manual (assigned) |
| Nesting | — | Flat |
| Best for | Project clusters | Workflow roles |

You can use both simultaneously. An agent can sit in a "Backend" pod on the canvas while also belonging to a "Code Review" team in the sidebar.

---

## The Three-Tier Pattern

For teams of 5+ agents, a three-tier hierarchy consistently works:

```
┌──────────────────────────────┐
│       Lead / Coordinator     │  Tier 1: Strategy & decisions
│    (Opus model recommended)  │
└──────────┬───────────────────┘
           │
    ┌──────┴──────┐
    │             │
┌───▼───┐   ┌────▼────┐
│Manager│   │Manager  │         Tier 2: Decomposition & verification
│  (A)  │   │   (B)   │
└───┬───┘   └────┬────┘
    │             │
┌───┼───┐   ┌────┼────┐
│   │   │   │    │    │
▼   ▼   ▼   ▼    ▼    ▼        Tier 3: Execution
W1  W2  W3  W4   W5   W6
```

### Tier 1: Lead / Coordinator
- Receives directives from the human
- Translates strategy into goals
- Delegates to managers
- Makes go/no-go decisions
- **Never writes code**

### Tier 2: Managers
- Breaks goals into atomic, assignable tasks
- Dispatches tasks to workers via message
- Verifies completed work
- Reports status up to the lead
- **Never writes code**

### Tier 3: Workers
- Receives task assignments
- Executes the work
- Reports completion back to manager
- Owns a specific domain (see Domain Boundaries below)

---

## The No-Code-Manager Rule

This deserves its own section because it's the most common failure mode at scale.

### The Problem

A manager receives a task and decides to "just quickly handle it" instead of delegating. The moment a manager starts coding:

1. **They stop managing.** While writing code, they're not tracking other workers.
2. **Workers go idle.** Completed tasks don't get reviewed. New tasks don't get dispatched.
3. **Parallel capacity dies.** A manager coding is one unit of work. A manager managing is N units of work (where N = number of workers).

### The Math

With 4 workers, a manager represents 4x leverage:

- Manager codes for 30 min → 30 min of work done
- Manager manages for 30 min → up to 120 min of parallel work orchestrated

The leverage gap grows with team size.

### The Fix

Put this in every manager's SOUL.md as an absolute constraint:

```markdown
## Constraint: No Code
I NEVER write code. Not to "help," not for "quick fixes," not for "simple things."
If I'm about to write code, I delegate it to a worker instead.
```

Identity-level constraints are more durable than instructions. When an agent sees `NEVER` in its SOUL.md, it treats it as a hard boundary.

---

## Domain Boundaries

Every worker agent needs a clear domain — the scope of what they own and what they don't touch.

### Why Boundaries Matter

Without clear domains:
- Two agents modify the same file simultaneously → merge conflicts
- Nobody owns the integration between modules → gaps
- "I thought they were handling that" → dropped work

### Defining Domains

| Agent | Owns | Does NOT Touch |
|-------|------|---------------|
| AuthWorker | Authentication, authorization, JWT | Database schema, UI |
| UIWorker | Components, layouts, styling | API routes, database |
| DataWorker | Database, queries, migrations | Frontend, auth logic |
| TestWorker | Test suite, CI pipeline | Production code changes |

### The Boundary Test

For any piece of work, you should be able to instantly answer "which agent owns this?" If you can't, the boundaries are too fuzzy. Sharpen them before dispatching.

---

## Configuring Hierarchy in SOUL.md

Every agent's SOUL.md should include explicit hierarchy information with UUIDs:

### Lead Agent

```markdown
## Hierarchy
- Reports to: Human
- Managers:
  - FeatureManager <UUID-1> — Feature development
  - InfraManager <UUID-2> — Infrastructure and deployment
```

### Manager Agent

```markdown
## Hierarchy
- Reports to: Lead <UUID-0>
- Peers: InfraManager <UUID-2>
- Workers:
  - AuthWorker <UUID-3>
  - UIWorker <UUID-4>
  - DataWorker <UUID-5>
```

### Worker Agent

```markdown
## Hierarchy
- Reports to: FeatureManager <UUID-1>
- Peers: UIWorker <UUID-4>, DataWorker <UUID-5>
```

**Always include UUIDs.** Names can be ambiguous; UUIDs are the canonical identifier for Pentagon messaging. When an agent is respawned with a new UUID, update every SOUL.md that references the old one (see [Chapter 4 — Lifecycle](04-lifecycle.md)).

---

## Communication Flow

Information flows through the hierarchy, not around it:

```
Human → Lead → Manager → Workers
Workers → Manager → Lead → Human
```

### Why not skip levels?

When a lead sends tasks directly to workers, bypassing the manager:
- The manager's task board goes stale
- Workers may get conflicting priorities from the lead and the manager
- No one at the manager level tracks the work

**Exception:** Emergency overrides where the lead must intervene directly. Even then, inform the manager immediately after.

---

## Scaling Guidelines

### Workers per Manager
**3-5 is the sweet spot.** Beyond 5, a manager struggles to track all workers, verify deliveries, and dispatch new work without becoming a bottleneck.

### Managers per Lead
**2-4 works well.** Beyond 4, the lead can't stay current on all fronts.

### When to Add a Worker
- A domain is too large for one agent
- Task queues are growing because a worker is overloaded
- A new capability area doesn't fit existing domains

### When to Add a Manager
- Worker count under a single manager exceeds 5
- Work streams are distinct enough to warrant separate oversight

### Model Selection by Tier

Pentagon lets you assign different Claude models to each agent:

| Tier | Recommended Model | Reasoning |
|------|------------------|-----------|
| Lead | Opus | Complex reasoning, strategic decisions |
| Manager | Sonnet | Task decomposition, coordination |
| Worker (complex) | Sonnet | Nuanced implementation work |
| Worker (simple) | Haiku | Straightforward tasks, cost-efficient |

Using Haiku for simple workers keeps costs manageable as you scale.

---

## Team Patterns

### The Product Team
One codebase, multiple concerns.
```
Lead → FeatureManager → [Frontend, Backend, Database, Testing]
     → InfraManager   → [CI/CD, Deployment, Monitoring]
```

### The Pipeline Team
Sequential workflow (content creation, data processing, CI/CD).
```
Lead → PipelineManager → [Ingest, Transform, Validate, Output]
```

### The Research Team
Exploration and analysis.
```
Lead → ResearchManager → [Analyst A, Analyst B, Synthesizer]
```

### The Dual-Manager Team
Distinct workstreams with some shared resources.
```
Lead → ManagerA → [Worker1, Worker2, SharedWorker]
     → ManagerB → [Worker3, Worker4, SharedWorker]
```

⚠️ **Shared workers need priority rules.** If both managers assign work, who takes precedence? Define this explicitly in the shared worker's SOUL.md.

---

## The Flat Team Anti-Pattern

Running 6+ workers with no manager feels efficient — no management overhead! It fails because:

- No one tracks who's doing what
- No one verifies completed work
- Workers make conflicting decisions with no coordinator
- The human becomes the accidental manager

**If you have more than 3 agents, designate a manager.** The coordination overhead pays for itself immediately.

---

## Key Takeaways

1. **Use pods for spatial grouping and teams for logical grouping.** They solve different problems.
2. **Three tiers scale.** Lead → Manager → Worker works from 5 to 15+ agents.
3. **Managers never code.** Enforce this in SOUL.md as an absolute constraint.
4. **Clear domain boundaries.** Every task should have an unambiguous owner.
5. **3-5 workers per manager.** Beyond that, add another manager.
6. **UUIDs in every SOUL.md hierarchy section.** Names are convenient; UUIDs are authoritative.
7. **Don't go flat.** More than 3 agents without a manager is a coordination failure waiting to happen.

---

Next: [Chapter 4 — Agent Lifecycle](04-lifecycle.md) — Spawning, heartbeat, dormancy, and respawning without losing context.
