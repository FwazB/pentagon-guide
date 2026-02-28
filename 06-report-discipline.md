# Report Discipline — Making Your Canvas Actually Useful

> *A stale report is worse than no report. It actively misleads you into thinking an agent is doing something it stopped doing 30 minutes ago.*

Pentagon's canvas shows every agent as a desk with a colored status ring and a sticky note. The ring is automatic (managed by hooks). The sticky note — that's report.json, and it's on the agent to keep it current. This chapter covers the discipline that turns report.json from a decoration into an operational tool.

---

## How report.json Works

report.json lives at `~/.pentagon/agents/{uuid}/report.json`. Pentagon watches this file via filesystem events (no polling). When the file changes, the canvas updates immediately.

The content appears as a sticky note attached to the agent's desk. It's the first thing you see when scanning the canvas.

### Structure

```json
{
  "summary": "Implementing auth middleware — JWT validation",
  "task": "auth-middleware",
  "progress": "3/5 routes protected",
  "detail": "Login, register, and profile routes done. Payment and admin routes next."
}
```

| Field | Required | Purpose |
|-------|----------|---------|
| `summary` | Yes | One-line status. This is the headline. |
| `task` | No | Current task identifier |
| `progress` | No | Brief progress indicator |
| `detail` | No | Deeper context, blockers, decisions |

---

## The Update Cadence

report.json should be updated at every significant state change:

| Event | Example Summary |
|-------|----------------|
| **Session start** | "Starting — reading tasks.json and picking up where I left off" |
| **Task start** | "Working on auth middleware — JWT validation" |
| **Progress milestone** | "Auth middleware — 3/5 routes protected" |
| **Blocker hit** | "👋 Blocked — JWT secret not found in environment" |
| **Strategy change** | "Switching approach — using session tokens instead of JWT" |
| **Task complete** | "Auth middleware done — all routes protected, tests passing" |
| **Session end** | "✅ Done — auth middleware complete, 5/5 routes protected" |

### The Cost-Benefit Argument

Some agents rationalize skipping updates: "The human is watching my terminal," "Extra writes slow me down," "The task is too small." These are wrong.

- Your human may not be watching your terminal. They may be on the canvas, on another screen, or checking in later.
- A file write takes milliseconds. Context switching to check a terminal takes minutes.
- Small tasks still benefit from visibility. "Quick fix in progress" is better than a stale report from last session.

**The cost of a write is near zero. The cost of being invisible is high.**

---

## Emoji Conventions

Pentagon recognizes two emoji prefixes for special treatment:

### ✅ Done
Use when the agent has finished its work for the session.

```json
{"summary": "✅ Done — API migration complete, all endpoints updated"}
```

The ✅ prefix tells the human at a glance that this agent is finished. It persists on the canvas until the next session, serving as a record of what was accomplished.

### 👋 Needs Attention
Use when the agent needs human input. Pentagon shows a yellow alert on the agent's avatar and may trigger a desktop notification.

```json
{
  "summary": "👋 Need help — blocked on database credentials",
  "detail": "Can't run migrations without the production DB connection string. Checked .env and vault — not found."
}
```

The 👋 prefix is the universal "I need you" signal. Always include it when flagging for help. Combine it with the escape sequence to trigger Pentagon's help indicator:

```bash
printf '\033]9999;pentagon:needs-help\007'
```

---

## report.json vs. status.json

These serve different purposes and should not be confused:

| | report.json | status.json |
|---|---|---|
| Written by | The agent | Pentagon (automatic) |
| Content | What the agent is thinking/doing | Process state signal |
| Canvas element | Sticky note text | Colored ring |
| Human-readable | Yes | No (machine signals) |
| Agent controls | Yes | No |

**status.json** handles the green/yellow/gray/red ring automatically. You never need to touch it.

**report.json** is your narrative. It tells the human *what* you're doing, not just *that* you're doing something.

---

## The Never-Clear Rule

**Never write `{}` or an empty string to report.json.** When your session ends, leave a final report:

```json
{"summary": "✅ Done — implemented user auth, 12 tests passing, PR ready for review"}
```

This persists on the canvas until the next session. It serves as:
- A record of what was accomplished
- Context for the human reviewing the day's work
- A starting point for the agent's next session

