# Chapter 11 — The v1.3 Beta

## What Changed, What Migrates, and What to Watch

> *Beta is where you discover which of your assumptions were actually guarantees.*

Pentagon v1.3.0 beta introduces isolation mode, channel-based communication, agent persistence, message queueing, and a rebuilt chat interface. This chapter documents what changed from the v1.2 stable line, what to expect during migration, and what to watch during adoption — drawn from community testing across beta builds 19 through 25.

Agents created on the v1.3 beta carry forward to stable when v1.3 graduates. Agents from v1.2 stable do not transfer to beta. If you plan to be on v1.3 stable, building your team on the beta is the path forward.

---

## Migration

### Stable to Beta

v1.3.0 beta is a clean break from v1.2 stable. Agent state, maps, and configurations do not transfer. You rebuild your agents from scratch.

What carries over:
- Your SOUL.md, MEMORY.md, and CLAUDE.md files (they live in your repo, not Pentagon's state)
- Your hierarchy designs, communication patterns, and lifecycle conventions
- Your git repos and branches

What doesn't:
- Agent UUIDs, conversation IDs, inbox paths
- Map layouts, team configurations, canvas positions
- Heartbeat settings, permission configurations

### Beta to Stable

Per public release notes, agents created on v1.3 beta persist through the stable graduation. Build your team on beta with the expectation that these agents become your stable agents.

---

## What's New in v1.3

### Isolation Mode

v1.3 introduces **isolation mode** — an opt-in setting that gives each agent a completely independent clone of every repository it works with. This replaces the git worktree model documented in [Chapter 4 — Git Worktree Isolation](04-lifecycle.md#git-worktree-isolation).

| | Worktrees (v1.2) | Isolation (v1.3) |
|---|---|---|
| **Mechanism** | Git worktrees sharing a single `.git` directory | Full independent clones |
| **Default** | On (automatic for second+ agents) | Off (enable in settings) |
| **Flexibility** | Limited — worktree branches are linked to the parent repo | Full — each clone behaves as a standalone repo |
| **Interaction model** | Pentagon's git wrapper manages worktree lifecycle | Agent interacts with its repo like a human engineer on a team |

**Enabling isolation:** Toggle it on in Pentagon's settings. Each agent then operates on its own clone with full git capabilities — branching, stashing, rebasing — without risk of conflicts with other agents.

**Impact on existing patterns:**
- The [worktree merge process](CLAUDE.md) (`git diff | git apply`) no longer applies when isolation is enabled — agents on separate clones push and pull through git like any team member
- Pentagon's git wrapper at `~/.pentagon/bin/git` may behave differently under isolation — verify commands before relying on them in automation
- Agents with isolation enabled can't see each other's uncommitted work (same as worktrees), but the merge strategy shifts from worktree-to-main to standard branch merging

**Caution (beta.19):** Community testers observed isolation bugs related to branch switching and source directory changes in beta.19. Subsequent beta builds addressed the reported bugs, but edge cases may surface. After enabling isolation and switching branches, verify the agent's working directory and current branch before proceeding.

### Channel-Based Communication

v1.3.0-beta.25 introduces **channels** — slack-like conversation spaces for agent communication. This is a shift from the inbox-based messaging model documented in [Chapter 4b — A2A Communication](04b-a2a-communication.md).

On v1.2, agents communicate by writing JSON files to each other's inbox directories. On v1.3 beta, agents communicate through channels. Per public release notes, team-level channels are forthcoming.

**What this means for SOUL.md instructions:**
- The `send_message` MCP tool targets conversations and channels rather than inbox file paths
- Communication instructions in SOUL.md should reference conversation IDs, which Pentagon provides to agents at spawn time
- The inbox-based filesystem messaging model (Write tool to `~/.pentagon/agents/{uuid}/inbox/`) may still function, but channels are the primary communication path
- See [Chapter 2 — Communication](02-communication.md) for the messaging fundamentals that still apply regardless of transport mechanism

### Agent Persistence

On v1.2 stable and early beta builds, agents could disappear from the map after quitting and reopening Pentagon (see [Chapter 10 — Troubleshooting](10-troubleshooting.md) for the map ID mismatch issue). v1.3.0-beta.25 resolves this — agents persist across application restarts.

This makes patterns that depend on agents surviving restarts more viable: [Always On heartbeat](04-lifecycle.md#heartbeat-system), scheduled polling, [auditor loops](05-auditor-pattern.md). On v1.2, these patterns required workarounds for the disappearance problem. On v1.3, they work as designed.

### Message Queueing

v1.3.0-beta.21 introduces automatic message queueing. Multiple messages sent to the same conversation queue and dequeue in order without the sender needing to wait for each to be processed.

On v1.2, sending multiple rapid messages to an agent could cause race conditions — messages might arrive out of order or collide during injection (see [Chapter 4b — Stage 4: Injection](04b-a2a-communication.md#stage-4-injection)). With queueing, the system handles sequencing.

**Practical impact:** Manager agents that fan out tasks to multiple workers can send all assignments without waiting for acknowledgment between each send. The messages queue and each worker processes them in order. This eliminates the send-wait-send pattern that was necessary on v1.2.

### Chat Interface

The chat UI was rebuilt in beta.19 and iterated on through beta.25. Relevant changes for agent operators:

- **Activity visibility:** Agents display what they're doing near the chat input (tool calls, file reads, specific actions) rather than a generic "working" indicator
- **Multiline input:** Multiline select restored in beta.19
- **A2A transparency:** Agent-to-agent communication is visible in DMs (beta.20+), giving operators insight into what agents are saying to each other without reading inbox files

### Auth and Session Fixes

v1.3.0-beta.19 fixed two auth bugs that affected earlier beta builds:

1. **Token refresh:** OAuth tokens were not refreshing, causing 401 authentication errors during long sessions. This was a common failure mode on earlier beta builds — agents would work fine for the first portion of a session, then fail on every API call.
2. **Spawn conversation ID:** Agents that spawned a sub-agent and immediately sent it a message received the wrong conversation ID for the new agent. Messages went to the wrong destination silently — the sender received no error, and the intended recipient never got the message.

Both are resolved in beta.19+. If you experienced auth failures or silent message loss after spawning on earlier builds, upgrade.

---

## Beta-Specific Failure Modes

These follow the [pitfall format from Chapter 8](08-common-pitfalls.md): What happens → Impact → Root cause → Prevention → Detection.

### Isolation Bugs on Branch Switching

**What happens:** With isolation enabled, switching branches or changing source directories causes the agent to lose track of its working directory or operate on the wrong branch.

**Impact:** Agent reads or writes unexpected files. Commits land on the wrong branch.

**Root cause:** Isolation clone management had bugs in beta.19 related to branch and source directory changes. Subsequent beta builds addressed the issue, but it may not be fully resolved across all edge cases.

**Prevention:** After enabling isolation and switching branches, verify the agent's working directory (`pwd`) and current branch (`git branch --show-current`) before proceeding with work. Inspect tool calls in the UI for directory paths.

**Detection:** Agent references files that don't exist on the expected branch. Git status shows an unexpected branch name.

### Retry Exhaustion on Large Context

**What happens:** Agent hits "Failed to respond after 3 attempts" during intensive operations (large writes, many parallel reads, long conversations).

**Impact:** Agent stops responding. Pentagon does not restart it — this is a response failure within a live session, distinct from [heartbeat restarts](04-lifecycle.md#heartbeat-system). Work in progress is lost if [MEMORY.md](01-agent-identity.md) wasn't updated.

**Root cause:** Context window exceeded the model's ability to respond within timeout. All 3 retries fail identically because the context doesn't shrink between attempts.

**Prevention:** Keep context lean. Batch reads (3-4 files max per parallel call). Write findings to MEMORY.md before starting the next phase. Don't accumulate long conversation histories for multi-phase work.

**Detection:** Pentagon UI shows the failed response indicator after 3 attempts.

### UI Reconnection After Sleep

**What happens:** After laptop sleep, the Pentagon UI loses its realtime connection. Agent status indicators become stale. Messages sent during the disconnection period may not appear until reconnection completes.

**Impact:** You see an agent as idle when it's working, or working when it's stopped. Communication timing becomes unreliable during the reconnection window.

**Root cause:** Realtime WebSocket connections drop during system sleep. beta.25 improved monitoring and automatic reconnection, but edge cases remain.

**Prevention:** After waking from sleep, give the UI a few seconds to reconnect before relying on agent status indicators or sending messages.

**Detection:** Agent status rings don't match expected behavior. Sent messages don't appear in the chat immediately.

---

## Patterns That Work

### Memory-First Workflow

Because beta conversations are more likely to be interrupted (retry exhaustion, session instability), adopt a memory-first approach (see [Chapter 1 — MEMORY.md](01-agent-identity.md) for the file's role in agent identity):

1. **Before starting work:** Read MEMORY.md to pick up where you left off
2. **After each milestone:** Update MEMORY.md with findings, decisions, and next steps
3. **Before large operations:** Save current state so a restart doesn't lose progress
4. **Task board as checkpoint:** Update tasks.json status at every transition — it survives respawns

This isn't just good practice — on beta, it's essential. An agent that updates memory continuously loses minutes on a restart. An agent that planned to "update memory at the end" loses everything.

### Static Teammate Registry

Runtime agent discovery has improved on v1.3 beta, but maintaining a hardcoded teammate registry remains best practice. In your [SOUL.md or MEMORY.md](01-agent-identity.md):

```
## Teammates
- **agent-name** (UUID) — role, conversation ID
```

This registry means your agent can communicate immediately after spawn without waiting for discovery calls. Update it when teammates are spawned or despawned. See [Chapter 2 — Communication](02-communication.md) for messaging fundamentals.

### Defensive Reads

Instead of reading 7 files in parallel:

```
# Too aggressive — likely to cause retry exhaustion
Read x 7 (parallel)

# Safe batch size
Read x 3 (parallel), then Read x 3 (parallel), then Read x 1
```

The extra round-trips are cheaper than a retry failure that kills the conversation.

---

## Version Reference: v1.3 Beta Builds

| Build | Key Changes |
|-------|------------|
| beta.19 | Auth token refresh fix; spawn conversation ID fix; chat UI rebuilt from scratch; multiline input restored; agent activity visibility; isolation mode introduced (with known bugs) |
| beta.20 | ~30 stability PRs; agents stopping randomly addressed; agent action visibility in DMs |
| beta.21 | Message queueing (multi-message send without waiting); nearing stable graduation |
| beta.25 | Agent persistence fixed (no more map disappearance); communication stability improvements; UI reconnection monitoring after sleep; chat polish; channel-based A2A communication introduced; isolation refinements |

---

## Key Takeaways

1. **v1.3 is a clean break from v1.2 — but beta agents carry to stable.** Agents don't migrate from v1.2 stable to v1.3 beta. Building on beta is building for v1.3 stable.
2. **Isolation mode replaces worktrees.** Opt-in via settings — each agent gets a full independent repo clone instead of a shared worktree. More flexible, but changes the merge strategy from worktree-apply to standard branch merging.
3. **Channels replace inbox-based messaging.** Agents communicate through slack-like channels rather than writing JSON files to inbox directories. Team-level channels are forthcoming per public release notes.
4. **Message queueing eliminates send-wait-send patterns (beta.21+).** Multiple messages to the same conversation queue automatically — managers can fan out tasks without waiting between sends.
5. **Agent persistence is resolved (beta.25+).** Agents survive application restarts. The map ID mismatch issue documented in [Chapter 10](10-troubleshooting.md) is fixed on v1.3.
6. **Auth tokens refresh correctly (beta.19+).** Long sessions no longer fail with 401 errors. Spawn-then-message conversation routing is also fixed.
7. **Keep context lean — retry exhaustion is still the most common failure mode.** Batch reads at 3-4, update MEMORY.md continuously, don't let conversations grow unbounded.

---

Back to [README](README.md)
