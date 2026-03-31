# Skills and Tools — Extending Agent Capabilities

> *A skill is a slash command that gives an agent a new capability without rewriting its soul. Drop a folder in the right directory and every agent on the team gets the power.*

Pentagon agents run on Claude Code, which means they inherit all of Claude Code's tool capabilities — file I/O, Bash, search, web access, and more. Pentagon adds layers on top: **skills** (custom slash commands), **MCP tools** (server-side capabilities via the Model Context Protocol), **hooks** (lifecycle event scripts), and **approval gates** (human sign-off guardrails).

---

## MCP Tools

### What Are MCP Tools?

Pentagon exposes server-side capabilities to agents via the Model Context Protocol (MCP). These tools are available to every agent without any installation — they're provided by the Pentagon server.

### Core MCP Tools

| Tool | Purpose |
|------|---------|
| `send_message` | Send a message to a conversation or channel |
| `find_conversation` | Find or create a conversation with specific participants |
| `read_messages` | Read messages from a conversation |
| `patch_document` | Update agent documents (report, memory, soul, purpose) |
| `read_document` | Read an agent's documents |
| `read_tasks` | Read an agent's task list |
| `write_tasks` | Update an agent's task list |
| `get_org_context` | Get organizational context — teams, teammates, map info |
| `spawn_agent` | Programmatically spawn a new agent |
| `token_info` | Check token/usage information |

### MCP Tools vs. Skills

| | MCP Tools | Skills |
|---|---|---|
| **Source** | Pentagon server | Filesystem (SKILL.md files) |
| **Installation** | Automatic — available to all agents | Manual — drop into skills directory |
| **Invocation** | Called directly as tools | Invoked via `/skill-name` slash command |
| **Capability** | Communication, document management, spawning | Custom workflows, deployment, reviews |
| **Customizable** | No — Pentagon defines them | Yes — you write them |

MCP tools handle Pentagon-native operations (messaging, documents, spawning). Skills handle custom project-specific workflows.

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

### Built-In Pentagon Skills

Pentagon ships with several built-in skills:

| Skill | Purpose |
|-------|---------|
| `/manage-pentagon-agent` | Spawn and configure agents |
| `/manage-pentagon-team` | Create and manage teams |
| `/manage-pentagon-canvas` | Arrange agents on the canvas |
| `/manage-pentagon-role` | Create organizational roles |

On v1.3+, communication skills (`/send-pentagon-message`) are superseded by MCP tools (`send_message`, `find_conversation`). The MCP tools are the recommended communication path.

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
5. Report result to the deploying manager via send_message

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

## Approval Gates

v1.3 introduces **approval gates** — guardrails that allow agents to flag decisions requiring human authorization before proceeding.

When an agent reaches a decision point covered by a gate, it pauses and requests human sign-off before proceeding. Configuration details may vary — consult Pentagon's documentation for current setup options.

**Use cases:**
- Production deployments
- Destructive operations (database migrations, branch deletions)
- External communications (sending emails, posting to services)
- Cost-sensitive operations (large API calls, cloud resource provisioning)

Approval gates complement the patterns in [Chapter 9 — Operational Playbook](09-operational-playbook.md) where human-in-the-loop is recommended for irreversible actions. Gates formalize this — instead of relying on SOUL.md instructions to "ask before pushing," the gate enforces it at the system level.

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
| **Read** | Reading files | Reading other agents' reports, configs |
| **Write** | Creating/overwriting files | Writing to agent inboxes (legacy message delivery) |
| **Edit** | Modifying files | Updating SOUL.md, MEMORY.md, PURPOSE.md, tasks.json |
| **Glob** | Finding files by pattern | Scanning agent directories |
| **Grep** | Searching file contents | Finding references to UUIDs, agent names |

### MCP Tools for Cross-Agent Operations

On v1.3+, use MCP tools for all cross-agent communication:

- `send_message` — Send messages to conversations/channels
- `find_conversation` — Look up or create conversation with participants
- `read_messages` — Read conversation history
- `patch_document` — Update your own report, memory, soul, or purpose

The Write tool remains valid for inbox delivery (v1.2 legacy), but MCP tools are the recommended path.

### The Bash Tool: Use Carefully

The Bash tool has scope guard constraints:

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

## Map and Directory Access

### The Agent's Working Directory

Each agent is assigned a project directory at spawn time. This is where it operates — reading code, writing files, running commands. Pentagon sets this via the config at spawn. v1.3+ supports multi-folder workspaces, allowing agents to work across entire project structures.

### Isolation Mode (v1.3+)

When isolation is enabled, each agent gets a fully independent clone of its repositories:

```
Agent A → /path/to/clone-a/  (independent git repo)
Agent B → /path/to/clone-b/  (independent git repo)
```

Agents interact with their clones like human engineers on a team — branching, committing, pushing, pulling through standard git.

