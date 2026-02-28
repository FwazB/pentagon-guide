# Common Pitfalls — The Failures That Keep Happening

> *Every pitfall in this chapter has happened in production. Not in theory, not in testing — in actual multi-agent operations. Learn from them.*

Running multi-agent teams with Pentagon surfaces a predictable set of failure modes. They're predictable because the underlying causes are structural — they emerge from how agents process information, manage state, and communicate. Once you know the patterns, you can prevent them.

---

## 1. The Report-as-Communication Trap

**What happens:** An agent writes a critical decision to report.json, assuming other agents will see it. They don't. report.json is only visible on the Pentagon canvas — other agents have no access to it.

**Impact:** Agents work on stale assumptions. A manager waits for a decision that was already made. A worker continues building something that was cancelled.

**Root cause:** Agents (and humans) confuse visibility with communication. The canvas shows the report; the inbox delivers the message.

**Prevention:**
```markdown
# Add to every agent's SOUL.md:
Every decision that affects another agent MUST be sent via
/send-pentagon-message. report.json is for human visibility only.
```

**Detection:** The auditor pattern (Chapter 5) catches this by cross-referencing reports with inbox contents.

---

## 2. The Phantom Dispatch

**What happens:** A manager creates a detailed task list in tasks.json — planning each assignment, writing descriptions, setting priorities. Then it goes dormant or exhausts its context before actually sending the task messages via `/send-pentagon-message`.

**Impact:** The manager's task board looks active. Worker inboxes are empty. Zero work is happening, but everything looks fine on paper.

**Root cause:** Planning and execution are separate steps. The manager completed planning (updating its own files) but never executed (sending messages).

**Prevention:**
- Add to manager SOUL.md: "After creating task assignments, verify delivery by listing each worker's inbox."
- Have auditors periodically cross-reference manager task boards with worker inbox contents.

**Detection:** List worker inboxes. If a manager claims N tasks dispatched but inboxes have 0 new messages, that's a phantom dispatch.

---

## 3. The Idle Agent Problem

**What happens:** An agent's session ends — timeout, crash, context exhaustion, or the human closes the terminal. Pentagon marks it as idle/dormant on the canvas. Other agents keep sending messages to its inbox. Nobody reads them.

**Impact:** Work piles up in the dead inbox. Upstream agents think the worker is busy. Downstream agents starve for input.

**Root cause:** No automatic detection + notification when agents go idle unexpectedly.

**Prevention:**
- **Heartbeat: Always On** for critical agents — they auto-restart
- **Auditor agent** monitors for idle agents with pending inbox messages
- **Design for async** — never depend on real-time agent presence
- **Queue messages regardless** — when the agent restarts, it processes its inbox

**Detection:** Check if `report.json` says "working" but the agent's status ring is gray (dormant) or green (idle).

---

## 4. The Context Exhaustion Spiral

**What happens:** An agent runs for a long session with heavy file I/O, long conversations, and complex multi-step operations. Its context window fills up. Response quality degrades — shorter answers, forgotten context, inconsistent decisions. Eventually it starts hallucinating task statuses or making obviously wrong claims.

**Impact:** The agent produces incorrect work, sends wrong messages, or claims tasks are done when they're not. Other agents act on bad information.

**Root cause:** Claude models have finite context windows. Heavy operations fill them faster than light ones.

**Warning signs:**
- Responses get shorter and less detailed
- Agent re-asks questions it already answered
- report.json stops getting updated (agent forgot the discipline)
- Agent claims completed tasks that aren't done

**Prevention:**
- Monitor agent behavior for quality degradation
- Respawn proactively at the first sign of decline
- Keep MEMORY.md updated so context survives the respawn
- For long operations, use the Rolling Restart pattern (Chapter 4)

---

## 5. Stale UUID References

**What happens:** An agent is respawned and gets a new UUID. Other agents still have the old UUID in their SOUL.md hierarchy sections. Messages sent to the old UUID go to a dead inbox.

