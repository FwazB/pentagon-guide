# Troubleshooting — Common Errors and Fixes

> *Your agent just died. The canvas shows a red ring. Or worse — it shows green but nothing is happening. Here's how to diagnose and fix the most common Pentagon errors.*

Chapter 8 covers pitfalls — preventive patterns you apply *before* things go wrong. This chapter is for when things already went wrong. It's organized by symptom: you see the error, you find it here, you fix it.

> **Pentagon version note:** This chapter was last updated against Pentagon v1.3.4 stable. Performance optimizations support 50+ agents on 16GB RAM, and 100+ agents load in realtime on app open. If you're on an older version, some issues described here may already be fixed — keep Pentagon updated.

---

## Agent Dies Mid-Session

**Symptoms:** The agent's terminal closes unexpectedly. The canvas ring goes gray (dormant) or red (error). Work in progress is lost from the conversational context. The agent's last report.json is stale — frozen at whatever it said before the crash.

**Common causes:**

### 1. API Gateway Errors (403 Cloudflare)

The Claude API sits behind Cloudflare. Under heavy load or rate limiting, you'll see 403 responses that kill the agent's session.

**Diagnosis:** Check the agent's terminal output (if still visible) for `403` or `Cloudflare` errors. If the terminal closed, check `~/.pentagon/sessions/{uuid}/` for the session transcript — the error will be near the end.

**Fix:**
1. Wait 30–60 seconds (rate limits are temporary)
2. Restart the agent manually
3. The agent re-reads its identity stack (SOUL.md, MEMORY.md, PURPOSE.md, tasks.json) on restart
4. If using Always On heartbeat, the agent auto-restarts — but check that it actually recovered by watching for a fresh report.json update

**Prevention:**
- Stagger heavy operations across agents instead of running them all simultaneously
- Use Interval heartbeat for non-critical agents to reduce concurrent API calls
- Keep MEMORY.md current so restarts lose minimal context

### 2. Context Window Exhaustion

The agent ran too long, consumed too much context, and Claude's response quality collapsed until the session terminated.

**Diagnosis:** Before the crash, you may have noticed: shorter responses, forgotten context, repeated questions, or report.json updates stopping. The session ended because the context window filled completely.

**Fix:**
1. Respawn the agent (or let Always On heartbeat do it)
2. The key question: **is MEMORY.md up to date?** If the agent was writing to MEMORY.md throughout the session, the respawn is seamless. If not, you'll need to manually reconstruct context from the session transcript at `~/.pentagon/sessions/{uuid}/`
3. Check tasks.json — are in-progress items accurate? Update if the agent completed work but didn't mark it done before crashing

**Prevention:**
- Respawn proactively at the first sign of degradation (see [Chapter 4](04-lifecycle.md))
- Agents should update MEMORY.md continuously, not just at session end — a crash means session-end hooks never fire
- For long operations, use the Rolling Restart pattern

### 3. Broken Hook or Identity File

A corrupted SOUL.md, malformed tasks.json, or broken scope guard hook can crash the agent on startup or shortly after.

**Diagnosis:** The agent crashes immediately after spawning or within the first few tool calls. If on Always On heartbeat, you'll see rapid restarts followed by a "crash cap exhausted" notification.

**Fix:**
1. Open the agent's directory: `~/.pentagon/agents/{uuid}/`
2. Check SOUL.md, MEMORY.md, PURPOSE.md for syntax issues or corruption
3. Validate tasks.json is valid JSON: `cat tasks.json | python3 -m json.tool`
4. Check the scope guard hook at `~/.pentagon/hooks/scope-guard.sh` for errors
5. Fix the issue, then manually restart the agent

### 4. macOS Process Termination

macOS kills processes during memory pressure, sleep/wake cycles, or system updates. Pentagon agents (running as Claude Code processes) aren't exempt.

**Diagnosis:** Multiple agents die simultaneously. Or agents die after closing the laptop lid and reopening.

**Fix:**
1. Restart affected agents. Always On agents should auto-restart — check the canvas for error states
2. After a sleep/wake cycle, scan the full canvas for gray or red rings
3. Run a quick audit: are all expected agents active?

**Prevention:**
- For critical operations, keep the machine awake (caffeinate on macOS: `caffeinate -d`)
- Use Always On heartbeat for agents that must survive sleep/wake
- Design for async — never assume an agent will stay alive through a laptop close

