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

**2. Create a team on the canvas**

Hold **Shift + click-drag** on the canvas to create a team region of any size. Name it and give it a color. To resize later, hold **Shift** again — the corners light up and you can drag any edge. You can use **auto-layout** to arrange agents automatically after spawning, then fine-tune positions manually.

**3. Spawn the lead agent first**

Click an empty cell inside the team. Configure:
- Name: Something clear (e.g., "ProjectLead")
- Model: Opus (strategic reasoning)
- Directory: Your project root
- SOUL.md: Include hierarchy, constraints, communication rules

**4. Spawn manager agents**

Same process, inside the same team. Each manager's SOUL.md references the lead's UUID.

**5. Spawn worker agents**

Each worker gets:
- A focused SOUL.md with domain boundaries
- Their manager's UUID in the hierarchy section
- tasks.json with initial work items

**6. Verify the chain**

Have the lead send a test message to each manager. Have each manager send a test to their workers. Confirm delivery by checking inboxes.

**7. Spawn the auditor**

Outside the team (or in a separate "Operations" team). SOUL.md marks it as independent. Heartbeat set to Interval (5-10 min).

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

**Tip: Sequential branch work doesn't need separate agents.** Pentagon supports seamless branch switching mid-session — an agent's working directory updates via `--resume` without killing the process. If your exploration is sequential (try approach A, then try approach B), a single agent can switch branches and keep working. Use the A/B pattern when you want **parallel** exploration — both approaches running simultaneously.

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

## Recipe 9: Setting Up Cross-Team Review

### Situation
You have multiple teams (e.g., Frontend, Backend, Infrastructure) and need a review process that spans all of them.

### Steps

**1. Create a "Review" role** via the sidebar (roles are logical, cross-cutting).

**2. Assign one reviewer per team** to the role. Each reviewer belongs to both their team and the Review role.

**3. Use the role's shared task board** for the review queue:

```json
[
  { "id": "pr-142", "title": "Review PR #142: Auth refactor", "status": "backlog", "tags": ["Backend team"] },
  { "id": "pr-145", "title": "Review PR #145: Dashboard redesign", "status": "backlog", "tags": ["Frontend team"] },
  { "id": "pr-147", "title": "Review PR #147: CI pipeline update", "status": "backlog", "tags": ["Infrastructure team"] }
]
```

**4. Each reviewer picks PRs from the shared queue**, reviews them, checks the item off, and reports.

**5. Role MEMORY.md** captures review standards, common issues, and approved patterns.

---

## Recipe 10: Map Organization

### Situation
You're running agents across multiple projects and need to keep them organized. Maps are mandatory in Pentagon — every agent belongs to one. When you open Pentagon, the first view is a map picker (like Figma's file picker).

### Steps

**1. Design your map layout**

Each map corresponds to a project, repo, or concern:

| Map | Purpose | Agents |
|-----|---------|--------|
| `backend-api` | API development | Lead, 2 managers, 6 workers |
| `mobile-app` | Mobile frontend | Lead, 1 manager, 3 workers |
| `infrastructure` | CI/CD, deployment | 1 manager, 2 workers |
| `operations` | Auditing, monitoring | Auditor, status bot |

Many users only need one map — don't create multiple maps unless you have genuinely separate concerns.

**2. Create maps before agents**

Maps must exist before you can spawn agents into them. Create them through the Pentagon map picker.

**3. Assign each agent to the right map at spawn time**

You can't change an agent's map after creation — pick the right one upfront.

**4. Maps are hard boundaries**

Agents on different maps cannot communicate with each other. This is by design — maps are the highest level of separation in Pentagon. If agents need to coordinate, they belong on the same map. Use teams for separation *within* a map.

**5. Reorganizing across maps**

