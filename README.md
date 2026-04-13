# Autonomous Dev Agents System

A multi-agent Claude Code system that runs a continuous development loop inside a Discord bot. Agents take tasks from a backlog, write PRDs, write TRDs, build features, review PRs, and keep the queue stocked — all on cron schedules.

**You own the `main` branch and decide when to ship.**

---

## How it works

```
PRD (Product Manager) → TRD (Developer) → Code (Developer) → Review (Reviewer) → Your merge
```

Nine agents, each with a single job:

| Agent | Schedule | Model | Job |
|-------|----------|-------|-----|
| Developer | every 10 min | Sonnet | Writes TRDs, builds features, pushes each commit incrementally |
| TRD Watcher | every 5 min | Sonnet | Reviews TRDs (architectural only) |
| Reviewer | every 10 min | Sonnet | Code reviews — only fires when new commits exist since last review |
| Project Manager | every 30 min | Opus | Keeps backlog stocked; re-checks roadmap health every other run |
| Product Manager | every 4h | Opus | Writes PRDs for upcoming goals |
| Domain Researcher | daily 7am | Sonnet | Research feed for PRDs |
| Merge Watcher | every 5 min | Sonnet | Unblocks tasks when deps merge; syncs main into open PR branches |
| System Reviewer | daily 9pm | Opus | Audits logs, scores the system, files proposals, syncs public repo |
| Log Trim | every 6h | Sonnet | Archives old log entries |

---

## Quick start

1. Set up [claude-code-discord-starter](https://github.com/anthropics/claude-code-discord-starter)
2. Read **[GUIDE.md](./GUIDE.md)** — full setup walkthrough
3. Copy the `prompts/` templates and adapt them to your project
4. Copy `crons/jobs.json` and set your Discord channel ID
5. Write your implementation roadmap and initial backlog
6. Run `node workspace/scripts/agent-watch.js` to monitor agents

---

## Repo structure

```
prompts/              ← agent instruction files (customize these)
  developer.md
  reviewer.md
  trd-watcher.md
  project-manager.md
  product-manager.md
  domain-researcher.md
  system-reviewer.md

crons/
  jobs.json           ← cron job definitions (plug into claude-code-discord-starter)

GUIDE.md              ← full setup and design guide
```

---

## Requirements

- [claude-code-discord-starter](https://github.com/anthropics/claude-code-discord-starter) running
- Claude Code with Opus access (PM, Project Manager, System Reviewer use Opus)
- A project repo (any stack — prompts are fully customizable)

---

Built by [@keithbooher](https://github.com/keithbooher). See [GUIDE.md](./GUIDE.md) for the full story.