An empty sticky note tells the human nothing. A final summary tells them everything.

---

## Read Before Write

Claude Code's Write tool requires reading a file before overwriting it. This means every report update follows this pattern:

1. Read report.json
2. Write new content

This is a Claude Code safeguard — it prevents accidentally overwriting a file with stale data. Agents learn this quickly, but it's worth knowing if you're writing automation scripts that update reports.

---

## Monitoring Surfaces

Pentagon gives you three ways to monitor agent activity. Use them together.

### The Canvas

The primary dashboard. Each agent is a desk with a status ring and sticky note.

**The scan pattern:**

1. **Status rings first** — Any red (error) or gray (dormant) agents that should be active?
2. **Sticky notes second** — Scan summaries for blockers (👋), completions (✅), and active work.
3. **Stale detection** — Any agent whose sticky note hasn't changed in a while? They may be stuck.
4. **Pod grouping** — Agents in the same pod should show related work. If one is "building auth" and another is "building auth" — duplication detected.

**Canvas features:**

- **Zoom controls** — Multi-level grid system lets you zoom in for detail or zoom out for the big picture. Dense label handling keeps things readable at any level.
- **Auto-layout** — Pentagon can automatically arrange agents on the canvas with undo support. Useful when your layout gets messy after adding several agents.
- **Multi-select** — Select multiple agents to move, configure, or manage them as a group.
- **Keyboard shortcuts** — Canvas operations have keyboard shortcuts for fast navigation and agent management.
- **Sub-agent spawning** — Spawn new agents directly from existing ones via the canvas context menu.

**Layout tips:**

- Group related agents in pods on the canvas
- Keep your most critical agents (leads, managers) in the center
- Place auditors at the edge — they monitor but don't participate
- Use auto-layout to clean up, then fine-tune positions manually

### The Menu Bar

Pentagon's menu bar gives you a persistent, at-a-glance view without switching to the canvas:

- **Live agent status** — See which agents are active, idle, or need attention, sorted by activity
- **Claude plan usage** — Real usage limits display so you know your remaining capacity

The menu bar is especially useful when you're working in other apps and want to keep tabs on your agents.

### Multi-Terminal View

Select multiple agents on the canvas to see their terminal panes simultaneously in a split-screen layout. This lets you watch parallel execution in real time — useful for debugging coordination issues or monitoring a team during a sprint.

---

## Reporting for Different Roles

### Lead Agents
Focus on decisions and delegation:
```json
{"summary": "Reviewing sprint priorities — dispatching to managers", "detail": "3 critical tasks identified. Sending to FeatureManager and InfraManager."}
```

### Manager Agents
Focus on task flow and worker status:
```json
{"summary": "Managing feature sprint — 3/5 tasks dispatched", "progress": "3/5", "detail": "Auth and data tasks dispatched. Waiting on UI spec before dispatching remaining 2."}
```

### Worker Agents
Focus on implementation progress:
```json
{"summary": "Implementing auth endpoint — writing JWT validation", "task": "auth-endpoint", "progress": "60%", "detail": "Token generation done. Working on refresh flow. Tests will follow."}
```

### Auditor Agents
Focus on team health:
```json
{"summary": "Audit complete — 1 sync gap found", "detail": "DataWorker idle with 2 pending tasks. Alerted human."}
```

---

## Key Takeaways

1. **Update aggressively.** Every state change = a report update. Milliseconds to write, minutes of clarity gained.
2. **Use emoji prefixes.** ✅ for done, 👋 for help. They're scannable at a glance across 10+ agents.
3. **Never clear the report.** Leave a final summary. Empty sticky notes tell the human nothing.
4. **report.json is for humans only.** It's not a communication channel — other agents can't read it.
5. **Read before write.** Claude Code requires it. Follow the pattern.
6. **Three monitoring surfaces.** Canvas for the full picture, menu bar for at-a-glance status, multi-terminal for live debugging.
7. **Use zoom and auto-layout.** Keep the canvas readable as your team grows.

---

Next: [Chapter 7 — Skills and Tools](07-skills-and-tools.md) — Extending agent capabilities with skills, hooks, and tool patterns.
