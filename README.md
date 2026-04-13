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
| Developer | every 10 min | Sonnet | Picks up Ready tasks; writes plan + TRD; builds feature once TRD approved; commits and pushes incrementally; opens/updates draft PR; moves task through backlog states |
| TRD Watcher | every 5 min | Sonnet | Scans In Progress + In Review for pending TRDs; reviews for architectural soundness, PRD coverage, scope, and test plan; approves or requests changes |
| Reviewer | every 10 min | Sonnet | Picks oldest In Review PR; skips if no new commits since last review; runs tests; reviews code against standards; approves or requests changes |
| Project Manager | every 30 min | Opus | Keeps Ready queue stocked at 2–3 tasks; closes stale tasks; surfaces off-plan ideas to proposals; re-checks roadmap health (sequencing, scope creep, gaps) every other run |
| Product Manager | every 4h | Opus | Writes PRDs for the next 2 upcoming goals that don't have one; updates existing PRDs with open question resolutions |
| Domain Researcher | daily 7am | Sonnet | Researches one market topic per run (competitors, user pain points, compliance, etc.); appends findings to product-notes.md; files proposals if warranted |
| Merge Watcher | every 5 min | Sonnet | Detects new main-branch merges; unblocks Blocked tasks whose dependencies just shipped; merges main into all open PR branches (alerts on conflicts) |
| System Reviewer | daily 9pm | Opus | Audits last 24h of agent logs; scores 7 dimensions; identifies problems with evidence; files proposals to proposals.md; updates system guide; syncs changes to public repo |
| Log Trim | every 6h | Sonnet | Archives agent-log.md entries older than 48h to a monthly archive file |

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