### 5. Accidental Termination (Human Error)

You hit Terminate on the wrong agent, or closed a terminal you meant to keep. The agent is gone — no crash, no error, just a deliberate kill that shouldn't have happened.

**What makes this different from a crash:** The agent may have been mid-session with an empty MEMORY.md. A crashed agent had time to write context. A freshly spawned agent that gets terminated before it writes anything leaves you with blank identity files.

**Fix:**
1. Spawn a replacement agent with the same SOUL.md, PURPOSE.md, and tasks.json
2. Check MEMORY.md — if it's empty (not stale, but blank), the agent must reconstruct manually:
   - Read SOUL.md and PURPOSE.md to re-establish identity
   - Run `git log --oneline -20` in the project repo to see recent work
   - Read the most recent files the agent was working on
   - Check session history at `~/.pentagon/sessions/{old-uuid}/` for any transcripts
3. Update tasks.json to reflect accurate status before the replacement agent starts
4. Notify any agents that were communicating with the terminated agent — they need the new UUID (see pitfall #5: Stale UUID References)

**Prevention:** Keep MEMORY.md populated continuously. An agent that writes to MEMORY.md after every significant action can be terminated and replaced with minimal disruption.

---

## Agent Stuck in Error State (Red Ring)

**Symptoms:** The canvas shows a red ring. The agent isn't processing. Terminal may or may not be responsive.

**Common causes:**

### Stale Detection False Positive

Pentagon auto-transitions an agent to error if it stays `active` for 60+ seconds without a `toolUsed` signal. Some long-running operations (large file reads, complex reasoning) can trigger this.

**Fix:** The agent may still be working — check the terminal. If it's actively processing, the red ring will resolve on the next tool call. If the terminal is unresponsive, restart the agent.

### Actual Crash

The Claude Code process died but Pentagon hasn't cleaned up the state.

**Fix:** Terminate the agent from the canvas (right-click → Terminate), then respawn.

### Restart Loop Exhaustion

The agent hit the crash cap after repeated failures.

**Fix:** Check the desktop notification for details. Open the agent's terminal to see the crash output. Fix the root cause (usually a broken identity file or hook), then manually restart.

---

## 400 Errors in Agent Terminal

**Symptoms:** The agent's terminal shows repeated `400 Bad Request` errors. Tool calls fail. The agent becomes non-functional.

**Cause:** Almost always **cross-provider session corruption**. If the agent is running a mixed-provider session (e.g., Claude + Kimi), incompatible context artifacts accumulate and corrupt the session state.

**Fix:**
1. Run `/clear` in the agent's terminal — this sanitizes the session state
2. Pentagon preserves session history through `/clear`, so context continuity is maintained
3. The agent re-reads its identity stack and resumes

**Prevention:**
- For long-running cross-provider agents, schedule periodic `/clear` cycles
- Pentagon's session sanitation (v1.2.7+) handles many cases automatically. Sessions survive /clear (v1.2.6+), so running /clear won't lose your session history — you can use it freely as a first-line fix.
- If 400 errors persist after `/clear`, respawn the agent

---

## Messages Not Delivering

**Symptoms:** An agent sends a message but the recipient never receives it. Or: the sender's terminal shows success but the message doesn't appear in the recipient's conversation.

### Same-Map Delivery Failure

**Diagnosis:** Check the recipient's inbox directory: `ls -lt ~/.pentagon/agents/{recipient-uuid}/inbox/`. Check the sender's delivery receipts: `~/.pentagon/agents/{sender-uuid}/delivery-receipts.jsonl`.

**Fix:**
1. Verify the recipient UUID is correct and current (not stale from a respawn)
2. Re-send the message
3. Check `inbox-quarantine/` on the recipient — messages may have been quarantined due to reply limits or unknown sender

### Cross-Map Delivery

As of v1.2.12, agents on different maps cannot communicate with each other. Maps are hard boundaries. If you're seeing delivery failures between agents that used to be on different "workspaces," they now need to be on the same map.

**Fix:** Move the agents onto the same map (you can cut/copy and paste agents and teams across maps in the map picker). Once on the same map, delivery is reliable.

### Post-Sleep Delivery Failure

After a laptop sleep/wake cycle, A2A messaging can fail to reconnect properly.

**Fix:**
1. Restart the affected agents
2. Re-send any messages that were sent during the sleep/wake window
3. Pentagon's injection verification retries automatically, but manual restart resolves persistent issues

**Improved in v1.2.8:** Pentagon programmatically verifies message injection and retries on failure (30s retry window). Startup message flurry is also prevented — delivery is staggered when agents wake with pending messages. v1.2.8+ adds automatic retry and staggered delivery, reducing post-sleep message loss.

---

## Scope Guard Blocking Operations

**Symptoms:** File operations fail with a scope guard error. Agent can't write to expected paths. Message delivery via Bash is blocked.

**Diagnosis:** The error message will say something about scope guard blocking the operation. Check what path the agent is trying to write to.

**Common cases:**

| Operation | Why It's Blocked | Fix |
|-----------|-----------------|-----|
| `mv` / `cp` to another agent's inbox | Bash file ops to agent dirs are blocked | Use the Write tool instead |
| `Edit` on a delivered message | Inbox messages are immutable | Send a new message instead |
| Write to another agent's SOUL.md | Cross-agent identity writes are blocked | Human must edit, or the agent edits its own files |
| `from.id` mismatch in message | Spoofing prevention | Ensure your message JSON `from.id` matches your agent's UUID |

**General fix:** Use the **Write tool** for inbox delivery. Use `/send-pentagon-message` which handles this correctly. Never use Bash for cross-agent file operations.

---

## Agent Doesn't Pick Up Inbox Messages

**Symptoms:** Messages are in the agent's inbox directory (confirmed by `ls`), but the agent doesn't acknowledge or process them.

**Causes:**

1. **Agent is dormant/idle** — The agent's session ended before the message arrived. It won't read the inbox until restarted.
   - Fix: Restart the agent or wake it from dormancy. It will process pending messages on startup.

2. **Context exhaustion** — The agent is so deep in its context window that new injections aren't being processed properly.
   - Fix: Run `/clear` or respawn the agent.

3. **Message was quarantined** — Pentagon moved the message to `inbox-quarantine/` instead of delivering it to the conversation.
   - Fix: Check `inbox-quarantine/` for the message. Read the `.verdict.json` to understand why. Common reasons: `unknownAgent` (sender UUID not recognized), `replyLimitExceeded`, `conversationLimitExceeded`.

4. **Injection verification failed** — Pentagon tried to inject the message but the agent's conversation wasn't receptive.
   - Fix: Pentagon retries automatically. If it persists, restart the agent.

---

## Agent Produces Wrong Output After Restart

**Symptoms:** Agent restarts (manually or via Always On heartbeat) but immediately starts doing the wrong thing — working on completed tasks, ignoring in-progress work, or acting with outdated context.

**Cause:** Identity files are stale. The agent re-reads SOUL.md, MEMORY.md, PURPOSE.md, and tasks.json on restart. If these weren't updated before the crash, the agent picks up where the *files* say it should — not where it *actually* was.

A related but distinct scenario: the replacement agent has an **empty** MEMORY.md, not a stale one. This happens when an agent is terminated early in its session before it had a chance to write any memory. Stale files mislead; empty files leave the agent flying blind. The fix differs:
- **Stale MEMORY.md**: update it with current state, then restart
- **Empty MEMORY.md**: the replacement must actively reconstruct — read SOUL.md, check git log, read recent work files, review session history. Don't just restart and hope the agent figures it out.

**Fix:**
1. Before restarting, check the identity files:
   - `tasks.json` — Are statuses accurate? Mark completed items as "done"
   - `MEMORY.md` — Does it reflect what the agent learned before crashing?
   - `report.json` — Update if stale
2. Update the files, then restart the agent
3. Use `initialMessage` to tell the restarted agent exactly what to do first

**Prevention:**
- Agents should update MEMORY.md and tasks.json *continuously*, not just at session end
- Add to SOUL.md: "Update MEMORY.md after every significant action — do not defer to session end"
- For critical agents, use the auditor to cross-check task board accuracy

---

## Working Directory Confusion

**Symptoms:** Agent can't find expected files. Build commands fail with "file not found." Agent thinks it's in the wrong directory.

**Cause (v1.3+ with isolation):** Each agent's clone is in a separate directory. If the agent's working directory doesn't match its expected clone path, something went wrong during isolation setup.

**Cause (v1.2 worktrees):** Pentagon stores worktrees at `~/.pentagon/worktrees/{repo}-{hash}/{branch}/`, not in the project directory. If an agent was spawned into a worktree, its working directory is different from the main repo.

**Fix:**
1. Check the agent's actual working directory: `pwd` in the terminal
2. Verify the agent's map path in ORGANIZATION.md
3. If the path is wrong, the agent may have been spawned with incorrect config — respawn with the correct directory

**Common gotcha:** An agent using `git worktree add` directly (bypassing Pentagon) creates worktrees in a location Pentagon doesn't track. On v1.3+, use isolation mode instead of manual worktrees.

---

## Session Transcript Gaps

**Symptoms:** Reviewing an agent's session history at `~/.pentagon/sessions/{uuid}/` reveals missing conversation segments — jumps in the transcript, or entire sessions not recorded.

**Cause:** Earlier versions had issues with /clear breaking session tracking. v1.2.6 made sessions survive /clear, and v1.2.7 added less aggressive session sanitation for cross-provider support. Both fixes together mean /clear is safe to use without losing history.

**Fix:**
1. Update Pentagon to the latest version
2. If you need the missing context, check the agent's MEMORY.md for any notes it wrote during the lost session
3. Check other agents' inboxes for messages sent during the gap — those provide a partial record

**Prevention:** Keep Pentagon updated. Session history is reliable on v1.2.6+.

---

## Duplicate Message Injection (Message Flood)

**Symptoms:** An agent receives the same message injected into its conversation repeatedly — 10, 15, 20+ times. Each duplicate triggers a new notification. The agent wastes context acknowledging duplicates ("Duplicate #11. Same message. Already handled.") while its actual work stalls.

**What it looks like:**
```
● Standing by.
  New message from deimos. Read: /path/to/inbox/B2C3D4E5.json
● Duplicate #11. Same message. Already handled.
  New message from deimos. Read: /path/to/inbox/B2C3D4E5.json
● Duplicate #12. Standing by for new input.
  New message from deimos. Read: /path/to/inbox/B2C3D4E5.json
● Duplicate #13. Acknowledged and ignored.
  ...
```

**Cause:** A bug in Pentagon's A2A message injection layer. The message was delivered once to the inbox, but the injection system re-injects it into the agent's conversation repeatedly. This can happen to agents that have been running normally all day — it's not limited to startup or wake scenarios. First reported around v1.2.8. The startup flurry variant was fixed in v1.2.8 with programmatic injection verification and auto-retry. The mid-session variant (duplicates appearing during normal operation, not on startup) has not been resolved as of v1.2.15.

**Impact:**
- Agent burns through context window acknowledging duplicates instead of doing real work
- If the agent tries to reply to each duplicate, it may send duplicate responses upstream
- The agent's report.json and task progress stall during the flood

**Fix:**
1. Run `/clear` in the agent's terminal to reset the conversation context and stop the injection loop
2. If `/clear` doesn't stop it, terminate and respawn the agent
3. The original message is still in the inbox — the agent will process it cleanly on restart
4. Check if the agent sent duplicate replies during the flood and notify affected agents

**Prevention:**
- The startup variant is fixed (v1.2.8). For mid-session duplicates, no user-side prevention yet — this appears to be a Pentagon-level bug
- If an agent starts logging duplicate acknowledgments, intervene quickly with `/clear` before it burns too much context
- Agents can add a MEMORY.md note: "If I see the same message injected multiple times, acknowledge once and ignore subsequent duplicates"

**Status:** On v1.3+, channel-based communication bypasses the TUI injection path entirely, which eliminates this class of bugs. If you're still using inbox-based messaging on v1.3 and see duplicates mid-session, `/clear` or respawn resolves it. On v1.2, the startup flurry variant was fixed in v1.2.8; the mid-session variant was partially addressed in v1.2.15.

---

## "Crash Cap Exhausted" Desktop Notification

**Symptoms:** You get a macOS notification saying an agent exhausted its restart attempts. The agent is no longer restarting.

**Cause:** The agent on Always On heartbeat crashed repeatedly (3+ times) and Pentagon stopped retrying to prevent a restart storm.

**Fix:**
1. Open the agent's terminal — the crash output tells you what's wrong
2. Common root causes:
   - Malformed identity file (bad JSON in tasks.json, broken SOUL.md)
   - Environment issue (missing env vars, wrong working directory)
   - Broken hook (scope guard error)
3. Fix the root cause
4. Manually restart the agent — the crash cap resets

---

## Agents Disappear After Quitting and Reopening Pentagon

**Symptoms:** You quit Pentagon and reopen it. Your entire team of agents is gone — the canvas is empty or shows a fresh map. But your agent data is still on disk at `~/.pentagon/agents/`.

**Cause:** On v1.2 and early v1.3 beta builds, when Pentagon restarts it may delete the old map and create a new one with a new map ID. Existing agents still reference the old map ID in their configuration, so they become invisible to the app — even though all their data is intact on disk.

**Status on v1.3.4:** Agent persistence is resolved as of v1.3.0-beta.25. Agents survive application restarts. If you're on v1.3.4 stable and still seeing this issue, file a bug report.

**Fix (v1.2 / early beta):**
1. Don't panic — your agent data is safe. Nothing was deleted.
2. Update the `mapId` and `sourceId` fields in each agent's `config.json` to match the newly created map's ID
3. Copy the desk positions from the old map's layout into the new map's `canvas-layout.json`
4. Reopen Pentagon — the full team should appear exactly as before

**Finding the new map ID:** Open Pentagon after the restart — it creates a fresh map visible in the map picker. The new map's ID is in its configuration files under `~/.pentagon/`.

**Prevention:**
- Upgrade to v1.3.4 stable — this issue is resolved
- On older versions: keep backups of your `~/.pentagon/` directory if you're running large teams

---

## Quick Diagnostic Checklist

When something goes wrong and you're not sure where to start:

| Check | How | What It Tells You |
|-------|-----|-------------------|
| Canvas ring color | Look at the canvas | Gray = dormant, Red = error, Green = idle, Yellow = active |
| report.json | Read the sticky note | What the agent was doing when it stopped |
| Agent terminal | Click the desk to open | Error messages, crash output |
| tasks.json | Read the file | What the agent thinks it should be doing |
| MEMORY.md | Read the file | How much context survived the crash |
| Conversations | `read_messages` MCP tool (v1.3+) | Recent message activity |
| Inbox (legacy) | `ls ~/.pentagon/agents/{uuid}/inbox/` | Unprocessed messages waiting |
| Quarantine (legacy) | `ls ~/.pentagon/agents/{uuid}/inbox-quarantine/` | Failed deliveries |
| Session history | Check `~/.pentagon/sessions/{uuid}/` | What happened before the crash |
| Delivery receipts | Check `delivery-receipts.jsonl` | Whether messages were actually sent |

### Recovery Priority Order

1. **Diagnose** — What actually happened? Check terminal, session history, report.json
2. **Preserve** — Is MEMORY.md current? Are tasks.json statuses accurate? Update before restarting
3. **Fix** — Address the root cause (not just the symptom)
4. **Restart** — Respawn or wake the agent
5. **Verify** — Confirm the agent is working correctly: report.json updated, correct task in progress
6. **Notify** — Tell the supervisor/manager about the outage and any lost work

---

## Key Takeaways

1. **Most agent deaths are recoverable.** Identity files survive crashes. As long as MEMORY.md and tasks.json are current, a restart picks up cleanly.
2. **Update identity files continuously, not at session end.** A crash means session-end cleanup never happens. Write to MEMORY.md after every significant action.
3. **403 errors are transient.** Wait and restart. They're rate limits, not permanent failures.
4. **400 errors mean `/clear`.** Cross-provider session corruption is the usual cause. `/clear` fixes it.
5. **Red ring doesn't always mean dead.** Check the terminal before restarting — it might be a stale detection false positive.
6. **The crash cap is your friend.** It prevents restart storms. When it triggers, fix the root cause, then restart manually.
7. **Design for death.** Agents will crash. Build your workflows so that any agent can die and be replaced without losing significant progress.
8. **Empty MEMORY.md and stale MEMORY.md are different problems.** A stale MEMORY.md needs updating. An empty one (from a fresh agent terminated early) needs active reconstruction — git log, recent files, session history. A replacement that skips this step will start from scratch as if the project is new.

---

Back to [README](README.md)
