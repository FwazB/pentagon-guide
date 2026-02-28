# Skills and Tools — Extending Agent Capabilities

> *A skill is a slash command that gives an agent a new capability without rewriting its soul. Drop a folder in the right directory and every agent on the team gets the power.*

Pentagon agents run on Claude Code, which means they inherit all of Claude Code's tool capabilities — file I/O, Bash, search, web access, and more. Pentagon adds a layer on top: **skills** (custom slash commands) and **hooks** (lifecycle event scripts) that extend what agents can do and how they integrate with Pentagon's monitoring.

---

## Skills

### What Is a Skill?

A skill is a directory containing a `SKILL.md` file that defines a slash command. When an agent types `/skill-name`, the skill's instructions are injected into the conversation, giving the agent new capabilities or workflows.

### Skill Sources (Priority Order)

Skills come from four sources. On name collision, higher priority wins:

1. **Agent skills** — `~/.pentagon/agents/{uuid}/skills/`
2. **Team skills** — Shared across team members
3. **Role skills** — Shared across agents with the same role
4. **Pentagon global skills** — Available to all agents

### Installing a Skill

To give an agent a skill:

```
~/.pentagon/agents/{uuid}/skills/{skill-name}/SKILL.md
```

To give every agent on the team a skill, install it at the team level.

### Batch Installation

To distribute a skill to all agents at once:

```bash
# For each agent directory
for agent_dir in ~/.pentagon/agents/*/; do
  mkdir -p "$agent_dir/skills/my-skill"
  cp /path/to/SKILL.md "$agent_dir/skills/my-skill/SKILL.md"
done
```

⚠️ **Bash variable expansion gotcha:** In zsh, `${AGENTS[$i]}` may not work as expected. Use simple `for` loops over glob patterns instead.

### Built-In Pentagon Skills

Pentagon ships with several built-in skills:

| Skill | Purpose |
|-------|---------|
| `/send-pentagon-message` | Send messages to other agents |
| `/manage-pentagon-agent` | Spawn and configure agents |
| `/manage-pentagon-team` | Create and manage teams |
| `/manage-pentagon-canvas` | Arrange agents on the canvas |
| `/manage-pentagon-role` | Create organizational roles |

These are the backbone of multi-agent coordination. `/send-pentagon-message` in particular is the primary communication mechanism (see [Chapter 2](02-communication.md)).

### Writing Custom Skills

A SKILL.md file is a markdown document that becomes part of the agent's prompt when invoked. It should include:

1. **Description** — What the skill does
2. **Usage** — How to invoke it, expected arguments
3. **Instructions** — Step-by-step procedure the agent follows
4. **Examples** — Concrete usage examples

```markdown
# Deploy Skill

Deploy the current branch to the staging environment.

## Usage
/deploy [environment]

## Instructions
1. Run the test suite: `npm test`
2. If tests pass, build: `npm run build`
3. Deploy using: `npm run deploy:${environment}`
4. Verify deployment: `curl https://${environment}.example.com/health`
5. Report result to the deploying manager via /send-pentagon-message

