# Autonomous Development System — Setup Guide

A multi-agent Claude Code system that runs a continuous development loop inside a Discord bot. Agents take tasks from a backlog, build features, review PRs, and keep the queue stocked — all on cron schedules. You own the `main` branch and decide when to ship.

---

## Prerequisites

- [claude-code-discord-starter](https://github.com/anthropics/claude-code-discord-starter) set up and running
- A project repo (Rails, React, whatever — the prompts are customizable)
- Claude Code with Opus access (PM, Project Manager, and Product Manager use Opus; Developer, Reviewer, Vet Industry Researcher use Sonnet)

---

## The Agents

### Developer (every 10 min, at :00/:10/:20/:30/:40/:50)

Does the actual coding. Wake-up order is strict:

1. **PAUSE check** — if `research/agents/PAUSE` exists, log and exit
2. **DEV_LOCK check** — mutex file prevents concurrent runs (see below)
3. **Fix-up** — check for any Changes Requested PRs you authored; address feedback first (TRD feedback or code feedback)
4. **Resume In Progress** — check the TRD status field first: if `awaiting-review` exit (blocked on reviewer), if `changes-requested` update the TRD and re-submit, if `approved` proceed building
5. **In-flight cap** — if 1+ of your PRs is In Review, exit (reviewer needs to act first)
6. **Pick top Ready task** — take the first item from Ready in the backlog

**On a new task, the developer writes the TRD first, then waits.** It branches, writes a Technical Requirements Document (`research/plans/<branch>-trd.md`), commits it, opens a draft PR, marks the task TRD as `awaiting-review`, and exits. It does not write any feature code until the Reviewer approves the TRD.

Once the TRD is approved, it resumes and builds to completion — committing in small focused units, marking PR ready when done, moving task to In Review.

**Never merges anything.**

### TRD Watcher (every 5 min, offset at :02)

Dedicated architectural reviewer for TRDs. Runs fast, exits silently when nothing is pending.

- Scans `In Progress` and `In Review` tasks for any with `TRD: ... — awaiting-review`
- Checks PR comments first — if it already reviewed this TRD in a prior run, exits silently (dedup)
- Reviews the TRD as an **architectural contract**: do the listed components cover the PRD? Does the approach fit VetWare patterns? Any scope creep or scalability red flags in the component design?
- **Does NOT review implementation details** — function signatures, algorithms, and exact code structure belong in the actual development, not the TRD
- **This is a real review** — will request changes on missing components, scope creep, bad architectural choices, or absent test coverage plans
- Approves by updating the TRD field in `backlog.md` + PR comment; requests changes by moving task to `Changes Requested`
- No test runs (that's the Reviewer's job). One TRD per run, then exits.
- Silent exit if nothing is pending — zero noise on no-op runs

### Reviewer (every 10 min, offset at :05/:15/:25/:35/:45/:55)

Code review only — TRDs are handled by the TRD Watcher.

- Picks the oldest PR in the In Review backlog section
- Checks out the branch and **runs the tests itself** — never trusts the developer's claim
- Reviews against your standards memories (backend patterns, frontend patterns, DRY, scope)
- Also verifies the implementation matches the approved TRD — if the Developer deviated from the architecture without explanation, that's grounds for changes-requested
- High bar: default is request changes. Must be able to name exactly which standards it checked before approving.
- On approve: moves task to Approved, then **writes/appends a task section to `research/goals/goal-NN-short-title.md`** — what shipped, key technical decisions, codebase areas touched, reviewer notes. First approval for a goal creates the file; subsequent tasks append new sections.
- **You merge to main — reviewer never touches main.**

### Project Manager (every 30 min, at :02/:32)

Keeps the queue stocked so the developer never idles.

- Reads the implementation roadmap (your source of truth for what to build)
- Keeps 2–3 tasks in Ready at all times
- **Never creates a task for a goal without a PRD** — checks `research/agents/prds/` first; if no PRD exists for the goal, logs the gap and waits for the Product Manager to write it
- Every task must trace to a goal in the roadmap — no invented work
- Off-roadmap ideas go to `proposals.md` for you to review
- Handles grooming: prioritization, closing stale tasks, flagging blocked work

### Product Manager (every 4 hours)

PRD writing is the primary job. Checks the next 2 upcoming goals in the roadmap — for any that lack a PRD in `research/agents/prds/`, writes one. PRDs define user stories, UX flows, success metrics, and scope boundaries.

Uses `product-notes.md` from the Vet Industry Researcher as its main research source. If the Researcher hasn't covered something needed for a solid PRD (a new goal area, compliance specifics, etc.), the Product Manager does the research itself — a weak PRD is worse than the extra time spent digging.

Runs every 4 hours so PRDs stay ahead of where the developer is working. Never touches the backlog or writes code.

### Vet Industry Researcher (daily at 7am)

Market intelligence. Researches one topic per run in the US vet PIMS landscape — competitor moves, user pain points, pricing models, compliance, integrations — and appends findings to `product-notes.md`. These findings feed the Product Manager's PRD writing.

Runs daily (research doesn't need to be more frequent). Uses Sonnet since it's search + summarization. Never writes PRDs, never touches the backlog.

### Merge Watcher (every 5 min)

Lightweight unblocking helper.

- Checks `git log origin/main --since='6 minutes ago'`
- If a merge happened: finds Blocked tasks whose dependency just shipped, moves them to Ready
- Posts to Discord if tasks moved; **silent exit if nothing changed**
- Does not create tasks, does not touch scope

### System Reviewer (daily at 9pm)

The system's self-improvement loop. Audits the last 24 hours of agent logs, scores how well each part of the pipeline is working, and files concrete improvement proposals.

- Scores 6 dimensions 1–5: developer throughput, TRD review speed, code review quality, backlog health, PRD coverage, overall
- Every score below 4 requires specific evidence from the log (timestamps, task IDs, entry content)
- Files proposals to `proposals.md` with impact/effort ratings — only for problems with clear evidence
- Appends a dated scorecard to `system-health.md` (persistent record you can track over time)
- Posts a brief summary to Discord at 9pm each night

This is how the system gets better without you manually reading every log entry.

### Log Trim (every 6 hours)

Housekeeping. Archives agent log entries older than 48h so the active log stays small and agents don't burn tokens re-reading history.

---

## The Backlog

One file: `research/agents/backlog.md`. Tasks flow through sections:

```
Ready → In Progress (TRD phase) → In Progress (build phase) → In Review → Approved → Shipped
                   ↘ TRD Changes Requested → (back to In Progress)
                                                            ↘ Changes Requested → (back to In Review)
Blocked  (waiting on dependency or you)
```

The TRD phase is new: Developer writes the Technical Requirements Document, commits it, and waits for Reviewer approval before writing any code.

**Who moves what:**
- Product Manager writes PRDs into `research/agents/prds/`
- PM creates tasks (only if a PRD exists) and reorders Ready
- Developer moves Ready → In Progress, fills in TRD status
- Reviewer moves TRD from `awaiting-review` → `approved` or `changes-requested`
- Developer (after TRD approval) builds and moves In Progress → In Review
- Reviewer moves In Review → Changes Requested or Approved
- You move Approved → Shipped after merging

### Task format

```markdown
### TASK-NNNN: short title
- **Goal:** Goal N from your roadmap (cite section)
- **PRD:** research/agents/prds/goal-NN-short-title.md
- **Scope:** what's in, what's NOT in
- **Acceptance:** bullet list of testable criteria
- **PR:** (filled by Developer)
- **Branch:** (filled by Developer)
- **TRD:** (filled by Developer — path + status: awaiting-review / changes-requested: reason / approved)
- **Notes:** anything reviewer should know
```

Task IDs are monotonic. PM picks the next number.

---

## The DEV_LOCK Mutex

Prevents two developer instances from running at the same time.

```bash
# Claim at wake-up
echo "$(date -u +%Y-%m-%dT%H:%M:%SZ) TASK-NNNN" > research/agents/DEV_LOCK

# Always release before exit — success, error, or early exit
rm research/agents/DEV_LOCK
```

Rules:
- Lock is < 25 min old → another run is active, exit immediately
- Lock is > 25 min old → prior run crashed, override it

---

## The Kill Switch

```bash
touch research/agents/PAUSE   # all agents exit immediately on wake-up
rm research/agents/PAUSE      # resume
```

Useful when you're doing something sensitive (big migration, manual refactor, etc.) and don't want agents touching the repo.

---

## Standards Memories

Agents read a set of memory files before every run. These define your coding standards — the reviewer enforces them, the developer follows them.

Suggested files in `memory/your-project/`:
- `feedback_backend_standards.md` — controller patterns, interactor shapes, query patterns
- `feedback_frontend_standards.md` — component conventions, hook rules, API module structure
- `feedback_separation_of_concerns.md` — what lives where
- `feedback_pull_requests.md` — PR policy (push early, draft PR immediately, etc.)
- `project_context.md` — current goal status, architecture decisions, key file index

The reviewer cross-checks every PR diff against these. The developer reads them before writing any code. Drift gets caught automatically.

---

## The Implementation Roadmap

A single file (`research/implementation-roadmap-v2.md` or similar) that defines every goal in dependency order. The PM treats this as law — every task it creates must trace to a goal here. If it spots something worth doing that isn't in the roadmap, it files a proposal for you instead of queueing the work.

This is the most important file in the system. Keep it updated as priorities shift.

---

## Cron Configuration

All jobs in `workspace/crons/jobs.json`:

```json
{
  "id": "project-agent-developer",
  "name": "Developer (every 10 min)",
  "schedule": "0,10,20,30,40,50 * * * *",
  "tz": "America/New_York",
  "enabled": true,
  "timeoutSeconds": 1500,
  "model": "sonnet",
  "discordChannel": "YOUR_CHANNEL_ID",
  "announceResult": true,
  "message": "You are the Developer agent. Read your instructions at ./research/agents/prompts/developer.md and follow them exactly. ..."
}
```

Cron schedules use standard 5-field crontab syntax. Models: `opus` for Product Manager and Project Manager (complex planning), `sonnet` for Developer, Reviewer, Vet Industry Researcher, Merge Watcher, Log Trim.

---

## The Watch Script

Run in a terminal to see live countdowns to each agent's next fire:

```bash
node workspace/scripts/agent-watch.js
```

Shows each agent name, time until next run, and a progress bar. Color-coded by urgency (red < 60s, yellow < 5min, normal otherwise). Sorted by soonest-to-fire. Refreshes every second.

To add/modify agents in the watch script, edit the `AGENTS` array at the top of `agent-watch.js` — name, schedule minutes, maxSecs, and color.

---

## Agent Prompt Files

Each agent reads its full instructions from a prompt file before acting:

```
research/agents/prompts/
  developer.md
  reviewer.md
  trd-watcher.md
  project-manager.md
  product-manager.md
  vet-industry-researcher.md
  system-reviewer.md
```

These are the most important tuning lever. When behavior is wrong, edit the prompt. Key things to include in each:

- **Wake-up checklist** — exact ordered steps
- **Hard rules** — explicit prohibitions (never merge to main, never expand scope, etc.)
- **Log format** — exactly how to write the agent-log entry
- **Discord summary format** — what to post when done

---

## Key Design Decisions

**PRD → TRD → Code is the mandatory sequence.** Nothing gets built without a PRD (product requirements) and an approved TRD (technical requirements). The Product Manager writes PRDs for upcoming goals proactively. The Developer writes a TRD before touching any code; the Reviewer approves it before the Developer builds. This catches architectural mistakes before they're baked into a PR.

**Developer works to completion, not in one-shot bursts.** The developer keeps building until the full task scope is done, committing each logical slice along the way. Good git history matters — watch commits in GitHub to see progress.

**TRD approval adds ~10 minutes to task start time.** Developer writes TRD and exits at :00. TRD Watcher fires at :02, reviews the TRD (no test run needed — it's a document review), approves or requests changes, exits. Developer fires at :10, sees TRD approved, starts building. The TRD Watcher is a dedicated fast-running cron separate from the Reviewer so a pending code review never delays a TRD approval.

**In-flow vs. In-Review distinction.** Draft PR = task is In Progress (TRD phase or build phase). Only mark PR ready + move to In Review when the full scope is done. This prevents the reviewer from blocking the developer mid-build.

**Reviewer never merges.** The reviewer approves PRDs (via backlog field update) and PRs; you merge to main. This keeps you as the final gate on what ships.

**Roadmap-traced tasks only, and PRD-gated.** The PM can't create tasks without a PRD. The PM can't invent strategy. Ideas not in the roadmap go to proposals.md for you to decide on.

**Standards memories over inline instructions.** Rather than embedding all the rules in every agent's cron message, agents read shared memory files. Update standards in one place; all agents pick them up on the next run.

---

## Operational Files

```
research/agents/
  backlog.md          ← task queue (single source of truth)
  agent-log.md        ← append-only audit log (agents read last 75 lines)
  agent-log-archive-YYYY-MM.md  ← archived entries >48h
  proposals.md        ← off-roadmap ideas for you to review
  product-notes.md    ← Vet Industry Researcher feed (read by Product Manager when writing PRDs)
  system-health.md    ← daily scorecards from System Reviewer (append-only)
  PAUSE               ← kill switch (touch to pause, rm to resume)
  DEV_PAUSE           ← auto-pause for Developer (written after 20 consecutive idle fires)
  REV_PAUSE           ← auto-pause for Reviewer (written after 20 consecutive idle fires)
  TRD_PAUSE           ← auto-pause for TRD Watcher (written after 20 consecutive idle fires)
  MW_PAUSE            ← auto-pause for Merge Watcher (written after 20 consecutive idle fires)
  DEV_IDLE            ← idle counter for Developer (reset on productive run)
  REV_IDLE            ← idle counter for Reviewer (reset on productive run)
  TRD_IDLE            ← idle counter for TRD Watcher (reset on productive run)
  DEV_LOCK            ← developer mutex
  prompts/            ← agent instruction files
  prds/               ← PRDs written by Product Manager (one per goal)
    goal-NN-short-title.md

research/goals/       ← goal summaries written by Reviewer on each approval (one file per goal)
  goal-NN-short-title.md   ← header + one ## TASK-NNNN section per approved task

research/plans/
  <branch-name>.md         ← Developer's operational plan (work breakdown)
  <branch-name>-trd.md     ← Technical Requirements Document (reviewed by TRD Watcher)
```

---

## Getting Started Checklist

- [ ] Set up claude-code-discord-starter and connect your Discord bot
- [ ] Create your implementation roadmap file
- [ ] Write your standards memory files
- [ ] Write the four agent prompt files (developer, reviewer, PM, product manager)
- [ ] Create `research/agents/prds/` directory
- [ ] Have the Product Manager write PRDs for the first 2–3 goals before enabling the Developer
- [ ] Create the initial backlog with 2–3 Ready tasks
- [ ] Configure `workspace/crons/jobs.json` with your channel ID and schedules
- [ ] Add agents to `agent-watch.js` so you can monitor them
- [ ] Run `node workspace/scripts/agent-watch.js` and watch the first few cycles
- [ ] Touch `PAUSE` before making manual changes to the repo; remove when done