### Legacy: Git Worktree Isolation (v1.2)

On v1.2, agents on the same repo got isolated worktrees managed by Pentagon:

```
~/.pentagon/worktrees/
└── {repo}-{hash}/
    ├── feature-auth/    ← Agent 1's isolated copy
    └── feature-ui/      ← Agent 2's isolated copy
```

Pentagon intercepted git commands via a wrapper at `~/.pentagon/bin/git`. On v1.3+, use isolation mode instead.

### Pentagon Directory Access

All agents can access `~/.pentagon/` for identity files and coordination. This directory is outside git and shared across all agents:

```
~/.pentagon/
├── agents/{uuid}/       ← Per-agent identity and inbox
├── teams/{uuid}/        ← Team-level shared files
├── roles/{uuid}/        ← Role-level shared files
├── hooks/               ← Scope guard hook
├── sessions/{uuid}/     ← Conversation transcripts
├── workspaces/{uuid}/   ← Map configurations
├── a2a-audit.jsonl      ← Agent-to-agent audit log
├── perf-log.jsonl       ← Performance metrics (CPU, I/O, timing)
└── canvas-layout.json   ← Persistent canvas state
```

---

## Security Considerations

Pentagon's security model is designed for trusted local environments — teams where all agents are under one operator's control. Understanding its boundaries helps you make informed decisions about what to trust.

### What the Scope Guard Protects

The scope guard provides **file-level write isolation** between agents. It prevents agents from modifying each other's identity files, overwriting each other's reports, or tampering with delivered messages. Inbox delivery is allowed via the Write tool with sender validation (`from.id` must match the agent's actual UUID).

This is effective for preventing accidental cross-contamination — agents stepping on each other's files by mistake. It's a guardrail, not a security perimeter.

### What It Doesn't Protect

**Read access is unrestricted.** Any agent can read any other agent's files — MEMORY.md, tasks.json, report.json, inbox messages, everything. In practice, agents need this for coordination (auditors read reports, managers read task boards). But it means agent knowledge is shared knowledge. Never store actual secrets — API keys, tokens, credentials — in agent files. Use environment variables for secrets.

**Message provenance is hook-validated, not cryptographically signed.** The scope guard validates that `from.id` matches the sender's `PENTAGON_AGENT_ID` environment variable. This catches casual spoofing but isn't cryptographic verification. For high-stakes decisions (deploy approvals, access grants), use approval gates or require human confirmation.

**Messages are read by an LLM.** Agent-to-agent messages become part of the receiving agent's context. A crafted message could contain instructions disguised as content — the same class of risk as prompt injection from any untrusted input. For critical actions, require explicit human approval via approval gates rather than relying solely on peer agent authorization.

### Security Patterns That Work

| Pattern | Why It Helps |
|---------|-------------|
| **Approval gates** | System-enforced human sign-off for destructive or sensitive operations |
| **Read-only auditors** | Auditors that can't take destructive actions are safer recipients of untrusted input |
| **Separation of audit and execution** | The agent that finds issues shouldn't be the same agent that fixes them |
| **Multi-agent approval chains** | Production pushes require approval from both a security reviewer and a project lead |
| **Human-in-the-loop for destructive actions** | Agent consensus isn't sufficient for irreversible operations |
| **Secrets in env vars, not files** | Agent files are readable by all agents — environment variables are per-process |

### Domain Context for Auditors

Agents performing security or quality audits need the project's design philosophy in their SOUL.md or PURPOSE.md. Without it, they apply generic assumptions that produce false positives. An auditor reviewing a permissionless system with centralized assumptions will flag intentional design patterns as vulnerabilities. Give auditors the "why" before they evaluate the "what."

---

## Key Takeaways

1. **MCP tools are the primary interface for cross-agent operations on v1.3+.** `send_message`, `find_conversation`, `patch_document` — use these over filesystem-based approaches.
2. **Skills are drop-in capabilities.** Install a folder with a SKILL.md and the agent gets a new slash command.
3. **Approval gates enforce human sign-off at the system level.** Use them for production deployments, destructive operations, and anything irreversible.
4. **The scope guard enforces file isolation.** Agents can't write to each other's directories — only inbox delivery via Write is allowed.
5. **Skills don't pollute git.** Pentagon-injected skills are excluded from version control.
6. **Prefer dedicated tools over Bash.** Read, Write, Edit, Glob, Grep are more reliable and have better scope guard compatibility.
7. **Never store secrets in agent files.** All agents can read all files. Use environment variables.
8. **Require human confirmation for irreversible actions.** Approval gates or manual sign-off — agent-to-agent trust alone is not sufficient for production operations.

---

Next: [Chapter 8 — Common Pitfalls](08-common-pitfalls.md) — The failures that keep happening and how to avoid them.
