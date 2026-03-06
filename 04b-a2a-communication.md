# A2A Communication — How Agent-to-Agent Messaging Works

> *Pentagon agents talk to each other by writing JSON files to inboxes. It sounds simple. Under the hood, there's an injection system, a verification layer, a quarantine mechanism, and a set of hard-won reliability fixes that make it all work.*

Chapter 2 covers communication *patterns* — when to message, threading, anti-patterns. This chapter covers the *machinery*: how messages actually travel from one agent's conversation to another's, what happens at each step, and how to debug when something breaks.

---

## Architecture Overview

A2A messaging in Pentagon has five stages:

```
┌─────────┐    ┌─────────┐    ┌──────────┐    ┌───────────┐    ┌──────────┐
│  Send    │ → │  Write   │ → │  Detect  │ → │  Inject   │ → │  Verify  │
│  (agent) │    │  (disk)  │    │ (Penta.) │    │ (Penta.)  │    │ (Penta.) │
└─────────┘    └─────────┘    └──────────┘    └───────────┘    └──────────┘
```

1. **Send** — The sending agent composes a JSON message and writes it to the recipient's inbox
2. **Write** — The file lands on disk at `~/.pentagon/agents/{recipient-uuid}/inbox/{message-id}.json`
3. **Detect** — Pentagon watches inbox directories for new files (within ~1 second)
4. **Inject** — Pentagon injects the message into the recipient agent's live conversation
5. **Verify** — Pentagon confirms the injection succeeded and retries if it didn't

Each stage can fail independently. Understanding where a failure happened tells you exactly how to fix it.

---

## Stage 1: Sending

The sending agent uses the `/send-pentagon-message` skill (or writes directly with the Write tool). The skill handles:

- Looking up the recipient in `directory.json`
- Building the message JSON with proper `from`, `to`, `conversationId`, and `replyTo` fields
- Writing the file to the recipient's inbox path

### The Write Tool Requirement

Pentagon's scope guard blocks all Bash file operations (`mv`, `cp`, `>`, `tee`) targeting other agents' directories. The **Write tool is the only allowed method** for delivering messages.

This isn't just a recommendation — it's enforced. If an agent tries `echo '...' > /path/to/inbox/msg.json`, the scope guard hook kills the command. This enforcement also validates `from.id` against the sender's actual agent UUID, preventing message spoofing.

Earlier versions (pre-v1.2.9) used a tmp-file-then-rename approach:
```bash
# Old approach (no longer used)
cp /tmp/msg.json "$INBOX/.$UUID.tmp"
mv "$INBOX/.$UUID.tmp" "$INBOX/$UUID.json"
```

v1.2.9 switched to direct Write tool delivery — simpler, saves tokens, and aligns with the scope guard.

---

## Stage 2: Writing to Disk

The message file is created atomically by the Write tool at:

```
~/.pentagon/agents/{recipient-uuid}/inbox/{message-id}.json
```

This is a real file on the local filesystem. There's no message broker, no queue service, no network hop. Agent-to-agent communication is local file I/O — which is why it's fast but also why it's bounded to a single machine.

Since each message is a separate file with a unique UUID, simultaneous delivery to the same inbox is safe — there are no write conflicts.

### Message Format

```json
{
  "id": "unique-message-uuid",
  "from": {
    "id": "sender-agent-uuid",
    "name": "SenderName",
    "status": "active",
    "teams": [{"id": "...", "name": "TeamName"}],
    "inbox": "/Users/you/.pentagon/agents/{sender-uuid}/inbox"
  },
  "to": "recipient-agent-uuid",
  "content": "Your message text here",
  "ts": "2026-03-05T19:38:04Z",
  "conversationId": "thread-uuid",
  "replyTo": "original-message-uuid-or-null"
}
```

The `from` block is a full contact card — it gives the recipient everything it needs to reply without looking up the sender separately.

---

## Stage 3: Detection

