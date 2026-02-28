# Agent Lifecycle — Spawning, Heartbeat, Dormancy, and Respawning

> *An agent going down is inevitable. An agent going down without anyone noticing? That's a design failure.*

Pentagon agents aren't permanent processes. They exhaust context windows, crash, complete their mission, or simply need fresh starts. The lifecycle system — spawning, heartbeat, dormancy, and respawning — determines whether these transitions are graceful or catastrophic.

---

## Lifecycle States

Every Pentagon agent moves through a defined sequence:

```
Spawn → Active ↔ Needs Input → Dormant → Delete
                      ↕
                    Error
```

| State | Ring Color | What's Happening |
|-------|-----------|-----------------|
| **Active** | Pulsing yellow | Agent is working — tool calls in progress |
| **Needs Input** | Green | Agent finished its turn, waiting for your next prompt |
| **Dormant** | Gray | Session paused, state preserved |
| **Error** | Red | Something went wrong — stale detection or crash |
| **Idle** | Green | Ready, no active work |

### State Transitions

Pentagon's StatusEngine manages transitions deterministically. The same state + signal always produces the same result. Key signals:

- `sessionStarted` → Agent goes active
- `toolUsed` → Refreshes active state
- `sessionEnded` → Agent goes idle
- `userPaused` → Agent goes dormant
- `userResumed` → Agent reactivates
- `errorOccurred` → Agent enters error state

**Stale detection:** If an agent stays `active` for 60+ seconds without a `toolUsed` signal, it auto-transitions to `error`. This catches hung processes and crashed sessions.

---

## Spawning

### From the Canvas

1. Click an empty grid cell (or use keyboard shortcuts)
2. **Select a workspace** (required — every agent must belong to a workspace)
3. Select a project directory
4. Name the agent
5. Pick a model (Opus, Sonnet, or Haiku)
6. Set permissions (Full Auto or Custom)
7. Choose branch strategy (existing branch or new worktree)
8. Click Create

The agent materializes as a desk on the canvas. You can also spawn sub-agents from existing agents via the canvas context menu.

### Cloning

Click the **+** indicator on adjacent empty cells to clone an existing agent. The clone gets its own git worktree (isolated branch) while inheriting the original's identity files. This is the fastest way to parallelize work on the same repo.

### Spawning via Skill

The `/manage-pentagon-agent` skill allows programmatic agent creation — useful when a manager agent needs to spawn workers.

### The initialMessage Power Move

When spawning programmatically, the `initialMessage` field tells the agent exactly what to do first. Don't waste it on pleasantries.

**Weak:**
```
Welcome! You are a testing agent. Please get oriented.
```

**Strong:**
```
You are a testing agent. Your FIRST actions:
1. Read tasks.json for your current assignments
2. Run the test suite in apps/api/ to establish a baseline
3. Report results to FeatureManager <UUID> via /send-pentagon-message
Start immediately.
```

The strong version eliminates the "orientation phase" where a fresh agent spends time figuring out what to do.

---

## Git Worktree Isolation

Pentagon uses git worktrees to prevent agents from stepping on each other's code:

- The **first agent** on a repo works on the existing branch
- **Every subsequent agent** gets its own worktree with an isolated branch

This means two agents can work on the same repository simultaneously without merge conflicts. Each agent sees its own working copy.

### Worktree Implications

- Changes are isolated until explicitly merged
- Agents can't see each other's uncommitted work
- The human (or a designated merge agent) handles integration
- Each worktree has its own node_modules, build artifacts, etc.

---

## Heartbeat System

The heartbeat controls what happens when an agent's session ends. This is one of Pentagon's most powerful features for long-running operations.

### Modes

| Mode | Behavior | Best For |
|------|----------|---------|
| **Manual** (default) | Agent runs only when you start it | One-off tasks, exploration |
| **Always On** | Auto-restarts when session ends | Continuous monitoring, critical services |
| **Interval** | Wakes periodically to check tasks | Polling workflows (PR reviews, inbox checks) |
| **Scheduled** | Runs on a cron schedule | Daily reports, maintenance windows |

### Configuration

Heartbeat settings live in `heartbeat-config.json` inside the agent's directory. You can also configure them through the Heartbeat tab in the Detail Panel.

### Always On Details

- Up to 3 restart attempts with 30-second delays
- If all retries fail → error state
- The agent re-reads SOUL.md, PURPOSE.md, MEMORY.md, and tasks.json on each restart
- Ideal for agents that need to be continuously available

### Auto-Restart Safety

Pentagon includes a **crash cap** that prevents restart storms. If an agent repeatedly crashes (e.g., due to a broken hook, corrupted identity file, or environment issue), Pentagon stops retrying after the cap is reached and sends a **desktop notification** alerting you that the agent exhausted its restart budget.

Without the crash cap, a crashing Always On agent would loop endlessly — restarting, crashing, restarting — consuming resources and flooding logs. The crash cap turns this into a single notification: "Agent X exhausted restart attempts. Investigate and manually restart."

**If you see this notification:**
1. Check the agent's terminal output for crash details
2. Review recent changes to identity files or hooks
3. Fix the root cause
4. Manually restart the agent

