# Communication — The Make-or-Break of Multi-Agent Operations

> *The #1 operational failure in multi-agent teams isn't bad code or wrong architecture — it's agents that think they communicated when they didn't.*

When you run a single agent, communication is simple: you type, it responds. When you run 5, 8, or 11 agents, communication becomes the critical infrastructure that holds everything together. Get it right and your team hums. Get it wrong and agents sit idle, duplicate work, or build on stale assumptions.

This chapter covers the communication patterns that separate functional multi-agent teams from chaotic ones.

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
/send-pentagon-message. report.json is for canvas visibility only —
other agents cannot read it.
```

This rule alone eliminates the majority of sync failures in multi-agent teams.

### When to Message vs. When to Report

| Action | report.json | /send-pentagon-message |
|--------|:-----------:|:---------------------:|
| Status update for human | ✅ | |
| Decision affecting another agent | ✅ | ✅ |
| Task assignment | | ✅ |
| Progress milestone | ✅ | Only if someone is waiting |
| Blocker or help needed | ✅ (with 👋) | ✅ to supervisor |
| Task completion | ✅ | ✅ to whoever assigned it |

**Rule of thumb:** If removing the information would cause another agent to make a wrong decision or sit idle, it must be a message.

---

## Message Anatomy

Every Pentagon message is a JSON file delivered to a recipient's inbox directory (`~/.pentagon/agents/{uuid}/inbox/`).

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
- **`from.inbox`** — The recipient uses this path to reply back to you.

---

## Atomic Message Delivery

Pentagon uses an atomic write protocol to prevent partial reads:

```
Step 1: Write to   {recipient-inbox}/.{messageId}.tmp
Step 2: Rename to  {recipient-inbox}/{messageId}.json
```

The rename is atomic on the filesystem — the recipient either sees the complete message or nothing. This prevents race conditions where an agent reads a half-written file.

**Scope guard enforcement:** Pentagon's scope guard blocks Bash file operations (`mv`, `cp`, redirects) targeting other agent directories. The **Write tool is the only allowed method** for delivering messages to another agent's inbox. The scope guard also validates that `from.id` matches the sender's agent ID — message spoofing is blocked at the hook level.

---

## Delivery Receipts

Pentagon tracks message delivery status automatically. When you send a message via `/send-pentagon-message`, Pentagon logs the delivery in `delivery-receipts.jsonl` — a lightweight audit trail showing:

- **sent** — Message was written to the recipient's inbox
- **delivered** — Recipient agent has received the message

This gives you observability into message flow without manually checking inboxes. Auditor agents can read delivery receipts to verify that messages actually landed.

---

## Inbox Reliability

Pentagon guarantees that messages are never lost, even when the recipient agent is offline:

- **Cold wake safety** — Messages sent to an agent that's dormant or restarting are safely queued. No message loss during cold agent wakes.
- **Safe quarantine** — If a message fails to deliver, it's quarantined (never deleted). Failed messages can be retried or inspected.
- **TTL-based cleanup** — Old messages are cleaned up automatically after their time-to-live expires, preventing inbox bloat.

These guarantees mean you can send messages to any agent regardless of their current state. The messages will be there when the agent wakes up.

---

## Verify Delivery — Don't Trust, Verify

Assume messages didn't arrive until you prove they did. This sounds paranoid. It's not.

A common failure: an agent creates its internal task board (planning), then goes idle before executing the actual `/send-pentagon-message` calls. Its tasks.json shows items dispatched but zero messages exist in any recipient inbox.

### Verification checklist (after sending critical messages):

1. **Check delivery receipts** — Read `delivery-receipts.jsonl` for sent/delivered status
2. **List the recipient's inbox** — `ls -lt {recipient-inbox}/` — confirm a new `.json` file appeared
3. **Read the message** — Verify content is correct and complete
4. **Check recipient's report** — After a reasonable interval, see if they acknowledged the work

For auditor agents running periodic checks, cross-reference dispatch claims against actual inbox contents. If a manager claims to have sent 5 task assignments, there should be 5 new message files across worker inboxes.

---

## Conversation Threading

Threads keep multi-agent conversations traceable. Without them, auditing a complex exchange across 6 agents is archaeology.

### Starting a new thread

```json
{
  "conversationId": "fresh-uuid",
  "replyTo": null,
  "content": "Task: implement the authentication module..."
}
```

### Replying to a thread

```json
{
  "conversationId": "same-uuid-from-original",
  "replyTo": "id-of-message-responding-to",
  "content": "Auth module complete. Files modified: ..."
}
```

### Thread discipline

- **One thread per topic.** Don't mix task assignments with status updates.
- **Always set `replyTo`** when responding. It creates a clear chain of causality.
- **Include context in replies.** The recipient may have been dormant and lost their conversational context. Don't reply "Done" — reply "Auth module complete. Files: `src/auth.ts`, `src/middleware/jwt.ts`. All tests passing. No blockers."

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

Pentagon provides an alternative to direct messaging for simple coordination: shared files at the pod and team level.

| File | Level | Purpose |
|------|-------|---------|
| **Shared tasks** | Pod / Team | Shared task list |
| **MEMORY.md** | Pod / Team | Shared knowledge base |
| **DISCUSSION.md** | Pod | Threaded notes for coordination |

These are useful for lightweight, asynchronous coordination — sharing context, splitting a task list, or leaving notes. For decisions that require acknowledgment or time-sensitive coordination, use direct messages.

---

## Anti-Patterns

### 1. The Silent Completer
**Problem:** Agent finishes a task, marks it done in tasks.json, but never tells the assigning agent.
**Fix:** Add to SOUL.md: "When you complete an assigned task, send a completion message to whoever assigned it."

### 2. The Report-Only Communicator
**Problem:** Agent writes everything to report.json and nothing to inboxes.
**Fix:** The #1 Rule. Enforce it in every agent's SOUL.md.

### 3. The Fire-and-Forget Dispatcher
**Problem:** Manager updates its own task board, assumes messages were sent, goes dormant.
**Fix:** Verify delivery. List recipient inboxes after dispatching. Don't stop until messages are confirmed.

### 4. The Context-Free Reply
**Problem:** Agent replies "Done" or "OK" with no details.
**Fix:** Require structured replies. Every response should include what was done, where, and what's next.

### 5. The Broadcast Spammer
**Problem:** Agent sends every status update to every other agent, flooding inboxes.
**Fix:** Messages go to specific recipients who need to act on them. Use report.json for general visibility.

---

## Key Takeaways

1. **report.json ≠ communication.** It's a canvas sticky note. Other agents can't read it.
2. **Message every decision that affects another agent.** Non-negotiable.
3. **Delivery receipts give you observability.** Check `delivery-receipts.jsonl` for sent/delivered status.
4. **Messages are never lost.** Inbox reliability guarantees safe delivery even to cold/dormant agents.
5. **Verify delivery.** Check delivery receipts and recipient inboxes. Messages that didn't land don't exist.
6. **Thread everything.** `conversationId` + `replyTo` make auditing possible.
7. **Be explicit.** Include enough context for the recipient to act without follow-up questions.
8. **Bake the rules into SOUL.md.** Agents follow what's in their identity. Put communication rules there.

---

Next: [Chapter 3 — Hierarchy Design](03-hierarchy-design.md) — How to structure teams that scale without chaos.