Pentagon runs a filesystem watcher on every active agent's inbox directory. When a new `.json` file appears, Pentagon picks it up within ~1 second.

Detection is passive — the recipient agent doesn't poll its inbox. Pentagon handles this entirely outside the agent's conversation. The agent has no idea a message is coming until Pentagon injects it.

### What Can Go Wrong Here

- **Agent is on a different map** — Maps are hard boundaries (v1.2.12+). Pentagon won't inject messages across maps. If a file is written to an inbox via direct filesystem manipulation (bypassing Pentagon), it sits there undetected — Pentagon only watches inboxes within the active map.
- **Agent is dormant/dead** — The file is safely queued. Pentagon delivers it when the agent next wakes up.
- **Inbox directory doesn't exist** — The Write tool will create it, but if the agent was deleted or the UUID is wrong, the message goes nowhere.

---

## Stage 4: Injection

This is the critical stage — and the one that historically caused the most bugs.

Pentagon takes the detected message and **injects it into the recipient agent's live Claude Code conversation**. The agent sees a notification like:

```
New message from atlas. Read: /path/to/inbox/{messageId}.json
```

The agent then uses its Read tool to open the file and process the content.

### How Injection Works

Pentagon programmatically submits the notification text to the Claude Code process's stdin — effectively "typing" the message notification into the agent's terminal. The agent sees it as a system-level notification that it must respond to.

This is the current architectural constraint: Pentagon piggybacks on Claude Code's TUI stdin to inject messages. It works, but it's also the source of several edge cases:

- **Context window pressure** — Each injected message consumes context. Under heavy message volume, the agent's context fills faster.
- **Injection timing** — If the agent is mid-tool-call when the injection arrives, the message may not be processed until the current operation completes.
- **Duplicate injection** — A bug (first reported v1.2.8) where the injection system re-injects the same message repeatedly. The startup variant was fixed in v1.2.8; the mid-session variant has not been resolved as of v1.2.15 (see [Troubleshooting](10-troubleshooting.md#duplicate-message-injection-message-flood)).

### Quarantine

If Pentagon can't cleanly inject a message, it moves the file to `inbox-quarantine/` and creates a `.verdict.json` file explaining why:

| Verdict | Meaning |
|---------|---------|
| `unknownAgent` | The sender's UUID isn't recognized (common after a respawn — the old UUID is stale) |
| `replyLimitExceeded` | The conversation thread has too many replies |
| `conversationLimitExceeded` | The agent has too many active conversations |

Quarantined messages aren't lost — they're recoverable. Check `inbox-quarantine/` when messages seem to disappear.

---

## Stage 5: Verification

Added in v1.2.8, injection verification closes the "delivered but never read" gap.

After injecting a message, Pentagon programmatically verifies that the injection actually took — that the agent's conversation now contains the message notification. If verification fails, Pentagon retries.

### Retry Behavior

- **Retry window:** 30 seconds (conservative, to accommodate slower machines)
- **Startup staggering:** When an agent wakes with multiple pending messages, Pentagon delivers them one at a time with spacing, preventing a "message flurry" that overwhelms the agent's first few seconds of context
- **Automatic:** No user intervention needed. Pentagon handles retry silently.

### What Verification Catches

Before v1.2.8, this failure mode was common:
1. Agent A sends a message to Agent B
2. The file lands in Agent B's inbox (Stage 2 succeeds)
3. Pentagon tries to inject it (Stage 4)
4. The injection silently fails (Agent B was mid-operation, or the TUI didn't accept it)
5. Agent B never sees the message, but Agent A's delivery receipt says "sent"

Now, verification catches step 4 failures and retries until the injection succeeds or the retry window expires.

---

## Delivery Receipts

Pentagon logs delivery status to `delivery-receipts.jsonl` in the sender's agent directory:

- **sent** — The file was written to the recipient's inbox (Stage 2 complete)
- **delivered** — The message was injected into the recipient's conversation (Stage 4 verified)

Auditor agents can read these receipts to verify that messages actually landed end-to-end. If a receipt shows "sent" but never "delivered," the injection failed — check if the recipient was dormant or if the message was quarantined.

Pentagon also maintains a global A2A audit log at `~/.pentagon/a2a-audit.jsonl` for cross-agent message tracking. This gives you a single-file view of all A2A activity across your map — useful for fleet-wide health checks without reading every agent's individual receipts.

---

## Map Boundaries

As of v1.2.12, maps are the hard boundary for A2A communication. Agents on the same map can message each other freely. Agents on different maps **cannot communicate at all**.

This is intentional. Maps are the highest level of separation in Pentagon — they're the foundation for upcoming multiplayer support, where you won't want everyone you work with to see everyone else you work with.

If you need agents to coordinate, they must be on the same map. Use teams for separation within a map.

---

## Practical Setup

### Minimal A2A Between Two Agents

**1. Confirm both agents are on the same map.** Check each agent's ORGANIZATION.md.

**2. Confirm mutual visibility.** Each agent should see the other in its `directory.json`. If not, they may be on different maps or one may not be spawned yet.

**3. Send a test message:**
```
/send-pentagon-message agent-name Hello — can you read this?
```

**4. Verify delivery:**
- Check your `delivery-receipts.jsonl` for "delivered" status
- Check the recipient's inbox: `ls ~/.pentagon/agents/{recipient-uuid}/inbox/`
- Wait for the recipient to acknowledge

### Setting Up a Manager-Worker Message Flow

**1. Add hierarchy to both agents' SOUL.md:**

Manager:
```markdown
## Hierarchy
- Workers: worker-a <UUID-A>, worker-b <UUID-B>

## Communication Rule
Every task assignment MUST be sent via /send-pentagon-message.
Verify delivery after sending. Don't stop until confirmed.
```

Worker:
```markdown
## Hierarchy
- Reports to: manager <UUID-M>

## Communication Rule
When I complete a task, I MUST send a completion report to my manager
via /send-pentagon-message. Include: what was done, files changed, any notes.
```

**2. The manager sends a task:**
```
/send-pentagon-message worker-a TASK: Implement the auth module. PRIORITY: high. REPORT BACK: files changed, tests passing.
```

**3. The worker completes and reports:**
```
/send-pentagon-message manager COMPLETED: Auth module. FILES: src/auth.ts, src/middleware/jwt.ts. STATUS: all tests passing. NEXT: ready for more work.
```

**4. Audit the flow:** An auditor agent (or the human) can verify by cross-referencing:
- Manager's `delivery-receipts.jsonl` → messages were sent
- Worker's `inbox/` → messages were received
- Worker's `delivery-receipts.jsonl` → completion report was sent back

### Scaling to Many Agents

For teams of 5+ agents:

- **Use threading.** Every message gets a `conversationId`. New topics get new threads. This keeps audit trails traceable.
- **Stagger heavy dispatches.** If a manager sends tasks to 6 workers simultaneously, the API load spikes. Add brief delays between dispatches or let Pentagon's staggering handle startup scenarios.
- **Monitor quarantine.** As message volume grows, so does the chance of hitting reply or conversation limits. Periodically check `inbox-quarantine/` across your agents.
- **Structured message formats.** Consistent formats (see [Chapter 2 — Message Patterns by Role](02-communication.md#message-patterns-by-role)) make auditing possible at scale.
- **Monitor fleet A2A health.** Have your auditor scan `~/.pentagon/a2a-audit.jsonl` or `delivery-receipts.jsonl` across agents and flag any sent-but-not-delivered entries older than 60 seconds.

---

## Debugging A2A Issues

When a message doesn't arrive, walk the five stages:

| Stage | Check | How |
|-------|-------|-----|
| 1. Send | Did the agent actually call /send-pentagon-message? | Check the agent's terminal or session transcript |
| 2. Write | Is the file in the recipient's inbox? | `ls ~/.pentagon/agents/{uuid}/inbox/` |
| 3. Detect | Is Pentagon running and watching? | Check Pentagon app is open and the agent is on the canvas |
| 4. Inject | Did the recipient's conversation get the notification? | Check recipient's terminal for "New message from..." |
| 5. Verify | Does delivery-receipts.jsonl show "delivered"? | Read the sender's `delivery-receipts.jsonl` |

### Common Failures

| Symptom | Likely Stage | Fix |
|---------|-------------|-----|
| File not in inbox | Stage 1 (never sent) or Stage 2 (wrong UUID) | Check sender's terminal; verify recipient UUID |
| File in inbox but agent didn't react | Stage 4 (injection failed) | Wait for retry, or restart the agent |
| File in quarantine | Stage 4 (rejected) | Read `.verdict.json` for the reason |
| "Sent" receipt but no "delivered" | Stage 4-5 (injection failed, retry exhausted) | Restart recipient agent |
| Agent acknowledges same message 10+ times | Stage 4 bug (duplicate injection) | Run `/clear` in recipient's terminal |
| Message between agents on different maps | Stage 3 (hard boundary) | Move agents to same map |

For deeper troubleshooting, see [Chapter 10 — Troubleshooting](10-troubleshooting.md#messages-not-delivering). For duplicate injection specifically, see [Duplicate Message Injection](10-troubleshooting.md#duplicate-message-injection-message-flood).

---

## Known Limitations

### TUI-Based Injection
Pentagon currently injects messages through Claude Code's terminal interface. This works but creates friction:
- Injection is tied to the TUI's readiness — if the terminal is busy, injection waits
- Each injection consumes context tokens (the notification text)
- Duplicate injection bugs stem from the TUI injection path

Community consensus is that breaking out of the TUI for injection would simplify the architecture significantly.

### Single-Machine Boundary
A2A communication is local file I/O — no network transport. All agents must run on the same machine. There's no built-in way to send a message to an agent on a different computer.

### No Guaranteed Ordering
Messages are delivered in approximately the order they're detected, but there's no strict ordering guarantee. If Agent A sends messages M1 and M2 in rapid succession, the recipient might process M2 before M1 if detection timing varies. For order-dependent workflows, use `replyTo` chains to establish explicit sequencing.

---

## Version History

| Version | A2A Change |
|---------|-----------|
| Pre-v1.2.7 | Aggressive session sanitation could corrupt cross-provider sessions |
| v1.2.7 | Less aggressive sanitation; cross-provider (Claude + Kimi) support stabilized |
| v1.2.8 | Startup flurry prevention; programmatic injection verification with 30s retry |
| v1.2.9 | `/send-pentagon-message` switched from mv to Write tool; scope guard alignment |
| v1.2.12 | Maps replace workspaces; cross-map messaging removed (hard boundary) |
| v1.2.15 | Additional A2A reliability fixes; session management improvements |

---

## Key Takeaways

1. **Five stages: Send → Write → Detect → Inject → Verify.** Know where a failure happened and you know how to fix it.
2. **Write tool only.** Bash file operations to other agents' directories are blocked by the scope guard. Always use the Write tool.
3. **Injection is the fragile stage.** Most A2A bugs live in Stage 4. Pentagon's verification (v1.2.8+) catches most failures automatically.
4. **Maps are hard boundaries.** Agents on different maps can't communicate. Put coordinating agents on the same map.
5. **Messages are never lost.** They're either in the inbox, in quarantine, or delivered. Check all three locations when debugging.
6. **Delivery receipts are your audit trail.** "sent" means disk write succeeded. "delivered" means injection verified. The gap between them is where bugs hide.
7. **The TUI is the bottleneck.** Current injection goes through Claude Code's terminal interface. This constrains throughput and is the source of duplicate injection bugs.

---

Back to [README](README.md) | Previous: [Chapter 4 — Agent Lifecycle](04-lifecycle.md) | Next: [Chapter 5 — The Auditor Pattern](05-auditor-pattern.md)
