# Operational Playbook — Recipes for Common Scenarios

> *Patterns you can copy-paste into your operations. Each recipe is a complete workflow — setup, execution, and verification.*

This chapter is a collection of operational recipes for multi-agent Pentagon teams. Each recipe covers a real scenario with step-by-step instructions.

---

## Recipe 1: Setting Up a New Team from Scratch

### Situation
You're starting a new project and need to stand up a multi-agent team.

### Steps

**1. Plan the hierarchy**

Before spawning anything, decide:
- How many tiers? (Usually 3: Lead → Manager → Workers)
- What domains? (Each worker needs a clear area)
- What model per tier? (Opus for leads, Sonnet for managers, Haiku for simple workers)

**2. Create a pod on the canvas**

Switch to Pod mode in the canvas toolbar. Draw a rectangular region for your project. Name it and give it a color. You can use **auto-layout** to arrange agents automatically after spawning, then fine-tune positions manually.

**3. Spawn the lead agent first**

Click an empty cell inside the pod. Configure:
- Name: Something clear (e.g., "ProjectLead")
- Model: Opus (strategic reasoning)
- Directory: Your project root
- SOUL.md: Include hierarchy, constraints, communication rules

**4. Spawn manager agents**

Same process, inside the same pod. Each manager's SOUL.md references the lead's UUID.

**5. Spawn worker agents**

Each worker gets:
- A focused SOUL.md with domain boundaries
- Their manager's UUID in the hierarchy section
- tasks.json with initial work items

**6. Verify the chain**

Have the lead send a test message to each manager. Have each manager send a test to their workers. Confirm delivery by checking inboxes.

**7. Spawn the auditor**

Outside the pod (or in a separate "Operations" pod). SOUL.md marks it as independent. Heartbeat set to Interval (5-10 min).

---

## Recipe 2: Morning Standup via Auditor

### Situation
You want a daily summary of team status without manually checking each agent.

### Steps

