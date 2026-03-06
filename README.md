# Pentagon: The Power User's Guide

**Mission control for your AI team — operated at scale.**

---

## What is Pentagon?

[Pentagon](https://pentagon.run) is a native macOS application that lets you manage multiple AI agents from a single screen. Each agent runs as an independent Claude Code process with its own identity, memory, tasks, and map. Pentagon gives you a spatial canvas where every agent appears as a desk — you can see who's active, who's idle, who needs help, and what everyone is working on, all at a glance.

Think of it as mission control: you spawn agents, assign them roles and projects, organize them into teams and roles, and watch them work in parallel. Pentagon handles process isolation (via git worktrees), file isolation (via scope guard), status monitoring, and lifecycle management (via heartbeat) so you can focus on directing the work.

Pentagon is actively developed by [Dark Research](https://darkresearch.ai), with frequent releases focused on performance, reliability, and developer experience. The project has a growing community of developers running multi-agent teams in production.

## Why This Guide?

Pentagon's [official docs](https://docs.darkresearch.ai/pentagon) cover installation and features. This guide covers what happens *after* setup — the operational patterns, failure modes, and coordination strategies that emerge when you actually run multi-agent teams. It's the difference between knowing what buttons to press and knowing how to run a team effectively.

Whether you're about to spawn your first agent or scaling to 10+, these patterns will save you from the mistakes that every multi-agent operator hits eventually.

## What You'll Learn

- **How identity files work** — SOUL.md, MEMORY.md, PURPOSE.md, tasks.json, report.json, and why each one matters
- **The #1 communication mistake** — and the one rule that prevents it
- **Team hierarchy patterns** — from 3 agents to 15+, with the roles that scale
- **Agent lifecycle management** — heartbeat, dormancy, respawning, and context exhaustion
- **The auditor pattern** — independent monitoring that catches what the hierarchy misses
- **Operational recipes** — step-by-step playbooks for common scenarios

## Who This Is For

Technical developers who want to get the most out of Pentagon. Whether you just installed it or you've been running agents for weeks, this guide gives you the patterns to operate effectively — from your first agent to a fleet of 15+.

## Community

Pentagon has an active community on Telegram where the developers and users hang out. **@pentagonpenty** — Pentagon's CX bot — is available in the group for real-time questions about features, configuration, and troubleshooting. For bugs and feature requests, the Discord and GitHub channels are your best bet.

## What's Coming

Pentagon is under active development. Features on the near-term roadmap:

- **Agent-specific configs** — Per-agent Claude configuration (currently config is at the machine level)
- **Agent sandboxes** — Local process isolation for agents, improving security and robustness
- **Hosted agents** — Cloud-hosted agents that run independently of your local machine
- **Multiplayer** — Multi-user collaboration on the same Pentagon instance
- **Multi-provider support** — Native support for non-Claude models (Kimi now supported — cross-provider sessions are live, with session sanitation to prevent corruption)

This guide will be updated as these features ship.

## How to Read This

Each chapter stands alone. Read them in order for the full picture, or jump to what you need:

| Chapter | Topic | When You Need It |
|---------|-------|-----------------|
| [1. Agent Identity](01-agent-identity.md) | The files that define an agent | Setting up any agent |
| [2. Communication](02-communication.md) | Messaging, the #1 rule, verification | Running 2+ agents |
| [3. Hierarchy Design](03-hierarchy-design.md) | Team structure, teams & roles, the no-code-manager rule | Running 5+ agents |
| [4. Agent Lifecycle](04-lifecycle.md) | Heartbeat, dormancy, respawning | Managing agent uptime |
| [5. The Auditor Pattern](05-auditor-pattern.md) | Independent monitoring and gap detection | Running 5+ agents |
| [6. Report Discipline](06-report-discipline.md) | Making the canvas useful | Always |
| [7. Skills and Tools](07-skills-and-tools.md) | Custom skills, hooks, tool patterns | Extending capabilities |
| [8. Common Pitfalls](08-common-pitfalls.md) | Failures and how to prevent them | Before scaling up |
| [9. Operational Playbook](09-operational-playbook.md) | Step-by-step recipes | When you need a how-to |
| [10. Troubleshooting](10-troubleshooting.md) | Common errors and fixes | When something breaks |

---

*By Fawaz B — [@Fwazsol](https://twitter.com/Fwazsol) · [GitHub](https://github.com/FwazB)*
