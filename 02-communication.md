# Communication — The Make-or-Break of Multi-Agent Operations

> *The #1 operational failure in multi-agent teams isn't bad code or wrong architecture — it's agents that think they communicated when they didn't.*

When you run a single agent, communication is simple: you type, it responds. When you run 5, 8, or 11 agents, communication becomes the critical infrastructure that holds everything together. Get it right and your team hums. Get it wrong and agents sit idle, duplicate work, or build on stale assumptions.

This chapter covers the communication patterns that separate functional multi-agent teams from chaotic ones, and the machinery that makes agent-to-agent messaging work.

---

## The #1 Rule: report.json Is Not Communication

This is the single most important operational lesson in this entire guide.

**report.json is a sticky note on the Pentagon canvas.** Humans see it. Other agents do NOT. It's a one-way broadcast to the UI, not a communication channel.

Agents routinely make this mistake: they write a decision or status update to report.json and assume everyone knows. In practice:

1. Agent A decides "GO — start the build pipeline"
2. Agent A writes this to report.json
3. Agent A moves on, assuming Agent B saw the decision
4. Agent B is still waiting for the decision — it can't read Agent A's report
5. Both agents are now out of sync, and neither knows it

**The fix is one rule, enforced in every agent's SOUL.md:**

```markdown
## Communication Rule
Every decision that affects another agent MUST be sent via
Pentagon's messaging tools. report.json is for canvas visibility only —
other agents cannot read it.
```

This rule alone eliminates the majority of sync failures in multi-agent teams.

### When to Message vs. When to Report

| Action | report.json | Message |
|--------|:-----------:|:-------:|
| Status update for human | ✅ | |
| Decision affecting another agent | ✅ | ✅ |
| Task assignment | | ✅ |
| Progress milestone | ✅ | Only if someone is waiting |
| Blocker or help needed | ✅ (with 👋) | ✅ to supervisor |
| Task completion | ✅ | ✅ to whoever assigned it |

**Rule of thumb:** If removing the information would cause another agent to make a wrong decision or sit idle, it must be a message.

---

## How Agents Communicate

Pentagon v1.3 provides two communication systems. Both are active on v1.3.4 stable.

### Channels and MCP Tools (v1.3+, Primary)

Channels are slack-like conversation spaces. Pentagon provides MCP tools for channel-based communication:

| MCP Tool | Purpose |
|----------|---------|
| `send_message` | Send a message to a conversation/channel |
| `find_conversation` | Find or create a conversation with specific participants |
| `read_messages` | Read messages from a conversation |

Agents use these tools directly — no skill invocation required. Pentagon provides the conversation ID at spawn time, and agents can discover or create conversations via `find_conversation`.

```markdown
## Communication Rule (v1.3+ SOUL.md)
Every decision that affects another agent MUST be sent via
Pentagon's MCP messaging tools (send_message).
report.json is for canvas visibility only.
```

**Message queueing (v1.3.0-beta.21+):** Multiple messages sent to the same conversation queue and dequeue in order automatically. Manager agents that fan out tasks to multiple workers can send all assignments without waiting for acknowledgment between sends.

### Inbox-Based Messaging (v1.2 Legacy, Still Active)

The inbox system from v1.2 remains operational on v1.3.4 stable. Each agent has an inbox directory at `~/.pentagon/agents/{uuid}/inbox/` where JSON message files are written and detected by Pentagon.

The `/send-pentagon-message` skill writes to inboxes using the Write tool. Pentagon watches inbox directories for new files (within ~1 second), then injects the message into the recipient's live conversation.

**Both systems coexist.** Channels via MCP tools are the recommended path forward. The inbox system continues to function and inbox directories remain populated on disk.

---

## Message Delivery Architecture

Understanding how messages travel helps you debug when things break. The architecture differs between the two systems:

### Channel-Based Flow (v1.3+)

```
Agent A → send_message (MCP) → Pentagon Server → Channel → Agent B
```

Pentagon handles routing, queueing, and delivery. Messages are sequenced automatically. The agent sees messages appear in the conversation without needing to read files from disk.

### Inbox-Based Flow (v1.2 Legacy)

```
┌─────────┐    ┌─────────┐    ┌──────────┐    ┌───────────┐    ┌──────────┐
│  Send    │ → │  Write   │ → │  Detect  │ → │  Inject   │ → │  Verify  │
│  (agent) │    │  (disk)  │    │ (Penta.) │    │ (Penta.)  │    │ (Penta.) │
└─────────┘    └─────────┘    └──────────┘    └───────────┘    └──────────┘
```

