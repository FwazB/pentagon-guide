# CLAUDE.md — Project Instructions in Practice

> *This is a real CLAUDE.md file. It serves as this guide's project instructions and as a working example of how to write one for your own Pentagon projects.*

A `CLAUDE.md` file is a Claude Code convention — a markdown file at the root of a project that Claude Code automatically loads into every agent's context when it starts working in that directory. It's how you give project-level instructions that persist across sessions and agents without putting them in every agent's SOUL.md.

For Pentagon projects, a good CLAUDE.md captures: coding conventions, terminology, version-specific behavior, and operational rules that every agent on the project should follow. What follows is this guide's actual CLAUDE.md — the same file that our agents read when working on the guide.

---

## Voice and Tone

### Community Guide — Not Official Documentation

This guide is **not affiliated with Arlington Labs or Pentagon**. We are community authors documenting patterns from hands-on experience.

**Never use language that implies insider access:**
- "being investigated," "the team is fixing," "we're developing," "we're working on"
- Internal developer names, @mentions, or references to private channels
- Promises about future features framed as insider knowledge

**Always:**
- State observations with version numbers: "has not been resolved as of v1.2.15"
- Attribute information to public sources: "per public release notes," "community members report"
- Frame discoveries as community knowledge: "community consensus is," "users have found that"

**Litmus test:** If a sentence sounds like it came from a Pentagon employee, rewrite it.

### Style Rules

