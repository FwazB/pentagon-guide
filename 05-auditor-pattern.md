# The Auditor Pattern — Independent Monitoring That Catches What the Hierarchy Misses

> *Every agent sees its own slice. The auditor sees the whole board. That bird's-eye view is irreplaceable.*

Hierarchies have blind spots. A manager knows what it dispatched but not what other managers did. A lead knows the strategy but not the implementation details. Workers know their tasks but not the broader picture. These blind spots become sync gaps, and sync gaps become lost work.

The auditor pattern adds an independent observer outside the chain of command that monitors everything and reports directly to the human.

---

## What Is an Auditor?

An auditor agent:

1. **Operates outside the hierarchy.** Reports only to the human, not to any lead or manager.
2. **Reads everything.** Checks all agent inboxes, reports, and task boards.
3. **Modifies nothing.** Doesn't touch code, task files, or agent configs. Only sends messages and updates its own files.
4. **Bridges gaps.** When it detects a communication failure, it sends the missing messages.

```
┌─────────────────────────────────┐
│           Human                 │
└───────┬─────────────┬───────────┘
        │             │
  ┌─────▼─────┐  ┌────▼─────┐
  │   Lead    │  │ Auditor  │ ← Independent
  │(hierarchy)│  │(observer)│   Reports only to human
  └─────┬─────┘  └──────────┘
        │              ▲
   [managers]          │ reads everything
        │              │ reports gaps
   [workers]───────────┘
```

The auditor's independence is what makes it effective. It can't audit the hierarchy if it's part of the hierarchy.

---

## What the Auditor Does

### 1. Communication Audit

The primary function. Periodically:

- **Scan all agent inboxes** — List files in each `~/.pentagon/agents/{uuid}/inbox/`
- **Read all report.json files** — What does each agent claim to be doing?
- **Cross-reference claims vs. evidence** — If a manager says "dispatched 5 tasks" but worker inboxes are empty, that's a gap

### 2. Idle Detection

- **Check agent status** — Who's active vs. idle vs. dormant?
- **Compare with reports** — If an agent's report says "working on X" but they're idle on the canvas, they went down without finishing
- **Alert the human** — "[Agent] is idle with 3 unread task assignments in inbox"

### 3. Gap Bridging

When the auditor finds a sync gap, it doesn't just report — it fixes:

- **Sends the missing message.** If a lead decided "GO" but didn't message the manager, the auditor sends it.
- **Relays human directives.** When the human types instructions in one terminal, the auditor broadcasts them to relevant agents.
- **Re-notifies idle agents.** If messages were sent to an agent that was idle, the auditor pings them again after restart.

### 4. Priority Monitoring

Catches misaligned priorities: a manager dispatching low-priority polish work while a critical bug fix sits unassigned. The auditor flags this to the human for correction.

---

## Setting Up an Auditor

### SOUL.md

```markdown
## Identity
Communications auditor. Independent observer.

## Role
Monitor all agent communications, detect sync gaps, and bridge
failures. Report ONLY to the human.

## Hierarchy
- Reports to: Human — ONLY
- Does NOT report to any agent in the hierarchy
- Has read access to all agent inboxes and reports

## Responsibilities
1. Periodic comms audits — scan all inboxes and reports
2. Idle agent detection — cross-reference status with reports
3. Gap bridging — send missing messages
4. Human relay — broadcast human directives to relevant agents
5. Priority monitoring — flag misaligned priorities

## Constraints
- NEVER write code
- NEVER modify other agents' files (except sending inbox messages)
- NEVER take orders from agents in the hierarchy
- NEVER make strategic decisions — observe and report only

## Communication Rule
Every decision that affects another agent MUST be sent via
/send-pentagon-message.
```

### tasks.json

```json
[
  { "id": "comms-audit", "title": "Run comms audit cycle", "status": "backlog" },
  { "id": "idle-check", "title": "Check for idle agents with pending work", "status": "backlog" },
  { "id": "delivery-verify", "title": "Verify message delivery for recent dispatches", "status": "backlog" },
  { "id": "gap-report", "title": "Report gaps to human", "status": "backlog" }
]
```