**1. Create an auditor agent** (if you don't have one)

Set heartbeat to **Scheduled** or **Interval** for the start of your workday.

**2. Add standup tasks to tasks.json:**

```json
[
  { "id": "read-reports", "title": "Read all agent report.json files", "status": "backlog" },
  { "id": "check-inboxes", "title": "List all unread inbox messages across all agents", "status": "backlog" },
  { "id": "idle-check", "title": "Identify idle agents with pending work", "status": "backlog" },
  { "id": "stale-check", "title": "Check for stale reports (>4 hours unchanged)", "status": "backlog" },
  { "id": "write-standup", "title": "Write standup summary to report.json", "status": "backlog" },
  { "id": "flag-blockers", "title": "Flag any blockers with 👋", "status": "backlog" }
]
```

**3. The auditor wakes, runs the checklist, and writes:**

```json
{
  "summary": "Standup — 8 agents reviewed, 1 blocker found",
  "detail": "6 active, 1 dormant, 1 idle with pending tasks. UIWorker blocked on design spec. All other agents progressing normally."
}
```

**4. You read the canvas** and address any flags.

---

## Recipe 3: Emergency Agent Replacement

### Situation
A critical agent (e.g., a manager) has exhausted its context and is producing unreliable output.

### Steps

**1. Harvest context**

```
Read: {agent-dir}/SOUL.md
Read: {agent-dir}/MEMORY.md
Read: {agent-dir}/tasks.json
Read: {agent-dir}/report.json
```

Evaluate what's still accurate. If the agent was hallucinating, cross-reference with actual files and other agents' reports.

**2. Note the old UUID**

You'll need this to update references.

**3. Terminate the agent**

Select the desk on the canvas → Delete (or context menu → Terminate).

**4. Spawn the replacement**

Click the now-empty cell. Configure with:
- Same SOUL.md (with any corrections)
- Same MEMORY.md (carried over from the old agent)
- Updated tasks.json (set incomplete items to backlog, remove done items)
- initialMessage: Tell the new agent exactly what to do first

**5. Update UUID references**

Search all agent SOUL.md files for the old UUID:

```bash
grep -rl "OLD_UUID" ~/.pentagon/agents/*/SOUL.md
```

**Note:** The scope guard prevents agents from editing other agents' files. UUID updates in SOUL.md must be done by the human directly (via the Detail Panel or manual file edit), or by each agent updating its own SOUL.md in its next session.

**6. Notify the supervisor**

Send a message to the replacement's supervisor:
```
[AgentName] has been respawned. New UUID: [new-uuid].
Old UUID [old-uuid] is dead. Please re-send any pending work.
```

**7. Verify activation**

Check that the new agent:
- Updated its report.json
- Read its identity files
- Started processing its initialMessage

---

## Recipe 4: Batch Skill Distribution

### Situation
You've written a custom skill and need to distribute it to all agents.

### Steps

**1. Write the skill**

Create a SKILL.md file with usage instructions, steps, and examples.

**2. Test on one agent**

Install it to a single agent's skills directory and verify it works.

**3. Distribute to all agents**

```bash
SKILL_DIR="/path/to/my-skill"
for agent_dir in ~/.pentagon/agents/*/; do
  mkdir -p "$agent_dir/skills/my-skill"
  cp "$SKILL_DIR/SKILL.md" "$agent_dir/skills/my-skill/SKILL.md"
done
```

**4. Verify**

```bash
for agent_dir in ~/.pentagon/agents/*/; do
  if [ -f "$agent_dir/skills/my-skill/SKILL.md" ]; then
    echo "✅ $(basename $agent_dir)"
  else
    echo "❌ $(basename $agent_dir)"
  fi
done
```

---

## Recipe 5: The A/B Exploration Pattern

### Situation
You have a hard problem and want two agents to try different approaches in parallel.

### Steps

**1. Spawn two worker agents** on the same repo. Pentagon gives each its own worktree.

**2. Give them different SOUL.md instructions:**

Agent A:
```markdown
## Approach
Implement using server-side rendering with Next.js App Router.
Focus on SEO and initial load performance.
```

Agent B:
```markdown
## Approach
Implement using client-side rendering with React SPA.
Focus on interactivity and transition smoothness.
```

**3. Give them the same tasks.json** — identical requirements.

**4. Let them work independently.** Each is in its own worktree, no conflicts.

**5. Compare results.** Review both implementations. Merge the winner, discard the other.

**6. Clean up** — Terminate the losing agent and delete its worktree.

---

## Recipe 6: Scaling from 3 to 10 Agents

### Situation
You started with a small team and need to scale up.

### Steps

**1. Don't add all agents at once.** Scale one tier at a time.

**2. Add a manager** (if you don't have one)

3 agents can self-coordinate. 5+ cannot. Designate or spawn a manager before adding more workers.

**3. Define domain boundaries** for new workers

Before spawning a new worker, clearly define what it owns and what it doesn't. Update existing workers' SOUL.md to exclude the new domain.

**4. Update the manager's SOUL.md** with new worker UUIDs.

**5. Spawn the auditor** (if you haven't already)

At 5+ agents, sync failures become likely. Start the auditor before you need it.

**6. Test the communication chain** — Lead → Manager → each Worker and back.

**7. Review model allocation**

| Count | Recommendation |
|-------|---------------|
| 3-5 agents | Sonnet everywhere, Opus for lead |
| 5-8 agents | Mix models by role |
| 8+ agents | Haiku for simple workers to manage cost |

---

## Recipe 7: Recovering from a Swarm Failure

### Situation
Multiple agents have stale context, wrong UUIDs, or conflicting task boards. The team is in a bad state.

### Steps

**1. Document the desired end state**

Write down: What should each agent be working on? What are the current priorities? What's the correct hierarchy?

**2. Terminate all affected agents**

Don't try to untangle the mess incrementally. A clean restart is faster.

**3. Respawn in hierarchical order**

```
Lead (first)    → Verify it's working
Managers (next) → Lead confirms each one
Workers (last)  → Managers confirm each one
Auditor (final) → Independent, reports to you
```

Each tier confirms operation before the next tier spawns. This ensures the communication chain is clean.

**4. Have the lead dispatch the current priorities**

The lead reads the documented end state and dispatches to managers, who dispatch to workers.

**5. Auditor verifies**

Run a full audit cycle to confirm all messages landed and all agents are working on the right things.

**Expected time:** ~10 minutes for a team of 10. Far faster than debugging a tangled state.

---

## Recipe 8: Cost-Optimized Team Configuration

### Situation
You need to run a team efficiently without excessive API costs.

### Steps

**1. Right-size your models**

| Role | Model | Reasoning |
|------|-------|-----------|
| Lead | Opus | Complex strategy, few calls |
| Manager | Sonnet | Task decomposition, moderate complexity |
| Complex worker | Sonnet | Nuanced implementation |
| Simple worker | Haiku | Straightforward tasks, high throughput |
| Auditor | Haiku | Scan + report loop, low complexity |

**2. Use dormancy aggressively**

Set idle agents to dormant instead of letting them sit active. Dormant agents cost nothing.

**3. Use interval heartbeat wisely**

Don't set every agent to Always On. Use Interval for agents that only need to work periodically (auditor, review bot).

**4. Batch related tasks**

Instead of 5 workers each handling one tiny task, assign 3 workers with 2 tasks each. Fewer agents = lower overhead.

**5. Monitor and prune**

If a worker has been dormant for days with no pending tasks, terminate it. You can always respawn later.

---

## Recipe 9: Setting Up Cross-Pod Review

### Situation
You have multiple pods (e.g., Frontend, Backend, Infrastructure) and need a review process that spans all of them.

### Steps

**1. Create a "Review" team** via the sidebar (teams are logical, cross-cutting).

**2. Assign one reviewer per pod** to the team. Each reviewer belongs to both their pod and the Review team.

**3. Use the team's shared task board** for the review queue:

```json
[
  { "id": "pr-142", "title": "Review PR #142: Auth refactor", "status": "backlog", "tags": ["Backend pod"] },
  { "id": "pr-145", "title": "Review PR #145: Dashboard redesign", "status": "backlog", "tags": ["Frontend pod"] },
  { "id": "pr-147", "title": "Review PR #147: CI pipeline update", "status": "backlog", "tags": ["Infrastructure pod"] }
]
```

**4. Each reviewer picks PRs from the shared queue**, reviews them, checks the item off, and reports.

**5. Team MEMORY.md** captures review standards, common issues, and approved patterns.

---

## Key Takeaways

1. **Every recipe follows the same pattern:** Plan → Configure identity files → Execute → Verify.
2. **Verification is not optional.** Every recipe includes a verification step because things fail silently.
3. **Start small, scale up.** Don't spawn 10 agents on day one. Start with 3, add as needed.
4. **The auditor makes everything else work.** It catches what you miss. Start it with the team.
5. **Clean restarts beat debugging tangles.** When the state is bad, reset. It's faster than untangling.

---

Back to [README](README.md)