You can cut/copy and paste agents and teams across maps. Copy clones the agent (it won't be linked to the original). This makes it easy to reorganize if your map layout needs to change.

**6. Shared resources**

If multiple maps need the same skill, install it at the role or global level rather than duplicating it per-agent.

---

## Recipe 11: The Auditor-Patcher Workflow

### Situation
You need to audit a codebase and apply fixes, but want separation between finding issues and fixing them.

### Steps

**1. Spawn two agents: an Auditor and a Patcher**

The auditor is read-only — it reviews code but never modifies it. The patcher applies fixes but never decides what to fix.

**2. Configure the Auditor's SOUL.md:**

```markdown
## Role
Code auditor. Read-only. Review the codebase for issues.

## Constraints
- NEVER modify code files
- NEVER push to any branch
- Write findings to structured reports
- Send fix lists to the Patcher via /send-pentagon-message

## Domain Context
[Include the project's design philosophy here — architecture patterns,
intentional trade-offs, tech stack choices. Without this context,
you'll flag intentional patterns as bugs.]
```

**3. Configure the Patcher's SOUL.md:**

```markdown
## Role
Code patcher. Apply fixes from audit findings.

## Constraints
- NEVER decide what to fix — only apply fixes from the Auditor
- NEVER push — commits only, operator approves pushes
- Stay on the assigned branch only
- Report every change with exact file paths, line numbers, and diffs

## Reporting Format
For each fix: (1) file:line and exact change, (2) build output,
(3) test output, (4) git diff --stat, (5) questions or concerns
```

**4. The workflow loop:**

```
Auditor reviews → sends structured findings to Patcher
Patcher applies fixes → sends structured report to Auditor
Auditor re-reviews → confirms fixes or flags regressions
Human approves push
```

**5. Push approval requires structured sign-off:**

```
PUSH APPROVED
- Fix 1: [description] — VERIFIED
- Fix 2: [description] — VERIFIED
- Fix 3: [description] — VERIFIED, with caveat: [note]
No regressions detected. Tests passing.
```

### Why This Works

- **No single agent can both find and "fix" issues without review** — reduces the chance of well-intentioned but incorrect fixes
- **The patcher never decides priority** — that comes from the auditor or the human
- **Structured reports create an audit trail** — every change is documented with evidence
- **The human remains the push authority** — agents propose, humans approve

---

## Recipe 12: Establishing Hard Rules

### Situation
You need map-level constraints that every agent follows and that survive across sessions.

### Steps

**1. Define your hard rules**

These are non-negotiable operational constraints:

```markdown
## Hard Rules
- NO git push from any agent — commits only, human pushes
- Stay on the assigned branch — never checkout other branches
- No AI co-author attribution in commits
- All test failures must be verified against clean HEAD before reporting
```

**2. Add them to every agent's MEMORY.md**

Hard rules go in MEMORY.md (not SOUL.md) because:
- MEMORY.md is read every session and survives respawns
- Agents can be instructed to add new rules to their own MEMORY.md when the operator gives them
- SOUL.md is for identity — rules that change operationally belong in memory

**3. When a rule is violated, propagate the correction**

If one agent breaks a rule, send the correction to ALL agents — not just the violator. One agent's mistake means all agents need the reinforcement.

**4. Pre-existing failure verification**

Before reporting test results or blaming your changes, always verify:

```bash
git stash && npm test && git stash pop
```

If tests fail on clean HEAD, the failures are pre-existing — not yours. This builds trust and prevents false blame in auditor-patcher workflows.

---

## Key Takeaways

1. **Every recipe follows the same pattern:** Plan → Configure identity files → Execute → Verify.
2. **Verification is not optional.** Every recipe includes a verification step because things fail silently.
3. **Start small, scale up.** Don't spawn 10 agents on day one. Start with 3, add as needed.
4. **The auditor makes everything else work.** It catches what you miss. Start it with the team.
5. **Clean restarts beat debugging tangles.** When the state is bad, reset. It's faster than untangling.
6. **Separate audit from execution.** The agent that finds issues shouldn't fix them. The agent that fixes them shouldn't approve the push.
7. **Hard rules in MEMORY.md.** Map-level constraints that survive sessions belong in every agent's memory.

---

Next: [Chapter 10 — Troubleshooting](10-troubleshooting.md) — Common errors and how to fix them.
