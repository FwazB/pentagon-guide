# Agent Identity — The Files That Make or Break Your Agents

> *The difference between an agent that picks up where it left off and one that starts from zero every session? Five well-written files.*

Every Pentagon agent lives in `~/.pentagon/agents/{uuid}/`. Inside that directory is a set of identity files that define who the agent is, what it knows, what it's working toward, what it's working on, and what it's doing right now. These files are the foundation of everything — resilience, auditability, handoffs, and respawns all depend on them.

This chapter covers each file, what goes in it, and the patterns that make them effective.

---

## The Identity Stack

### Core Files (You Manage These)

| File | Question It Answers | Who Reads It | Update Frequency |
|------|-------------------|--------------|-----------------|
| **SOUL.md** | Who is this agent? | The agent (via system prompt) | When identity evolves |
| **MEMORY.md** | What does it know? | The agent (via system prompt) | Every session |
| **PURPOSE.md** | What's its mission? | The agent (via system prompt) | When mission shifts |
| **tasks.json** | What's on its plate? | The agent + Pentagon | As work progresses |
| **report.json** | What's it doing now? | Humans (canvas sticky note) | Aggressively |

### Automatic Files (Pentagon Manages These)

| File | What It Does |
|------|-------------|
| **status.json** | Process state → canvas ring color (green/yellow/gray/red) |
| **ORGANIZATION.md** | Auto-generated org chart — the agent's teams, teammates, map |
| **directory.json** | Machine-readable contact list of visible agents |
| **agent-card.json** | Agent metadata (name, description, skills) |
| **delivery-receipts.jsonl** | Message delivery tracking (sent/delivered status) |
| **journal.jsonl** | Agent activity log |
| **task-notes/** | Per-task deep context files (task-notes/{task-id}.md) |

Each file serves a distinct purpose. Mixing them — putting task details in SOUL.md, or mission statements in MEMORY.md — creates confusion that compounds across sessions.

---

## SOUL.md — Who the Agent Is

SOUL.md is injected into the Claude Code system prompt when the agent spawns or respawns. It shapes how the agent thinks, communicates, and makes decisions. Think of it as the agent's professional DNA.

### What Goes In It

- **Role and expertise** — "You are a backend engineer specializing in Python/FastAPI"
- **Personality and style** — Methodical? Fast-moving? Terse or verbose?
- **Constraints** — What the agent should NOT do (this is often more important than what it should do)
- **Hierarchy** — Who it reports to, its peers, its direct reports (with UUIDs for unambiguous messaging)
- **Behavioral rules** — Critical operational rules that must persist across sessions

### Example: A Manager Agent

```markdown
## Identity
Lead engineer for the API team. Powered by Claude Sonnet 4.6.

## Role
Break strategic directives into atomic tasks. Delegate to workers.
Verify deliveries. Coordinate across the team.

## Hierarchy
- Reports to: ProjectLead <UUID>
- Workers: AuthWorker <UUID>, DataWorker <UUID>, TestWorker <UUID>

## Constraints
- NEVER write code. My job is decomposition, delegation, and verification.
- NEVER skip message delivery verification after dispatching tasks.

## Communication Rule
Every decision that affects another agent MUST be sent via
Pentagon's messaging tools. report.json is for human visibility only —
other agents cannot read it.
```

### Key Insight: Constraints Beat Instructions

Telling an agent what NOT to do is often more effective than telling it what to do. An agent with `NEVER write code` in its SOUL.md will consistently delegate. An agent told to "focus on management" will occasionally decide to "just quickly write this one function" — and then three workers sit idle while the manager debugs.

### Changes Require Respawn

SOUL.md is loaded at spawn time. Editing it while the agent runs has no effect until you respawn the agent. This is by design — it prevents identity drift mid-session.

---

## MEMORY.md — What the Agent Knows

MEMORY.md is the agent's persistent brain. It survives across sessions and respawns — the only file that makes a restarted agent smarter than a fresh one.

### What Goes In It

- **Project context** — Architecture, key dependencies, deployment details
- **Conventions** — Naming patterns, file structures, commit message formats
- **Learnings** — What worked, what failed, debugging insights, workarounds
- **Known issues** — Flaky tests, deprecated APIs, edge cases
- **Procedures** — Workflows and recipes that the agent has discovered or been taught

### Example

```markdown
## Project Context
- Monorepo structure: apps/web, apps/api, packages/shared
- API uses Express + Prisma, deployed on Railway
- Frontend is Next.js 14 with App Router

## Conventions
- Branch naming: feature/{ticket-id}-{short-description}
- Commits: conventional commits (feat:, fix:, chore:)
- PRs require at least one approval before merge

## Learnings
- Prisma migrations must run before seed scripts — order matters
- The shared package must be built before running web or api
- Rate limiter on /api/auth resets every 15 minutes, not per-request

## Known Issues
- Test suite for auth module is flaky on CI — retry once before failing
- WebSocket connections drop on Heroku after 55s idle — use heartbeat pings
```

### Memory vs. Soul

Soul defines *who* the agent is. Memory defines *what the agent knows*. A new agent with the same SOUL.md but empty MEMORY.md will behave correctly but lack context. The soul is the personality; the memory is the experience.

### Shared Memory

Teams and roles have their own MEMORY.md files for group-level context. Use agent-level MEMORY.md for individual knowledge and team/role-level MEMORY.md for shared project knowledge. This prevents every agent from duplicating the same context.

### Hard Rules Section

One of the most effective MEMORY.md patterns is a dedicated **Hard Rules** section at the top — non-negotiable operational constraints that the operator has established:

```markdown
## Hard Rules
- NO git push — commits only, human pushes
- Stay on the assigned branch — never checkout other branches
- No AI co-author attribution in commits
- Always verify test failures against clean HEAD before reporting
```

Why MEMORY.md and not SOUL.md? Hard rules are operational constraints that may change as the project evolves. SOUL.md is identity — who the agent is. MEMORY.md is knowledge — what the agent knows, including rules it must follow. When the operator issues a new rule, the agent should immediately write it to MEMORY.md so it persists across sessions.

**When one agent breaks a rule, propagate the correction to all agents.** One violation means every agent needs the reinforcement.

### Hygiene

- **Synthesize, don't log.** MEMORY.md is a reference, not a journal. Write conclusions, not play-by-play.
- **Keep it under 200 lines.** If it's longer, you're over-documenting. Split into linked files if needed.
- **Prune regularly.** Delete outdated facts. Update stale procedures.
- **Update every session.** An empty MEMORY.md after multiple sessions means the agent isn't learning.
- **Update continuously, not at session end.** Context can compress or exhaust mid-session. If your memory isn't written yet, it's lost.

---

## PURPOSE.md — What the Agent Is Here to Do

PURPOSE.md defines the agent's mission — what success looks like. Without it, agents optimize for activity instead of outcomes. Think of it as a mission brief: concrete objective, clear scope, measurable success criteria.

### What Goes In It

- **Mission** — One clear statement of what the agent is trying to accomplish
- **Deliverables** — What the agent is expected to produce
- **Scope** — What's in bounds and what's off limits
- **Success criteria** — How to know when the mission is complete

### Example

```markdown
## Mission
Build and maintain the authentication system for the API.

## Deliverables
- JWT-based auth with refresh tokens
- Role-based access control (admin, user, viewer)
- Password reset flow with email verification

## Scope
- Own: src/auth/, src/middleware/auth.ts, tests/auth/
- Do NOT touch: src/billing/, src/notifications/

## Success Criteria
- All auth endpoints pass integration tests
- Zero security vulnerabilities in OWASP top 10 categories
- Response time < 200ms for token validation
```

### Purpose vs. Soul vs. Memory

| File | Defines | Answers |
|------|---------|---------|
| SOUL.md | Identity | "Who am I?" |
| PURPOSE.md | Mission | "What am I here to do?" |
| MEMORY.md | Knowledge | "What do I know?" |

An agent with a clear PURPOSE.md stays focused. Without one, agents drift — they pick up whatever work is nearby instead of driving toward a specific outcome.

### Purpose Evolves

Don't stress about writing the perfect mission statement upfront. Start with what the agent has been asked to do, refine it as the scope becomes clearer, and update it when the mission shifts. A one-line purpose grounded in reality beats an elaborate statement full of aspirations.

---

## tasks.json — What's On the Plate

Pentagon tracks agent work through `tasks.json` — a structured JSON task board that Pentagon watches via filesystem events. When you edit it (manually or through the Detail Panel), the agent sees the changes immediately.

### Format

JSON array with structured task objects:

```json
[
  {
    "id": "auth-endpoint",
    "title": "Implement authentication endpoint",
    "status": "inProgress",
    "priority": "high"
  },
  {
    "id": "rate-limiting",
    "title": "Add rate limiting to public API routes",
    "status": "backlog",
    "priority": "medium",
    "description": "Apply per-IP rate limiting to all /api/public/ routes"
  },
  {
    "id": "db-pooling",
    "title": "Set up database connection pooling",
    "status": "done",
    "outcome": "Configured PgBouncer with 20 max connections, tested under load"
  }
]
```

### Fields

| Field | Required | Description |
|-------|----------|-------------|
| `title` | Yes | Brief, actionable task description |
| `status` | Yes | `backlog` → `inProgress` → `review` → `done` |
| `id` | No | Unique identifier for referencing in messages and task-notes |
| `priority` | No | `low`, `medium`, `high`, or `critical` |
| `description` | No | Detailed requirements, acceptance criteria |
| `dependencies` | No | Task IDs that must complete first |
| `tags` | No | Categorization labels |
| `branch` | No | Git branch associated with the task |
| `outcome` | No | What was accomplished (set when marking `done`) |
| `createdBy` | No | Who created the task (agent name or "human") |

### Status Workflow

```
backlog → inProgress → review → done
```

- **backlog** — Queued, not started
- **inProgress** — Actively being worked on
- **review** — Work complete, awaiting verification
- **done** — Finished (include an `outcome` describing what was accomplished)

### Task Notes

For tasks that need deep context — research notes, design decisions, implementation details — create a file at `task-notes/{task-id}.md`. This keeps tasks.json lean while preserving rich context for complex work.

### Best Practices

- **Keep it lean** — 5-10 active items. A 50-task board overwhelms the agent.
- **Always set `outcome` on done tasks** — Future sessions and auditors need to know what was accomplished, not just that something was checked off.
- **Clean up regularly** — Remove `done` tasks once their outcomes have been recorded and they're no longer relevant context.
- **Use `id` for cross-referencing** — Task IDs connect tasks.json entries to task-notes, report.json, and messages.

### Shared Tasks

Teams and roles also have shared task files. Agents can pull work from their individual board, their team's board, or their role's board. This enables coordination without direct communication for simple cases.

---

## report.json — What's Happening Now

report.json is the agent's sticky note on the Pentagon canvas. It's the single most visible indicator of what an agent is doing. Humans glance at it constantly. It updates in real time on the canvas.

### Structure

```json
{
  "summary": "Implementing auth endpoint — handling JWT validation",
  "task": "auth-endpoint",
  "progress": "3/5 routes complete",
  "detail": "Login and register done. Working on token refresh. Password reset next."
}
```

- **`summary`** (required) — One line. What you're doing right now.
- **`task`** — Current task identifier from tasks.json.
- **`progress`** — Brief progress indicator (e.g., "3/5", "step 2 of 4").
- **`detail`** — Context, blockers, decisions, thinking.

### When to Update

| Event | Update? |
|-------|---------|
| Session start | Yes — immediately, before anything else |
| Starting a new task | Yes |
| Progress milestone | Yes |
| Hitting a blocker | Yes |
| Changing strategy | Yes |
| Task completion | Yes |
| Session end | Yes — final summary with ✅ prefix |

### The Critical Distinction

**report.json is for humans, not agents.** Other agents cannot read your report.json. It's visible on the canvas, nowhere else. If a decision needs to reach another agent, send a message — don't just write it in your report.

This is the #1 cause of sync failures in multi-agent teams (see [Chapter 2](02-communication.md)).

### Emoji Conventions

- **✅** — Done. `"✅ Auth module complete — all tests passing"`
- **👋** — Needs human attention. `"👋 Need help — blocked on missing API credentials"`

---

## Organizational Awareness

Pentagon gives every agent awareness of its place in the organization through two auto-managed files.

### ORGANIZATION.md

A human-readable file describing the agent's teams, teammates, map, and role peers. Pentagon updates this automatically as you move agents around — you never edit it directly, but agents read it to know who they're working with.

```markdown
# Organization

## You
- Name: APIManager <UUID>
- Position: grid (5, 3)

## Map: backend (3 agents)
- Path: /Users/you/project
- Location: local

### Other agents in this map
- AuthWorker <UUID> — active
- DataWorker <UUID> — idle
```

### directory.json

A machine-readable contact list of agents visible to the current agent. Each entry is a contact card:

```json
[
  {
    "id": "A1B2C3D4-...",
    "name": "AuthWorker",
    "status": "active",
    "teams": [{"id": "...", "name": "Backend"}],
    "workspace": "backend",
    "source": "project-root",
    "inbox": "/Users/.../.pentagon/agents/A1B2C3D4-.../inbox"
  }
]
```

Agents use directory.json to look up recipients when sending messages. On v1.3+, channels and MCP tools (`send_message`, `find_conversation`) are the primary communication method, but the `inbox` field remains in the contact card for legacy compatibility.

---

## Automatic Files

These files are managed by Pentagon. You don't edit them, but understanding what they do helps with debugging and auditing.

### status.json — Process State

Pentagon monitors agent processes and writes status signals to this file. The StatusEngine reads these signals and determines the agent's display state on the canvas:

| State | Ring Color | Meaning |
|-------|-----------|---------|
| `idle` | Green | Ready for input |
| `active` | Pulsing yellow | Working |
| `waitingInput` | Green | Finished turn, awaiting prompt |
| `dormant` | Gray | User-paused |
| `error` | Red | Something went wrong |

### agent-card.json — Agent Metadata

Stores the agent's name, description, and registered skills. Pentagon manages this — it's used for discovery and canvas display.

### delivery-receipts.jsonl — Message Tracking

Logs sent/delivered status for every message the agent sends. Useful for auditing communication flow (see [Chapter 2](02-communication.md)).

### journal.jsonl — Activity Log

A detailed log of agent activity. Pentagon writes this continuously. Useful for post-session debugging and auditing.

### Session History

Conversation transcripts are automatically preserved at `~/.pentagon/sessions/{uuid}/`. Each session appends to a continuous file, giving every agent a complete lifetime record. You don't manage this — it happens automatically. Session history is reliable — `/clear` preserves session continuity (v1.2.6+), and cross-provider session sanitation prevents corruption (v1.2.7+). If you still notice gaps, updating to the latest Pentagon version usually resolves them.

---

## Key Takeaways

When you spawn an agent, think of the identity stack as a pre-flight checklist:

1. **SOUL.md** — Does the agent know who it is and how to behave?
2. **PURPOSE.md** — Does the agent know what it's here to accomplish?
3. **MEMORY.md** — Does the agent have the context it needs?
4. **tasks.json** — Does the agent know what to work on?
5. **report.json** — Will the agent update its status from moment one?

An agent with all five files properly configured is resilient. It can be restarted, respawned, or handed off to a different human and it picks up right where it left off.

An agent without them is fragile. It depends on conversational context that evaporates when the session ends.

**Invest in identity files upfront. The payoff compounds across every session.**

---

Next: [Chapter 2 — Communication](02-communication.md) — The make-or-break of multi-agent operations.