## Examples
/deploy staging
/deploy production
```

### Skill Design Principles

- **Single responsibility.** One skill = one capability.
- **Self-contained.** The skill should include all instructions needed. Don't assume the agent knows the workflow.
- **Idempotent when possible.** Running the skill twice should be safe.
- **Report results.** The skill should end by reporting what happened — to report.json, to the assigning agent, or both.

### Skills and Git

Pentagon-injected skills (symlinked into your agent's skill directory) are excluded from git tracking. They won't appear in your agent's PRs or branches. This means you can install skills freely without polluting the project's version control.

---

## Hooks

### What Are Hooks?

Pentagon uses Claude Code's hook system to enforce agent boundaries. The primary hook is the **scope guard** — a `PreToolUse` hook that runs before every file write and Bash command.

### The Scope Guard

The scope guard (`~/.pentagon/hooks/scope-guard.sh`) enforces file isolation between agents. It prevents agents from accidentally or intentionally modifying each other's files.

**What it blocks:**

| Action | Allowed? | Details |
|--------|----------|---------|
| Write to own agent directory | ✅ | Always allowed |
| Write to another agent's directory | ❌ | Blocked with error message |
| Write to another agent's **inbox** | ✅ | Allowed via Write tool only, with sender validation |
| Edit a message in another agent's inbox | ❌ | Messages are immutable once delivered |
| Bash redirect to another agent's directory | ❌ | Heuristic scan catches `>`, `>>`, `tee`, `cp`, `mv` |
| Write to project files, non-agent paths | ✅ | No restriction |

**Sender validation:** When an agent writes to another agent's inbox, the scope guard validates that:
1. The content is valid JSON (messages must be structured)
2. The `from.id` field matches the sending agent's `PENTAGON_AGENT_ID`
3. The tool used is `Write` (not `Edit` — inbox messages are immutable)

This prevents message spoofing — an agent cannot send a message pretending to be a different agent.

### Status Tracking

Pentagon tracks agent status (the colored ring on each desk) natively through process monitoring. The status pipeline is:

```
Claude Code process signals → Pentagon StatusEngine → Canvas ring color
```

You don't need to manage status tracking — it's handled automatically by Pentagon.

### You Don't Manage Hooks

The scope guard is auto-generated and auto-managed. You don't need to edit it. If an agent's file operations are being unexpectedly blocked, check that `PENTAGON_AGENT_ID` and `PENTAGON_BASE` environment variables are set correctly in the agent's session.

---

## Tool Usage Patterns

Claude Code provides a rich set of tools. Here's how they interact with Pentagon operations:

### File Tools

| Tool | Use For | Pentagon Relevance |
|------|---------|-------------------|
| **Read** | Reading files | Reading other agents' reports, inboxes, configs |
| **Write** | Creating/overwriting files | Writing to agent inboxes (message delivery) |
| **Edit** | Modifying files | Updating SOUL.md, MEMORY.md, PURPOSE.md, tasks.json |
| **Glob** | Finding files by pattern | Scanning inbox directories |
| **Grep** | Searching file contents | Finding references to UUIDs, agent names |

### The Write Tool for Cross-Agent Operations

The Write tool is the **only** tool that can deliver messages to other agent inboxes. The scope guard blocks Edit (messages are immutable) and Bash redirects to agent directories.

Use Write for:
- Delivering messages to other agent inboxes (the only allowed method)
- Creating files in your own agent directory

**Important:** You can no longer use Write to modify other agents' identity files (SOUL.md, MEMORY.md, etc.). The scope guard blocks all cross-agent writes except inbox delivery. To update another agent's files after a respawn, the human must do it directly or through the agent's own session.

### The Bash Tool: Use Carefully

Bash is powerful but has scope guard constraints:

- **Redirects and file operations** (`>`, `>>`, `tee`, `cp`, `mv`) targeting other agent directories are blocked
- The scope guard scans Bash commands heuristically — it catches common patterns but not obfuscated ones
- **Variable expansion** in zsh can behave unexpectedly (avoid `${ARRAY[$i]}` patterns)
- **Prefer dedicated tools** over Bash equivalents: Read over `cat`, Glob over `find`, Grep over `grep`

### The Task Tool: Parallel Work

Claude Code's Task tool spawns subagents for parallel research or exploration. In a Pentagon context, this is useful for:

- Scanning multiple agent directories simultaneously
- Researching codebase questions without consuming the main agent's context
- Running parallel verification checks

---

## Workspace and Directory Access

### The Agent's Working Directory

Each agent is assigned a project directory at spawn time. This is where it operates — reading code, writing files, running commands. Pentagon sets this via the config at spawn.

### Git Worktree Isolation

Agents on the same repo get isolated worktrees. Each agent works on its own branch:

```
Repo root/
├── .git/
├── .claude/worktrees/
│   ├── agent-1-branch/  ← Agent 1's isolated copy
│   └── agent-2-branch/  ← Agent 2's isolated copy
└── (main working tree)  ← First agent's working copy
```

### Pentagon Directory Access

All agents can access `~/.pentagon/` for identity files, inboxes, and coordination. This directory is outside git and shared across all agents:

```
~/.pentagon/
├── agents/{uuid}/       ← Per-agent identity and inbox
├── pods/{uuid}/         ← Pod-level shared files
├── teams/{uuid}/        ← Team-level shared files
├── hooks/               ← Scope guard hook
├── sessions/{uuid}/     ← Conversation transcripts
├── workspaces/{uuid}/   ← Workspace configurations
├── roles/               ← Organizational role definitions
├── a2a-audit.jsonl      ← Agent-to-agent audit log
└── canvas-layout.json   ← Persistent canvas state
```

---

## Key Takeaways

1. **Skills are drop-in capabilities.** Install a folder with a SKILL.md and the agent gets a new slash command.
2. **The scope guard enforces file isolation.** Agents can't write to each other's directories — only inbox delivery via Write is allowed.
3. **Use Write for inbox delivery.** It's the only tool that passes scope guard validation for cross-agent messaging.
4. **Skills don't pollute git.** Pentagon-injected skills are excluded from version control.
5. **Batch-install skills with simple loops.** One script can equip your entire team.
6. **Prefer dedicated tools over Bash.** Read, Write, Edit, Glob, Grep are more reliable and have better scope guard compatibility.

---

Next: [Chapter 8 — Common Pitfalls](08-common-pitfalls.md) — The failures that keep happening and how to avoid them.