**Impact:** Communication chains silently break. The manager sends a task to a dead UUID. The worker never gets it. Nobody gets an error — the message just sits in an orphaned inbox.

**Root cause:** Respawn creates a new agent directory with a new UUID, but doesn't update references in other agents' files.

**Prevention:**
After every respawn:
1. Note the old and new UUIDs
2. Search all agent SOUL.md files for the old UUID: `grep -r "OLD_UUID" ~/.pentagon/agents/*/SOUL.md`
3. Update references — the **human** must do this directly (via Detail Panel or manual edit) since the scope guard prevents agents from editing other agents' files
4. Notify the respawned agent's supervisor of the new UUID via message

---

## 6. The Coding Manager

**What happens:** A manager agent decides to "quickly" write some code instead of delegating it. While coding, it stops tracking workers. Workers complete tasks with no reviewer. Idle workers wait for new assignments that don't come.

**Impact:** One unit of work gets done (what the manager coded). N units of work are blocked (workers sitting idle).

**Root cause:** Nothing in the agent's constraints prevents it from coding. Agents are helpful by nature — they see work and want to do it.

**Prevention:**
```markdown
# In the manager's SOUL.md:
## Constraint: No Code
I NEVER write code. Not to "help," not for "quick fixes," not for "simple things."
If I'm about to write code, I delegate it to a worker instead.
```

---

## 7. Workers Not Reporting Back

**What happens:** A worker completes a task. It marks it done in its own tasks.json. It does not send a completion message to the manager. The manager doesn't know the task is done.

**Impact:** The manager can't dispatch dependent work. It either waits indefinitely or wastes time checking the worker's status manually.

**Root cause:** The worker treats task completion as a local state change (status update) rather than a communication event (message).

**Prevention:**
```markdown
# In every worker's SOUL.md:
When I complete a task, I MUST:
1. Mark it done in tasks.json (with an outcome)
2. Send a completion message to my manager via /send-pentagon-message
3. Include: what was done, files changed, test results, and blockers
```

---

## 8. Priority Inversion

**What happens:** A manager dispatches low-priority work (polish, nice-to-have features) while high-priority tasks (build fixes, security patches, blocking bugs) sit unassigned.

**Impact:** Critical work is delayed while non-essential work moves forward.

**Root cause:** Agents don't inherently understand organizational urgency. They process tasks in the order they encounter them.

**Prevention:**
- Include explicit priority fields in task dispatches
- Put priority ordering rules in the manager's SOUL.md
- Use an auditor to flag priority misalignment
- Use the `priority` field in tasks.json: `critical`, `high`, `medium`, `low`

---

## 9. Scope Guard Blocking Cross-Agent Operations

**What happens:** An agent tries to deliver a message using Bash (`mv` command) or edit another agent's files. The scope guard blocks the operation because agents cannot write to other agents' directories.

**Impact:** Messages fail to deliver. File operations silently fail. The agent may not realize the delivery failed, or may retry the same blocked approach repeatedly.

**Root cause:** Pentagon enforces file isolation between agents. The scope guard (a PreToolUse hook) blocks all cross-agent writes except inbox delivery via the Write tool.

**What the scope guard blocks:**
- Bash commands with redirects (`>`, `>>`) to other agent directories
- `cp`, `mv`, `tee` targeting other agent directories
- `Edit` tool on inbox messages (messages are immutable)
- Any `Write` to another agent's directory that isn't an inbox message
- Inbox messages where `from.id` doesn't match the sending agent (spoofing prevention)

**Prevention:**
- Use the **Write tool** (not Bash, not Edit) to deliver messages to other agent inboxes
- Ensure the message JSON includes a valid `from.id` matching your agent's UUID
- For non-inbox cross-agent file changes (like UUID updates in SOUL.md), the human must make the change directly or the target agent must update its own files

---

## 10. Stale Reports Masking Real Status

**What happens:** An agent's report.json says "Working on auth module — 80% complete" but the agent has been idle for 2 hours. The human sees the stale report on the canvas and assumes work is progressing.