### Interval Pattern

Set an interval (e.g., 5 minutes). The agent:
1. Wakes up
2. Reads tasks.json for pending work
3. Executes any backlog or inProgress items
4. Goes dormant
5. Wakes again after the interval

This is excellent for review bots, inbox processors, or any agent that needs to periodically check for new work.

---

## Dormancy

Dormancy is a user-initiated pause. The agent's session is suspended, but its identity files and state are preserved. Think of it as putting an agent to sleep.

### When to Use Dormancy

- Agent is between tasks and you don't want it consuming resources
- You need to temporarily free up compute for other agents
- Work is paused but will resume later

### Dormancy vs. Termination

| | Dormancy | Termination |
|---|---|---|
| Session state | Preserved | Lost |
| Identity files | Intact | Intact (but no active process) |
| Resume | Quick (respawn reads identity) | Full restart required |
| Canvas visibility | Gray ring | Removed |
| Cost | Zero (no active process) | Zero |

**Prefer dormancy over termination** when you expect the agent to resume. The respawn is faster because the context is fresh.

---

## Context Exhaustion

Every Claude model has a finite context window. As an agent works — reading files, processing messages, writing code — its context fills up. When it gets low:

1. **Response quality degrades** — Shorter, less detailed outputs
2. **Consistency drops** — Agent forgets earlier context, contradicts itself
3. **Tool calls fail** — Complex multi-step operations break
4. **report.json stops updating** — The agent forgets the discipline

### Warning Signs

- Agent re-asks questions it already answered
- Reports become repetitive or stale
- Agent claims tasks are done that clearly aren't
- Responses get noticeably shorter

### The Proactive Respawn

Don't wait until an agent is completely non-functional. At the first sign of degradation:

1. Read the agent's current MEMORY.md, tasks.json, and report.json
2. Note any in-progress work
3. Respawn the agent (Pentagon preserves identity files)
4. The fresh agent re-reads its identity stack and continues

---

## Respawning

Respawning gives an agent a fresh context window while preserving its identity. Pentagon handles the mechanics — the key is ensuring the identity files are up to date before respawning.

### Pre-Respawn Checklist

1. **MEMORY.md** — Does it contain everything the agent learned this session? If not, update it.
2. **tasks.json** — Is the task board accurate? Set completed tasks to `done` with outcomes, ensure pending items are clear.
3. **report.json** — It will be read on respawn; make sure it reflects current state.
4. **SOUL.md** — Any behavioral adjustments? Add them now; they'll take effect on respawn.

### The UUID Problem

When an agent is respawned, it may get a new UUID. Every other agent that references the old UUID in their SOUL.md hierarchy section now has a stale reference. Messages sent to the old UUID go nowhere.

**After respawning, update all UUID references:**

1. Identify all agents whose SOUL.md references the old UUID
2. Edit each SOUL.md to replace old UUID → new UUID
3. Send a message to the respawned agent's supervisor notifying them of the new UUID

This is the most commonly missed step in the respawn process. One stale UUID silently breaks an entire communication chain.

### Emergency Replacement

When an agent is too degraded for graceful respawn:

1. Terminate immediately
2. Audit what the agent did since degradation began — check files modified, messages sent
3. Roll back any bad output if needed
4. Spawn a fresh agent with the same SOUL.md and MEMORY.md
5. Update UUID references across the team
6. Notify the supervisor

---

## Lifecycle Patterns

### The Rolling Restart

For long-running operations that will exceed context windows:

1. Monitor agent behavior for degradation signs
2. When degradation starts, update MEMORY.md with current progress
3. Respawn the agent
4. The fresh agent reads MEMORY.md and continues seamlessly

This keeps agents perpetually fresh. The cost is a brief pause during respawn.

### The Hot Spare

Keep a pre-configured agent template (SOUL.md, PURPOSE.md, MEMORY.md, tasks.json) ready for critical roles. If your lead agent goes down, you don't want to spend 15 minutes writing identity files from scratch.

### The Swarm Reset

When multiple agents have stale context, conflicting task boards, or broken UUID references:

1. Document the desired end state
2. Terminate all affected agents
3. Respawn in hierarchical order: Lead → Managers → Workers
4. Each tier confirms operation before the next is spawned
5. This takes ~10 minutes for a team of 10 but cleanly resolves cascading state issues

---

## Key Takeaways

1. **Heartbeat keeps agents alive.** Use Always On for critical roles, Interval for periodic checks.
2. **Dormancy over termination.** If the agent will resume, pause it — don't kill it.
3. **Respawn proactively.** Don't wait for full context exhaustion. First sign of degradation = time to respawn.
4. **Update UUID references.** The most commonly missed step. One stale UUID = silent communication failure.
5. **initialMessage is your superpower.** Get fresh agents productive in seconds by telling them exactly what to do first.
6. **Identity files are your insurance.** Well-maintained SOUL.md and MEMORY.md make every restart painless.

---

Next: [Chapter 5 — The Auditor Pattern](05-auditor-pattern.md) — Independent monitoring that catches what the hierarchy misses.