- **No temporal language.** Don't write "now supports," "recently improved," "has been updated." Just state the fact. "Pentagon preserves session history through `/clear`" — not "Pentagon now preserves." The guide should read as a timeless reference, not a changelog.
- **No marketing copy.** Avoid vague superlatives — "incredibly powerful," "seamlessly," "blazing fast." Factual descriptions are fine ("adds minimal overhead" is OK if it's true; "unbelievably lightweight" is not). Describe what things do, not how impressive they are.
- **No filler.** Every sentence should earn its place. If you can cut a sentence without losing information, cut it.
- **Be specific.** Cite version numbers, file paths, and concrete examples. "v1.2.8 added injection verification" is better than "recent versions improved reliability."
- **Practitioner voice.** Write like an experienced operator teaching another operator. Direct, practical, no hand-holding — but not condescending.

## Chapter Structure

### Files

| File | Chapter | Section |
|------|---------|---------|
| `README.md` | Introduction | — |
| `01-agent-identity.md` | Agent Identity | Foundations |
| `02-communication.md` | Communication | Foundations |
| `03-hierarchy-design.md` | Hierarchy Design | Foundations |
| `04-lifecycle.md` | Agent Lifecycle | Operations |
| `05-auditor-pattern.md` | The Auditor Pattern | Operations |
| `06-report-discipline.md` | Report Discipline | Operations |
| `07-skills-and-tools.md` | Skills and Tools | Advanced |
| `08-common-pitfalls.md` | Common Pitfalls | Advanced |
| `09-operational-playbook.md` | Operational Playbook | Advanced |
| `10-troubleshooting.md` | Troubleshooting | Reference |
| `CLAUDE.md` | CLAUDE.md Example | Reference |
| `SUMMARY.md` | GitBook navigation | — |

### Conventions

- **SUMMARY.md** controls GitBook's sidebar navigation. Update it when adding chapters.
- **Chapter numbering:** Sequential integers. Use letter suffixes (e.g., `04b`) for chapters inserted between existing ones — don't renumber.
- **Each chapter stands alone.** Readers may jump directly to any chapter. Include enough context to understand without reading prior chapters, but link to them for depth.
- **Chapter anatomy:** Title with subtitle → epigraph → intro paragraph → sections → Key Takeaways → Next link.
  - **Epigraph:** Blockquote with italicized text. One sentence capturing the chapter's core insight. `> *Your insight here.*`
  - **Key Takeaways:** Numbered list, 5-8 items. Each item: **bolded first phrase** followed by explanation.
  - **Next link:** `Next: [Chapter N — Title](file.md) — One-line teaser.`
- **Pitfall format (ch08):** Each pitfall follows: **What happens** → **Impact** → **Root cause** → **Prevention** → **Detection** (optional).
- **Troubleshooting format (ch10):** Each issue follows: **Symptoms** → **Common causes** (numbered sub-causes, each with Diagnosis/Fix/Prevention) → summary fix.
- **Cross-references:** Use relative markdown links: `[Chapter 2](02-communication.md)`. For sections: `[Duplicate Injection](10-troubleshooting.md#duplicate-message-injection-message-flood)`.

## Pentagon Version Reference

Use these version notes when writing about features or fixes. Always cite the version where a behavior was introduced or changed.

| Version | Key Changes |
|---------|------------|
| v1.2.6 | Sessions survive `/clear` |
| v1.2.7 | Less aggressive session sanitation for cross-provider (Claude + Kimi) |
| v1.2.8 | Startup message flurry fix; programmatic injection verification with 30s retry window |
| v1.2.9 | `/send-pentagon-message` switched from `mv` to Write tool; `/manage-pentagon-worktree` skill; branch switching uses `--resume` |
| v1.2.12 | Multimap introduced — "maps" replace "workspaces"; map picker view (Figma-like); cross-map communication removed (hard boundary) |
| v1.2.13 | Team migration bug fixed |
| v1.2.15 | A2A reliability fixes; session management improvements |
| v1.3.0-beta.19 | Auth token refresh fix; spawn conversation ID fix; chat UI rebuilt; isolation mode introduced |
| v1.3.0-beta.20 | ~30 stability PRs; agents stopping randomly addressed; agent action visibility in DMs |
| v1.3.0-beta.21 | Message queueing (multi-message auto-sequencing); nearing stable graduation |
| v1.3.0-beta.25 | Agent persistence; communication stability; UI reconnection monitoring after sleep; channel-based A2A; isolation refinements |
| v1.3.4 | Stable graduation from beta; all beta features (channels, isolation, persistence, message queueing) ship as stable; approval gates; multi-folder workspaces |

### Terminology

| Term | Usage |
|------|-------|
| **Map** | The top-level organizational boundary (formerly "workspace" pre-v1.2.12). Use "map" in all new content. |
| **Team** | Colored rectangular region on the canvas. Spatial grouping. |
| **Role** | Sidebar-only logical grouping. Cross-cutting. |
| **Scope guard** | Pentagon's PreToolUse hook enforcing file isolation between agents. |
| **Heartbeat** | The system controlling agent restart behavior (Manual, Always On, Interval, Scheduled). |
| **Injection** | How Pentagon delivers messages into an agent's Claude Code conversation via TUI stdin. |
| **Isolation** | v1.3 opt-in setting replacing worktrees — gives each agent a full independent repo clone. Off by default. |
| **Channel** | v1.3 slack-like conversation space for agent communication, replacing inbox-based file messaging. |
| **Approval gate** | v1.3 guardrail allowing agents to flag decisions requiring human sign-off before proceeding. |
| **Multi-folder workspace** | v1.3 map-level feature — a single map supports agents working across entire project structures, not just one directory. |
| **MCP tools** | Pentagon's server-side tools (`send_message`, `find_conversation`, `read_messages`, `patch_document`, etc.) available to agents via the Model Context Protocol. Primary communication method on v1.3+. |

**Note:** The `workspace` field still appears in Pentagon's JSON files (`directory.json`, spawn requests). This is the actual field name in the data — don't rename it in code examples or JSON samples. Only update the prose term.

## Workflow

### How This Guide Is Maintained

This guide is maintained by two Pentagon agents:

1. **Writer agent** plans changes, writes chapters, handles rewrites, commits, and pushes
2. **Verification-editor agent** independently fact-checks claims, audits voice rule compliance, and flags issues

The writer drafts content and sends it to the verification-editor for review before presenting to the human. The verification-editor checks every behavioral claim against version citations, audits for temporal language and marketing copy, verifies cross-references, and confirms structural compliance with this CLAUDE.md.

This is a practical example of the auditor-patcher pattern from [Chapter 9](09-operational-playbook.md), adapted for documentation work.

### Git Conventions

- **Commits:** Descriptive one-line summaries. Include scope when touching multiple files: "Quality pass — remove temporal language across 9 files"
- **Push:** Only after review. Never force-push.

## Updating the Guide

When Pentagon releases a new version:

1. **Identify what changed** — read release notes or changelog
2. **Plan affected chapters** — which chapters reference the changed behavior?
3. **Update with version specificity** — "v1.2.X introduced Y" not "Pentagon recently added Y"
4. **Check for terminology changes** — has the UI renamed any concepts?
5. **Update the version reference table** in this file
6. **Review the full diff** before committing — ensure community voice rules are followed

---

Back to [README](README.md)