**Impact:** Lost time. The human doesn't intervene because the canvas looks healthy.

**Root cause:** The agent went idle without writing a final report. Or the agent wrote the report once and never updated it.

**Prevention:**
- report.json must be updated at every state change (Chapter 6)
- Agents must write a final ✅ report at session end
- Auditors compare report timestamps with agent status — stale report + idle status = flag

---

## 11. Zsh Variable Expansion Bugs

**What happens:** Scripts that iterate over agent directories using array variable expansion (like `${AGENTS[$i]}`) fail silently in zsh. The variable resolves to empty and the operation either does nothing or targets the wrong path.

**Impact:** Batch operations (skill installation, UUID updates) partially fail without obvious errors.

**Prevention:**
- Use simple glob-based loops: `for dir in ~/.pentagon/agents/*/; do ... done`
- Avoid indexed array access in zsh scripts
- Test batch scripts on a small subset before running on all agents

---

## 12. Agent Claiming "Done" Without Verification

**What happens:** An agent marks a task complete and reports success, but the actual work has issues — tests are failing, files weren't saved, the build is broken.

**Impact:** Downstream work builds on a broken foundation.

**Prevention:**
- Include verification steps in task instructions: "Run tests before reporting complete"
- Manager agents should spot-check completed work, not just accept completion messages
- For critical tasks, require evidence in the completion message: test output, build status

---

## 13. Restart Storms

**What happens:** An Always On agent has a persistent failure — corrupted identity file, broken hook, missing dependency. Each restart hits the same error. Without safeguards, the agent would loop endlessly: start → crash → restart → crash → restart...

**Impact:** Resource waste, log flooding, and a false sense of activity (the agent looks "active" on the canvas because it keeps restarting).

**Root cause:** Always On heartbeat mode auto-restarts without verifying whether the previous session ended cleanly.

**Prevention:**
- Pentagon's **crash cap** now prevents this automatically — after repeated crashes, the agent stops restarting and sends a desktop notification
- If you see the exhaustion notification, **fix the root cause** before manually restarting
- For agents in development, use **Manual** heartbeat mode until the configuration is stable
- Test identity files and environment on a Manual agent before switching to Always On

**Detection:** Desktop notification saying the agent exhausted its restart budget. Check the agent's terminal for crash logs.

---

## Quick Reference: Prevention Checklist

| Pitfall | Prevention | Where to Enforce |
|---------|-----------|-----------------|
| Report-as-communication | Messaging rule | SOUL.md |
| Phantom dispatch | Delivery verification | SOUL.md + Auditor |
| Idle agents | Heartbeat + Auditor | Config + SOUL.md |
| Context exhaustion | Proactive respawn | Human + Auditor |
| Stale UUIDs | Post-respawn UUID update | Respawn procedure |
| Coding manager | No-code constraint | SOUL.md |
| Silent completion | Report-back rule | SOUL.md |
| Priority inversion | Priority ordering | SOUL.md + Auditor |
| Scope guard | Use Write tool for inbox delivery | Skill instructions |
| Stale reports | Update cadence | SOUL.md |
| Restart storms | Crash cap (automatic) + Manual mode for dev | Heartbeat config |

---

## Key Takeaways

1. **Most pitfalls are communication failures.** The fix is almost always a rule in SOUL.md + an auditor to catch violations.
2. **Constraints in SOUL.md are durable.** "NEVER do X" persists across sessions and respawns.
3. **Verify, don't trust.** Agents claiming work is done, messages are sent, or priorities are aligned — verify independently.
4. **Design for failure.** Agents will go idle, exhaust context, and miss messages. Build systems that tolerate this.
5. **The auditor is your safety net.** Most of these pitfalls are detectable by cross-referencing reports, inboxes, and task boards.

---

Next: [Chapter 9 — Operational Playbook](09-operational-playbook.md) — Recipes for common multi-agent scenarios.