### Heartbeat Configuration

Set the auditor to **Interval** mode (e.g., every 5-10 minutes) so it periodically wakes, runs an audit cycle, and goes dormant. This gives you continuous monitoring without burning resources.

---

## The Audit Loop

### Step 1: Scan Reports

```
For each agent in directory.json:
  Read {agent-dir}/report.json
  Record: agent name, claimed activity, timestamp
```

### Step 2: Check Statuses

```
For each agent:
  If report says "working" but status is "idle" → FLAG
  If report hasn't changed in >30 minutes → FLAG
  If report is empty or stale → FLAG
```

### Step 3: Scan Inboxes

```
For each agent:
  List {agent-inbox}/*.json
  Count unread messages
  Cross-reference: do dispatch claims match actual inbox contents?
```

### Step 4: Bridge Gaps

```
For each gap found:
  If fixable (missing message) → send it
  If needs human decision → report with evidence and recommendation
  If agent is idle with pending work → alert human
```

### Step 5: Report

```json
{
  "summary": "Comms audit complete — 2 gaps found, 1 bridged",
  "detail": "Bridged: Lead→Manager decision message. Flagged: DataWorker idle with 3 pending tasks."
}
```

---

## Common Catches

### The Phantom Dispatch
A manager updates its own task board to show 7 tasks dispatched. Auditor checks worker inboxes — zero messages. The manager planned the work but went dormant before actually sending the messages.

**Fix:** Auditor sends the task messages on the manager's behalf.

### The Decision That Never Landed
A lead writes "GO — start the build" in report.json. The manager is waiting for the go signal. report.json isn't communication — the manager never got the message.

**Fix:** Auditor sends the decision as a proper message to the manager.

### The Idle Worker with a Full Inbox
A worker's session ended after completing one task. Three more assignments arrived in its inbox. No one noticed for hours.

**Fix:** Auditor flags the idle agent to the human, who respawns or resumes it.

### Priority Misalignment
A manager dispatches cosmetic polish tasks while a critical blocker sits unassigned. The lead doesn't know because the manager's status roll-up is overdue.

**Fix:** Auditor reports the misalignment to the human with the specific tasks involved.

---

## Anti-Patterns

### The Auditor That Manages
**Problem:** Auditor starts assigning tasks and directing workers.
**Fix:** The auditor observes and bridges. It doesn't direct. Decisions go to the human.

### The Auditor That Reports to the Lead
**Problem:** Auditor sends findings to the lead agent instead of the human.
**Fix:** Auditor reports to the human only. This ensures independence.

### The Auditor That Doesn't Act
**Problem:** Auditor writes detailed reports but doesn't bridge gaps.
**Fix:** For simple gaps (missing messages), bridge immediately. For complex gaps (priority conflicts), report to the human with a recommendation.

---

## Scaling

**Up to ~15 agents:** One auditor handles everything.

**15+ agents:** Split by concern:
- Comms Auditor — message flow and delivery
- Quality Auditor — code quality, test results
- Priority Auditor — task alignment and bottlenecks

Each reports independently to the human.

---

## When to Use This Pattern

- **Always** for teams of 5+ agents. The coordination complexity makes gaps inevitable.
- **Optional** for 2-4 agents. The human can manually check at this scale.
- **Critical** when you have multi-tier hierarchies. The more levels, the more places messages can get lost.

The auditor adds one agent's worth of cost. It saves multiples of that in prevented sync failures and lost work.

---

## Key Takeaways

1. **Independence is non-negotiable.** The auditor reports to the human only.
2. **Don't just report — bridge.** Detecting a gap is half the job. Sending the missing message is the other half.
3. **Cross-reference everything.** Task boards, inboxes, reports. When they don't match, you've found a gap.
4. **Use interval heartbeat.** Periodic auditing catches problems without burning resources.
5. **Start the auditor with the team.** Don't wait for failures. By then, you've already lost work.

---

Next: [Chapter 6 — Report Discipline](06-report-discipline.md) — Making your canvas actually useful.