1. **Send** — Agent composes JSON and writes it to the recipient's inbox via the Write tool
2. **Write** — File lands at `~/.pentagon/agents/{recipient-uuid}/inbox/{message-id}.json`
3. **Detect** — Pentagon's filesystem watcher picks up the new file (within ~1 second)
4. **Inject** — Pentagon injects the message into the recipient's live Claude Code conversation via TUI stdin
5. **Verify** — Pentagon confirms injection succeeded and retries if it didn't (v1.2.8+, 30s retry window)

Each stage can fail independently. When debugging delivery issues, walk the stages in order.

### Injection: The Fragile Stage

Stage 4 (injection) historically caused the most bugs. Pentagon injects messages by programmatically submitting text to the Claude Code process's stdin — essentially "typing" the message notification into the agent's terminal. This creates edge cases:

- **Context window pressure** — Each injected message consumes context
- **Injection timing** — If the agent is mid-tool-call, the message waits
- **Duplicate injection** — A bug where the injection system re-injects the same message repeatedly. The startup variant was fixed in v1.2.8. The mid-session variant was addressed in subsequent releases.

Channel-based communication (v1.3+) bypasses the TUI injection path entirely, which eliminates this class of bugs.

---

## Inbox Message Anatomy (v1.2 Legacy)

Every inbox message is a JSON file in the recipient's inbox directory:

```json
{
  "id": "unique-message-uuid",
  "from": {
    "id": "sender-agent-uuid",
    "name": "SenderName",
    "status": "active",
    "teams": [],
    "inbox": "/path/to/sender/inbox"
  },
  "to": "recipient-agent-uuid",
  "content": "Your message text here",
  "ts": "2026-02-27T19:38:04Z",
  "conversationId": "thread-uuid",
  "replyTo": "original-message-uuid-or-null"
}
```

Key fields:

- **`conversationId`** — Groups messages into threads. Generate a fresh UUID for new conversations. Reuse the same ID when replying.
- **`replyTo`** — The `id` of the message you're responding to. `null` for the first message in a thread.
- **`from.inbox`** — The recipient uses this path to reply back.

---

## Delivery Reliability

### Channel Reliability (v1.3+)

Channels handle message sequencing and delivery through Pentagon's server layer. Message queueing (v1.3.0-beta.21+) eliminates the race conditions that plagued inbox-based rapid-fire messaging. Multiple messages to the same conversation queue automatically.

### Inbox Reliability (v1.2 Legacy)

Pentagon guarantees that inbox messages are never lost, even when the recipient is offline:

- **Cold wake safety** — Messages sent to a dormant agent are queued until it wakes
- **Message quarantine** — Messages that can't be delivered cleanly are moved to `inbox-quarantine/` with a `.verdict.json` file explaining why: `unknownAgent`, `replyLimitExceeded`, or `conversationLimitExceeded`
- **Injection verification (v1.2.8+)** — Pentagon verifies that messages are actually injected into the recipient's conversation, with automatic retry on failure
- **Startup flurry prevention (v1.2.8+)** — When an agent wakes with multiple pending messages, Pentagon staggers delivery to prevent overwhelming the agent's context

### Delivery Receipts

Pentagon logs delivery status to `delivery-receipts.jsonl` in the sender's agent directory:

- **sent** — The file was written to the recipient's inbox (Stage 2 complete)
- **delivered** — The message was injected into the recipient's conversation (Stage 4 verified)

Pentagon also maintains a global A2A audit log at `~/.pentagon/a2a-audit.jsonl` for cross-agent message tracking.

---

## Map Boundaries and Messaging

Maps are the highest level of separation in Pentagon. **Agents cannot and should not communicate across maps.** This is by design — maps are hard boundaries, not soft groupings.

If you find yourself wanting agents on different maps to talk to each other, that's a signal your agents belong on the same map. Maps should separate entirely independent concerns (different projects, different clients, different contexts). Agents that need to coordinate belong together.

Pre-v1.2.12, "workspaces" existed where maps are. Some users ran cross-workspace relay agents. That pattern is no longer supported. Maps solidify the boundary.

**Migration path:**
- Move agents that need to communicate onto the same map (you can cut/copy and paste agents and teams across maps)
- If you need separation *within* a map, use teams — they provide spatial grouping without communication barriers

---

## Quarantine (Inbox System)

If Pentagon can't cleanly inject an inbox message, it moves the file to `inbox-quarantine/` and creates a `.verdict.json`:

| Verdict | Meaning |
|---------|---------|
| `unknownAgent` | Sender UUID not recognized (common after a respawn — stale UUID) |
| `replyLimitExceeded` | Conversation thread has too many replies |
| `conversationLimitExceeded` | Agent has too many active conversations |

Quarantined messages are recoverable. Check `inbox-quarantine/` when messages seem to disappear.

---

## Verify Delivery — Don't Trust, Verify

Assume messages didn't arrive until you prove they did. This sounds paranoid. It's not.

A common failure: an agent creates its internal task board (planning), then goes idle before executing the actual message sends. Its tasks.json shows items dispatched but zero messages exist in any recipient's conversation.

### Verification checklist (after sending critical messages):

**For channel-based messages (v1.3+):**
1. **Check conversation** — Use `read_messages` to verify the message appears in the conversation
2. **Check recipient's report** — After a reasonable interval, see if they acknowledged the work

**For inbox-based messages (v1.2 legacy):**
1. **Check delivery receipts** — Read `delivery-receipts.jsonl` for sent/delivered status
2. **Check your outbox** — Pentagon copies sent messages to `inbox-sent/`. Verify the message appears there.
3. **List the recipient's inbox** — `ls -lt {recipient-inbox}/` — confirm a new `.json` file appeared
4. **Read the message** — Verify content is correct and complete
5. **Check recipient's report** — After a reasonable interval, see if they acknowledged the work

For auditor agents running periodic checks, cross-reference dispatch claims against actual message delivery. If a manager claims to have sent 5 task assignments, there should be 5 messages visible across the relevant conversations or worker inboxes.

---

## Conversation Threading

Threads keep multi-agent conversations traceable. Without them, auditing a complex exchange across 6 agents is archaeology.

### Channel threading (v1.3+)

Channels handle threading through conversation IDs. Each channel is a conversation — participants send messages to it via `send_message` with the conversation ID. Pentagon maintains ordering and history.

### Inbox threading (v1.2 legacy)

Use `conversationId` and `replyTo` fields in the message JSON:

```json
{
  "conversationId": "fresh-uuid",
  "replyTo": null,
  "content": "Task: implement the authentication module..."
}
```

Replying:
```json
{
  "conversationId": "same-uuid-from-original",
  "replyTo": "id-of-message-responding-to",
  "content": "Auth module complete. Files modified: ..."
}
```

### Thread discipline

- **One thread per topic.** Don't mix task assignments with status updates.
- **Include context in replies.** The recipient may have been dormant and lost conversational context. Don't reply "Done" — reply "Auth module complete. Files: `src/auth.ts`, `src/middleware/jwt.ts`. All tests passing. No blockers."

---

## Message Patterns by Role

### Manager → Worker (Task Dispatch)

```
TASK: [clear title]
PRIORITY: [critical/high/medium/low]
DETAILS: [what to build, acceptance criteria]
REPORT BACK: [what the worker should send when done]
```

Be explicit about what "done" looks like. Workers completing tasks but not reporting back is a recurring problem.

### Worker → Manager (Completion Report)

```
COMPLETED: [task title]
FILES: [created/modified files]
STATUS: [tests passing, any caveats]
NEXT: [ready for more work / blocked on X]
```

### Manager → Supervisor (Status Roll-Up)

```
STATUS UPDATE — [area of responsibility]
COMPLETED: [list]
IN PROGRESS: [list]
BLOCKED: [list with reasons]
DECISION NEEDED: [if applicable]
```

---

## Shared Coordination Files

Pentagon provides an alternative to direct messaging for simple coordination: shared files at the team and role level.

| File | Level | Purpose |
|------|-------|---------|
| **Shared tasks** | Team / Role | Shared task list |
| **MEMORY.md** | Team / Role | Shared knowledge base |
| **DISCUSSION.md** | Team | Threaded notes for coordination |

These are useful for lightweight, asynchronous coordination — sharing context, splitting a task list, or leaving notes. For decisions that require acknowledgment or time-sensitive coordination, use direct messages.

---

## Debugging A2A Issues

When a message doesn't arrive, the debugging approach depends on the communication system:

### Channel-based debugging (v1.3+)

1. **Did the agent call `send_message`?** Check the agent's terminal or session transcript
2. **Is the conversation ID correct?** Use `find_conversation` to verify
3. **Are both agents on the same map?** Maps are hard boundaries
4. **Is the recipient agent active?** Check the canvas

### Inbox-based debugging (v1.2 legacy)

Walk the five stages:

| Stage | Check | How |
|-------|-------|-----|
| 1. Send | Did the agent call the messaging skill? | Check terminal |
| 2. Write | Is the file in the recipient's inbox? | `ls ~/.pentagon/agents/{uuid}/inbox/` |
| 3. Detect | Is Pentagon watching? | Check that the agent is on the canvas |
| 4. Inject | Did the recipient see a notification? | Check terminal for "New message from..." |
| 5. Verify | Does delivery-receipts.jsonl show "delivered"? | Read sender's `delivery-receipts.jsonl` |

### Common Failures

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Message never sent | Agent planned but didn't execute | Check terminal; re-send |
| Inbox file exists but agent didn't react | Injection failed | Wait for retry or restart agent |
| File in quarantine | Rejected by Pentagon | Read `.verdict.json` |
| "Sent" receipt but no "delivered" | Injection failed, retry exhausted | Restart recipient |
| Agent acknowledges same message 10+ times | Duplicate injection bug | Run `/clear` in recipient's terminal |
| Cross-map message | Hard boundary | Move agents to same map |

For deeper troubleshooting, see [Chapter 10 — Troubleshooting](10-troubleshooting.md).

---

## Anti-Patterns

### 1. The Silent Completer
**Problem:** Agent finishes a task, marks it done in tasks.json, but never tells the assigning agent.
**Fix:** Add to SOUL.md: "When you complete an assigned task, send a completion message to whoever assigned it."

### 2. The Report-Only Communicator
**Problem:** Agent writes everything to report.json and nothing to conversations.
**Fix:** The #1 Rule. Enforce it in every agent's SOUL.md.

### 3. The Fire-and-Forget Dispatcher
**Problem:** Manager updates its own task board, assumes messages were sent, goes dormant.
**Fix:** Verify delivery. Check conversations or recipient inboxes after dispatching. Don't stop until messages are confirmed.

### 4. The Context-Free Reply
**Problem:** Agent replies "Done" or "OK" with no details.
**Fix:** Require structured replies. Every response should include what was done, where, and what's next.

### 5. The Broadcast Spammer
**Problem:** Agent sends every status update to every other agent, flooding conversations.
**Fix:** Messages go to specific recipients who need to act on them. Use report.json for general visibility.

---

## Known Limitations

### Single-Machine Boundary
A2A communication is local — no network transport. All agents must run on the same machine. There's no built-in way to send a message to an agent on a different computer.

### No Guaranteed Ordering (Inbox System)
Inbox messages are delivered in approximately the order they're detected, but there's no strict ordering guarantee. For order-dependent workflows, use `replyTo` chains for explicit sequencing. Channel-based messaging (v1.3+) handles sequencing through message queueing.

---

## Version History

| Version | Communication Change |
|---------|---------------------|
| Pre-v1.2.7 | Aggressive session sanitation could corrupt cross-provider sessions |
| v1.2.7 | Less aggressive sanitation; cross-provider support stabilized |
| v1.2.8 | Startup flurry prevention; injection verification with 30s retry |
| v1.2.9 | `/send-pentagon-message` switched from mv to Write tool |
| v1.2.12 | Maps replace workspaces; cross-map messaging removed |
| v1.2.15 | A2A reliability fixes; session management improvements |
| v1.3.0-beta.19 | Auth token refresh fix; spawn conversation ID routing fix |
| v1.3.0-beta.21 | Message queueing — multi-message auto-sequencing |
| v1.3.0-beta.25 | Channel-based A2A communication; communication stability improvements |
| v1.3.4 | Stable graduation — channels, MCP tools, message queueing all ship as stable |

---

## Key Takeaways

1. **report.json ≠ communication.** It's a canvas sticky note. Other agents can't read it.
2. **Use MCP tools on v1.3+.** `send_message`, `find_conversation`, and `read_messages` are the primary communication path. The inbox system still works but channels are the recommended approach.
3. **Message every decision that affects another agent.** Non-negotiable.
4. **Message queueing eliminates send-wait-send.** On v1.3+, managers can fan out tasks to multiple workers without waiting between sends.
5. **Verify delivery.** Messages that didn't land don't exist.
6. **Be explicit.** Include enough context for the recipient to act without follow-up questions.
7. **Maps are hard boundaries.** Agents on different maps cannot communicate. Put coordinating agents on the same map.
8. **Bake the rules into SOUL.md.** Agents follow what's in their identity. Put communication rules there.

---

Next: [Chapter 3 — Hierarchy Design](03-hierarchy-design.md) — How to structure teams that scale without chaos.
