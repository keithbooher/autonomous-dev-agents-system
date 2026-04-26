# Autonomous Development System — Complete Setup Guide

A multi-agent Claude Code system that runs a continuous development loop inside a Discord bot. Agents take tasks from a backlog, write architectural plans, build features, review PRs, and keep the queue stocked — all on cron schedules, around the clock, without you babysitting. You own the `main` branch and decide when to ship. The system catches its own mistakes through a TRD-then-code sequence, a dedicated code reviewer, a CI auto-fixer that patches broken tests, a nightly code auditor that files improvement tasks, and a nightly self-audit agent that files improvement proposals. You configure it once, tune it via prompt files, and watch it build your product.

> **Deployment:** This guide covers both local (laptop/Mac) and VPS (Linux server) deployments. VPS deployment is recommended for production use — it keeps agents running 24/7 without leaving your laptop on. See Part 1E for VPS-specific setup.

---

## Architecture Overview

```
Discord (your phone / browser)
    │
    ▼
claude-code-discord-starter (the harness)
    ├── Window 0: Claude Code session (VegaPunk / your AI assistant)
    ├── Window 1: cron-runner.js  ←── fires agents on schedule
    └── Window 2: discord-slash-handler.js

cron-runner reads workspace/crons/jobs.json
    │
    ├── Every 10 min: Developer agent ──────────────────────────────────────┐
    ├── Every 10 min (offset :05): Reviewer agent                           │
    ├── Every 5 min (offset :02): TRD Watcher                               │
    ├── Every 5 min: Merge Watcher                                           │
    ├── Every 30 min: Project Manager                                        │
    ├── Every 4 hours: Product Manager          All agents read/write:       │
    ├── Daily 7am: Industry Researcher          [your-project]/research/     │
    ├── Daily 9pm: System Reviewer              agents/backlog.md            │
    ├── Every 3 hours: Codebase Auditor         agents/agent-log.md  ───────┘
    ├── Every 4 hours: Log Trim
    ├── Every 2 min: Main CI Fixer  ← trigger-based, skips if no failure signal
    ├── Every 2 min: PR CI Fixer    ← trigger-based, skips if no failure signal
    └── Every 5 min: Stall Watcher  ← diagnoses & re-signals when a READY trigger file sits unprocessed >30 min

Your project repo (symlinked into workspace/)
    research/
      agents/
        backlog.md          ← task queue
        agent-log.md        ← audit log
        PAUSE               ← kill switch
        DEV_LOCK            ← developer mutex
        proposals.md        ← off-roadmap ideas
        product-notes.md    ← research feed
        system-health.md    ← nightly scorecards
        audit-state.md      ← tracks which code areas the Auditor has reviewed
        prompts/            ← agent instruction files
        prds/               ← product requirement docs
      plans/                ← TRDs and work breakdowns
      goals/                ← reviewer-written approval summaries
      implementation-roadmap.md   ← source of truth for what to build

/tmp/
    [your-project]-main-ci-failed    ← written by GitHub Actions on main failure
    [your-project]-pr-ci-failed-NNN  ← written by GitHub Actions on PR failure

workspace/memory/
    knowledge.db            ← SQLite + Gemini semantic memory
    remember.js / recall.js ← memory interface
```

**Task flow:**

```
Ready → In Progress (TRD phase) → In Progress (build phase) → In Review → Approved → Shipped
             ↘ TRD Changes Requested ──────────────────────────────────────↗
                                                  ↘ Changes Requested ─────↗
Blocked  (waiting on dependency or manual input)
```

---

## Prerequisites

- **macOS or Linux** (the startup script uses tmux and bash; a VPS running Ubuntu works great)
- **Node.js** 18+ (`node --version`)
- **tmux** (`brew install tmux` on macOS; `apt install tmux` on Ubuntu)
- **Claude Code** installed and authenticated (`claude --version`; run `claude` once to authenticate with OAuth)
- **GitHub CLI** (`gh`) authenticated — agents use `gh pr create`, `gh pr review`, etc.
- **A Discord account** with permission to create a bot application
- **A project repo** on GitHub — Rails, React, whatever; the agent prompts are fully customizable
- **A Gemini API key** (free tier is fine) — only needed for the SQLite semantic memory system; optional if you skip that feature
- **git** configured with your identity
- **(VPS only) A self-hosted GitHub Actions runner** — required if you want the CI auto-fixers. Without it, the CI runs remotely and the trigger files are never written. See Part 1E.

---

## Part 1: Base Infrastructure Setup

### Step 1A: Create your Discord bot

1. Go to [discord.com/developers/applications](https://discord.com/developers/applications) and click **New Application**.
2. Name it (e.g. "VegaPunk" or whatever you want the bot called).
3. In the left sidebar, click **Bot**.
   - Click **Reset Token** and copy the token — this is your `DISCORD_BOT_TOKEN`. Save it somewhere safe; you only see it once.
   - Under **Privileged Gateway Intents**, enable **Message Content Intent**. Without this the bot can't read messages.
   - Under **Privileged Gateway Intents**, also enable **Server Members Intent** (needed by some features).
4. In the left sidebar, click **OAuth2 → URL Generator**.
   - Under **Scopes**, check `bot` and `applications.commands`.
   - Under **Bot Permissions**, check: `Send Messages`, `Read Messages/View Channels`, `Read Message History`, `Add Reactions`, `Embed Links`.
   - Copy the generated URL, open it in your browser, and add the bot to your Discord server.
5. Go back to **General Information** and copy the **Application ID** — this is your `DISCORD_APP_ID`.
6. In Discord, right-click the channel you want the bot to post in and click **Copy Channel ID**. You'll need this for cron jobs.

> Tip: enable Developer Mode in Discord settings (User Settings → Advanced → Developer Mode) to get the right-click "Copy ID" option.

### Step 1B: Clone and configure claude-code-discord-starter

```bash
git clone https://github.com/anthropics/claude-code-discord-starter.git
cd claude-code-discord-starter
```

Create your env file:

```bash
cp config/.env.example config/.env  # or create it fresh
```

Edit `config/.env` with your values:

```bash
# ─── Discord ──────────────────────────────────────────────────────────────────
DISCORD_BOT_TOKEN=your_bot_token_here
DISCORD_APP_ID=your_application_id_here

# ─── Cron timezone ────────────────────────────────────────────────────────────
# All cron schedules in jobs.json use this timezone (IANA format)
CRON_TZ=America/New_York

# ─── Gemini (for semantic memory — optional) ──────────────────────────────────
# Get a free key at aistudio.google.com
GEMINI_API_KEY=your_gemini_key_here

# ─── Do NOT set ANTHROPIC_API_KEY ─────────────────────────────────────────────
# The system uses Claude Code OAuth (your ~/.claude.json), not the API key.
# Setting ANTHROPIC_API_KEY will override OAuth and bill your API account.
```

**What each variable does:**
- `DISCORD_BOT_TOKEN` — authenticates the bot with Discord's gateway. Required.
- `DISCORD_APP_ID` — tells the slash command handler which application to register commands under. Required.
- `CRON_TZ` — all cron job schedules are interpreted in this timezone. Agents log in this timezone too. Set to wherever you live.
- `GEMINI_API_KEY` — used by `remember.js` and `recall.js` to generate semantic embeddings. Only needed if you use the SQLite memory system.

### Step 1C: Set up your project repo in workspace

The harness expects your project repo to live (or be symlinked) inside `workspace/`. The easiest approach:

```bash
# Option A: symlink an existing repo
ln -s /path/to/your-project workspace/your-project

# Option B: clone directly into workspace
cd workspace && git clone https://github.com/you/your-project.git
```

All agent cron messages and prompts will reference paths like `./your-project/research/agents/...`. Adjust accordingly.

### Step 1D: Run the system with start-local.sh

```bash
bash start-local.sh
```

This starts three tmux windows inside a session named `claude`:

- **Window 0 (claude)** — the main Claude Code session. This is the "VegaPunk" brain that reads your Discord messages and responds. It runs in a restart loop so it recovers from crashes.
- **Window 1 (cron)** — `cron-runner.js`, which reads `workspace/crons/jobs.json` and fires each agent on schedule by spawning a fresh Claude Code process per job. Logs go to `workspace/crons/logs/cron-runner.log`.
- **Window 2 (slash)** — `discord-slash-handler.js`, which listens for `/slash` commands (like `/dev`, `/reviewer`, `/pm`) that trigger agents on demand.

**Useful tmux commands:**

```bash
tmux attach -t claude:0      # watch the main Claude session
tmux attach -t claude:cron   # watch the cron runner
tmux attach -t claude:slash  # watch the slash handler
# Ctrl-B then D to detach without stopping anything
tmux kill-session -t claude  # stop everything
```

### Step 1E: VPS deployment (optional but recommended)

Running on a VPS keeps agents alive 24/7 without tying up your laptop. Any Linux VPS works — a $20–40/month instance with 4GB RAM is plenty for one project with CI.

**Recommended setup:**
1. Provision an Ubuntu 22.04+ VPS (DigitalOcean, Hetzner, Linode, etc.)
2. Install prerequisites: `apt install tmux git nodejs npm` + install `gh` CLI + authenticate Claude Code
3. Clone the discord starter and your project repo onto the VPS
4. Copy your `config/.env` to the VPS
5. Run `bash start-local.sh` inside a persistent tmux session — it survives SSH disconnects

**Connecting from your laptop:**
- SSH into the VPS and run `tmux attach -t claude` to watch the running system
- Or just use Discord — that's the whole point

**Self-hosted GitHub Actions runner (required for CI auto-fixers):**

The Main CI Fixer and PR CI Fixer agents work by reading trigger files written to `/tmp/` when CI fails. These files are written by a GitHub Actions job that runs on a `self-hosted` runner — meaning the runner must be on the same machine as your agent system.

Setup (on the VPS):
```bash
# Follow GitHub's runner setup instructions:
# repo → Settings → Actions → Runners → New self-hosted runner → Linux
# Then install and configure the runner at e.g. /home/runner/actions-runner/
./config.sh --url https://github.com/[you]/[your-project] --token [TOKEN]
./run.sh &   # or install as a service with ./svc.sh install && ./svc.sh start
```

Then in your `.github/workflows/ci.yml`, add a job that fires only on `main` failures and writes the trigger file:

```yaml
fix-main-on-failure:
  needs: test
  if: failure() && github.ref == 'refs/heads/main'
  runs-on: self-hosted
  steps:
    - name: Signal CI failure to local agent
      run: |
        echo "${{ github.run_id }} ${{ github.sha }}" > /tmp/[your-project]-main-ci-failed
```

For PR CI failures, add a separate job:

```yaml
signal-pr-failure:
  needs: test
  if: failure() && github.event_name == 'pull_request'
  runs-on: self-hosted
  steps:
    - name: Signal PR CI failure to local agent
      run: |
        echo "${{ github.event.pull_request.number }} ${{ github.head_ref }} ${{ github.run_id }}" \
          > /tmp/[your-project]-pr-ci-failed-${{ github.event.pull_request.number }}
```

The CI Fixer agents check for these files every 2 minutes. If found, they investigate the failure, fix it, and push. If not found, the preCommand skips the Claude spawn entirely (zero token cost).

**Kanban Board:**

The harness includes an optional web-based Kanban dashboard (`workspace/kanban/server.js`). It serves a visual board you can view in a browser, driven by a local `kanban-board.md` file in your project. Agents don't write to it directly — you use it as a human-readable status view.

If you're running on a VPS, you can expose the Kanban server on a local port and access it via SSH tunnel:

```bash
# On VPS: start the kanban server (already part of the cron system)
node workspace/kanban/server.js

# On laptop: tunnel to it
ssh -L 3001:localhost:3001 [your-vps-ip]
# Then open http://localhost:3001 in your browser
```

For Mac users who want a live-updating board: the `start-fswatch.sh` script watches `kanban-board.md` locally and syncs it to the VPS via `scp` on every save. This lets you edit the board on your Mac and see it update on the VPS in real time.

---

## Part 2: Your Project Repo Setup

### Step 2A: Directory structure to create

Inside your project repo, create this structure before enabling agents:

```
your-project/
  research/
    agents/
      backlog.md                   ← task queue (you create the initial version)
      agent-log.md                 ← append-only audit log (start with a header comment)
      proposals.md                 ← off-roadmap ideas (start empty)
      product-notes.md             ← research feed (start empty)
      system-health.md             ← nightly scorecards (start with a header)
      PAUSE                        ← kill switch (touch to create, rm to remove)
      prompts/
        developer.md
        reviewer.md
        trd-watcher.md
        project-manager.md
        product-manager.md
        industry-researcher.md
        merge-watcher.md
        system-reviewer.md
        codebase-auditor.md
        main-ci-fixer.md
        pr-ci-fixer.md
      prds/                        ← product requirement docs, one per goal
    plans/                         ← TRDs and work breakdowns (developer writes these)
    goals/                         ← reviewer-written goal summaries (auto-created)
    implementation-roadmap.md      ← your roadmap (you write and maintain this)
```

Create the directories and stub files:

```bash
mkdir -p research/agents/prompts research/agents/prds research/plans research/goals

# Stub files
echo "# agent-log.md — append-only. Newest entries at the bottom." > research/agents/agent-log.md
echo "# proposals.md" > research/agents/proposals.md
echo "# product-notes.md" > research/agents/product-notes.md
echo "# system-health.md — nightly scorecards" > research/agents/system-health.md
```

Add the state files to `.gitignore` — they're local-only operational files that should never be committed:

```gitignore
# Agent state files — local only, never commit
research/agents/backlog.md
research/agents/agent-log.md
research/agents/agent-log-archive-*.md
research/agents/proposals.md
research/agents/product-notes.md
research/agents/velocity.md
research/agents/audit-state.md
research/agents/PAUSE
research/agents/DEV_LOCK
research/agents/REVIEWER_LOCK
research/agents/TRD_WATCHER_LOCK
research/agents/MERGE_WATCHER_LOCK
research/agents/PROJECT_MANAGER_LOCK
research/agents/PRODUCT_MANAGER_LOCK
research/agents/AUDITOR_LOCK
research/agents/AUDITOR_PAUSE
research/agents/SYSTEM_REVIEWER_LOCK
research/agents/*_PAUSE
research/agents/*_IDLE
research/agents/MW_DETECTED_MERGES
```

The `prompts/` directory and `prds/` directory ARE committed — they're configuration, not state.

### Step 2B: Writing the agent prompt files

Each agent reads its full instructions from a prompt file on every wake-up. These are the most powerful tuning lever in the system. When agent behavior is wrong, edit the prompt.

Every prompt file should contain these four sections:

1. **Identity and context** — who this agent is, what repo/stack it works on, where files live
2. **Read these first** — list of memory/context files to read before acting
3. **Wake-up checklist** — exact ordered steps with hard rules inline
4. **Log and Discord format** — exactly how to write entries

Below are skeleton templates. Replace `[your-project]` and `[your-stack]` throughout.

---

#### `prompts/developer.md` — skeleton

```markdown
# Developer Agent

You are the Developer in a multi-agent cron system working on [your-project].
You wake on a cron, claim a DEV_LOCK, and work until the current task is done.

## Read these before doing anything

1. `[your-project]/research/agents/backlog.md` — your task source
2. `memory/[your-project]-context/project_[your-project].md` — project context
3. `memory/[your-project]-context/feedback_backend_standards.md`
4. `memory/[your-project]-context/feedback_frontend_standards.md`
5. `memory/[your-project]-context/feedback_pull_requests.md`

## Environment

- Repo: `./[your-project]/`
- [your-stack setup commands — e.g. ruby version, db setup, test commands]
- Agent state files are gitignored — NEVER `git add` backlog.md, agent-log.md, proposals.md, velocity.md
- GitHub push auth: `cd [your-project] && git remote set-url origin "https://$(gh auth token)@github.com/[you]/[your-project].git"`

## Wake-up checklist

### 1. PAUSE check
If `[your-project]/research/agents/PAUSE` exists, log one line and exit.

### 2. DEV_LOCK check
Check `[your-project]/research/agents/DEV_LOCK`:
- Does not exist → proceed
- Exists and < 25 min old → another instance is running, log "DEV_LOCK held" and exit
- Exists and >= 25 min old → stale, override it

Claim the lock: `echo "$(date -u +%Y-%m-%dT%H:%M:%SZ) TASK-NNNN" > [your-project]/research/agents/DEV_LOCK`

Always release before every exit: `rm -f [your-project]/research/agents/DEV_LOCK`

### 3. Fix-up first (highest priority)
Check the `Changes Requested` section of backlog.md for any task with a PR you authored.
If found: check out the branch, read reviewer feedback, address only what was flagged (no opportunistic refactors), run tests, push, move task back to In Review. Release lock, log, post Discord summary, stop.

### 4. Resume In Progress
Check `In Progress` in backlog.md.
Read the task's TRD field:
- `awaiting-review` → TRD Watcher hasn't approved yet, exit (do not write feature code)
- `changes-requested: <reason>` → update TRD file, push, set field to `awaiting-review`, exit
- `approved` → proceed to build

When building: work until all acceptance criteria are done. Commit each logical slice and push immediately — do not batch. FINAL commit prefix: `FINAL: feat(scope): description`. When scope complete: `gh pr ready <num>`, move task to In Review.

### 5. In-flight cap
If any open PR corresponds to a task in `In Review` in backlog.md (count >= 1), exit. The reviewer must act before you start new work.

### 6. Pick the top Ready task
If backlog.md Ready section is empty, log "no ready tasks" and exit. Otherwise take the first task.

### 7. Start it: write the TRD first
Do NOT write any feature code.
- Branch from main: `git checkout main && git pull && git checkout -b <branch>`
- Check PRD field — if missing or file doesn't exist, move task to Blocked and exit
- Write operational plan at `research/plans/<branch>.md`
- Write TRD at `research/plans/<branch>-trd.md` (see TRD format below)
- Commit plan + TRD, push, open draft PR, set TRD field to `awaiting-review`, move task to In Progress, exit

### TRD format
The TRD is an architectural contract — what pieces are needed and why, not how they'll be coded.

Sections: What we're building | Technical components needed (new backend, new frontend, schema changes, API changes) | Key architectural decisions | Test coverage plan | Out of scope | Risks and open questions

### 8. Log format
```
## YYYY-MM-DD HH:MM ET DEVELOPER
- did: <one line>
- task: TASK-NNNN
- trd: written / awaiting-review / approved / building
- tests: green / red / skipped (reason)
- metrics: run_type=productive | commits=N | tests_added=N
- next: <what you'd do next run>
```
No-op: `metrics: run_type=no-op | reason=<brief>`

### 9. Discord summary
3–5 lines: what you did, PR link, TRD or test status, anything to watch.

## Hard rules
- Always release DEV_LOCK before exiting — no exceptions
- Never write feature code before TRD is approved
- Never merge anything
- Never branch from staging — always from main
- Never touch a task you didn't pick up in this run
- Never expand scope — if something is wrong, write it to proposals.md
- If anything feels wrong or unexpected, release lock, note it in proposals.md, and exit
```

---

#### `prompts/reviewer.md` — skeleton

```markdown
# Reviewer Agent

You are the Reviewer in a multi-agent cron system. You are the gate between the Developer and main.
Keith (the human) merges approved PRs — you never merge anything.
You hold a high bar. Your default is to request changes, not approve.
You do code reviews only. TRD reviews are handled by the TRD Watcher — do not check for pending TRDs.

## Read these before doing anything
1. `[your-project]/research/agents/backlog.md`
2. `memory/[your-project]-context/feedback_backend_standards.md`
3. `memory/[your-project]-context/feedback_frontend_standards.md`
4. `memory/[your-project]-context/feedback_separation_of_concerns.md`
5. `memory/[your-project]-context/feedback_pull_requests.md`

## Wake-up checklist

### 1. PAUSE check
If PAUSE exists, log and exit.

### 2. REVIEWER_LOCK check
Same mutex pattern as DEV_LOCK — 20-minute threshold, stale override.
Claim: `echo "$(date ...) PR-NN" > [your-project]/research/agents/REVIEWER_LOCK`
Release before every exit.

### 3. Find a PR to review
`gh pr list --state open --json number,title,isDraft,reviewDecision`
Pick the oldest PR that:
- Is in the In Review section of backlog.md
- Is NOT a draft (`gh pr view <num> --json isDraft`)
- Has new commits since your last review (dedup: compare lastReview vs lastCommit timestamps)

If nothing qualifies: log no-op, release lock, exit. Do NOT post to Discord on no-op.

### 4. Check CI first
`gh pr checks <num>`
- Failing → request changes citing which checks failed, stop
- Pending → skip this cycle (log `reason=ci-pending`), come back next run
- Passing → proceed

### 5. Read the PR
`gh pr diff <num>` — check all changed files. If ALL files are under `research/plans/`, this is TRD-only — skip it.
Read plan file, TRD, and related backlog task to understand scope and acceptance criteria.
Verify the implementation matches the approved TRD.

### 6. Review against standards
Check every relevant standards memory file. Be specific: cite the file, line, and rule.
Things to check: controllers stay thin, business logic in service objects, tests cover happy path + edge cases, no N+1 queries, no scope creep, Capybara/E2E specs for new UI flows, no duplication of existing utilities.

### 7. Decide
- Request changes: `gh pr review <num> --request-changes --body "..."` → move task to Changes Requested
- Approve: only if CI passes + scope matches TRD + you can name every standards memory you checked → move to Approved → write goal summary to `research/goals/goal-NN-short-title.md`
- Defer to human: leave a comment explaining uncertainty → note in backlog

### 8. Log format
```
## YYYY-MM-DD HH:MM ET REVIEWER
- did: reviewed PR #NN (round N)
- decision: approved / changes-requested / deferred
- standards checked: <list>
- metrics: run_type=productive | pr=PR-N | decision=X
- next: <what next>
```

## Hard rules
- Never merge anything
- Never approve if CI is failing
- Round 2+ reviews: only check whether original feedback was addressed — no new nits
- Justify or defer — every approval must cite which standards memories you checked
- One review per run, then exit
```

---

#### `prompts/trd-watcher.md` — skeleton

```markdown
# TRD Watcher

You are the TRD Watcher — a fast-running dedicated architectural reviewer. You review TRDs (Technical Requirements Documents), not code. Code reviews belong to the Reviewer agent.

## Wake-up checklist
1. PAUSE check (same pattern — exit if PAUSE file exists)
2. Check TRD_WATCHER_LOCK (7-minute threshold)
3. Scan `In Progress` AND `In Review` sections of backlog.md for any task with `TRD: ... — awaiting-review`
4. If none found: exit silently — no log, no Discord post
5. If found: check if you already left a TRD review comment on this PR in a prior run (`gh pr view <num> --comments`). If yes, exit silently — no re-review
6. Review the TRD for:
   - Does it cover all components the PRD requires?
   - Is the approach architecturally sound for the stack?
   - Any scope creep or premature abstraction?
   - Are schema changes appropriate?
   - Is there a real test coverage plan?
7. Approve: update the TRD field in backlog.md from `awaiting-review` to `approved`. Post a PR comment approving it.
8. Request changes: update the TRD field to `changes-requested: <specific reason>`. Move task to TRD Changes Requested in backlog.md.
9. One TRD per run, then exit.

## Hard rules
- Do NOT review code — only the TRD document
- No test runs — this is a document review
- Exit silently when nothing is pending — zero noise on no-op runs
- Do not re-review a TRD you already commented on (check PR comments first)
```

---

#### `prompts/project-manager.md` — skeleton

```markdown
# Project Manager

You are the Project Manager. You keep the developer's queue stocked and well-groomed. You do not write code or create strategy.

## Wake-up checklist
1. PAUSE check
2. Check PROJECT_MANAGER_LOCK (12-minute threshold)
3. Read backlog.md and implementation-roadmap.md
4. Count tasks in Ready. If < 2, check if the next goal in the roadmap has a PRD in `research/agents/prds/`. If no PRD, log the gap and wait — do NOT create tasks without a PRD.
5. If PRD exists: create 1–2 new Ready tasks that trace to that goal. Use the PRD for scope and acceptance criteria.
6. Groom: close stale tasks, reprioritize Ready order, flag anything blocked
7. Off-roadmap ideas go to proposals.md, not into the backlog
8. Log and post Discord summary

## Hard rules
- Never create a task without a PRD in prds/
- Every task must trace to a goal in the roadmap — no invented work
- Never write code or touch branches
- Off-roadmap ideas → proposals.md only
```

---

#### `prompts/product-manager.md` — skeleton

```markdown
# Product Manager

You are the Product Manager. PRD writing is your primary job. You never touch the backlog or write code.

## Wake-up checklist
1. PAUSE check
2. Check PRODUCT_MANAGER_LOCK (12-minute threshold)
3. Read implementation-roadmap.md — find the next 2 upcoming goals
4. For each goal: check if a PRD exists in `research/agents/prds/goal-NN-short-title.md`
5. If no PRD: write one. Use product-notes.md as your research source. If product-notes.md doesn't cover what you need for a solid PRD, do the research yourself — a weak PRD is worse than the time spent digging.
6. PRD should include: goal overview, user stories, UX flows, success metrics, scope boundaries, out-of-scope items
7. Log and post Discord summary

## Hard rules
- Never touch backlog.md
- Never write code
- A weak PRD that gets built into the wrong thing is worse than no PRD — if you don't have enough info, research first
```

---

#### `prompts/industry-researcher.md` — skeleton

```markdown
# Industry Researcher

You are the Industry Researcher — market intelligence for the product team. You run daily and research one topic per run in your product's market. You are a domain expert and research feed — never touch backlog.md, never write PRDs, never write code.

## Wake-up checklist
1. PAUSE check
2. Check what has been covered recently in product-notes.md to avoid repetition
3. Pick one topic in your domain: competitor moves, user pain points, pricing, compliance, integrations, market trends
4. Research it — use web search
5. Append findings to product-notes.md with a date header
6. If research surfaces a strong product insight worth raising as a proposal, append to proposals.md
7. Log and post Discord summary

## Hard rules
- Never touch backlog.md
- Never write PRDs
- Append-only to product-notes.md — never rewrite old entries
```

---

#### `prompts/merge-watcher.md` — skeleton

```markdown
# Merge Watcher

You are a lightweight unblocking helper. You run every 5 minutes. Run fast and exit.

## Wake-up checklist
1. PAUSE check + MW_PAUSE check
2. MERGE_WATCHER_LOCK check (8-minute threshold)
3. Check `git log origin/main --since='20 minutes ago' --oneline`
4. If merges found:
   a. Update backlog.md: find tasks whose blocked reason cites a dependency that just merged, move them to Ready
   b. For each merged PR, find matching task and move to Shipped. Append to velocity.md
5. ALWAYS sync open PR branches after checking for merges:
   - `gh pr list --state open --json headRefName,number`
   - For each: check if it's behind main, if so: `git checkout <branch> && git merge origin/main --no-edit && git push`
   - If conflict: resolve using judgment (take both sides' additive changes), commit, push. If you can't resolve: abort and post to Discord
6. If no merges and nothing to sync: log a no-op entry and exit silently
7. Post to Discord only if tasks were moved or branches were synced

## Hard rules
- Never create tasks or edit task scope
- backlog.md, agent-log.md, proposals.md, velocity.md are gitignored local-only files — NEVER git add them
- Commits from this agent are exclusively for merging main into feature branches
```

---

#### `prompts/system-reviewer.md` — skeleton

```markdown
# System Reviewer

You are the System Reviewer — the system's self-improvement loop. You run nightly, audit the last 24 hours, and file improvement proposals.

## Wake-up checklist
1. Read the last 24 hours of agent-log.md
2. Score 7 dimensions 1–5 with specific evidence from the log:
   - Developer throughput (tasks completed vs stalled)
   - TRD review speed (time from TRD commit to approval)
   - Code review quality (specificity of feedback, turnaround)
   - Backlog health (queue depth, staleness, blocked tasks)
   - PRD coverage (does PM have PRDs ahead of developer's work?)
   - Token efficiency (no-op rates, unnecessary Claude spawns, redundant work)
   - Overall
3. Every score below 4 requires specific evidence: timestamps, task IDs, log quotes
4. File proposals to proposals.md for every identified problem — include: what the problem is, evidence from the log, proposed fix, impact (H/M/L), effort (H/M/L)
5. Append a dated scorecard to system-health.md
6. Post Discord summary with scores and top 1–2 issues

## Hard rules
- Be honest — a 3/5 is fine if that's what the evidence shows
- No proposals without log evidence
- Don't score dimensions you don't have data for — mark as N/A
```

---

#### `prompts/codebase-auditor.md` — skeleton

```markdown
# Codebase Auditor

You are the Codebase Auditor — a quality-improvement agent that fires every few hours and audits one area of the codebase per run. You file concrete, scoped AUDIT-NNNN tasks to the backlog. You do NOT write code yourself.

## Wake-up checklist
1. PAUSE check
2. Check AUDITOR_LOCK (25-minute threshold)
3. Read `audit-state.md` to see which areas have been reviewed recently and pick an area that hasn't been touched in the longest time
4. Read the actual source files in that area — use Glob to find files, then read them. Never audit from memory.
5. Check against standards (backend: skinny controllers, business logic in service objects, SQL-first queries, N+1 patterns, unbounded list queries, proper scoping, request specs; frontend: arrow functions, async/await, axios via api/ modules, no business logic in components, proper hook patterns, MUI v7 syntax)
6. For each real violation found: file an AUDIT-NNNN task with:
   - Specific file path and line number
   - What the problem is
   - What the fix looks like (concrete — not "improve this", but "extract X to an interactor")
   - Acceptance criteria that proves it's fixed
7. Update `audit-state.md`: mark the area as reviewed with today's date
8. Log and post Discord summary

## Hard rules
- File tasks only for real violations — not style preferences, not hypothetical risks
- One area per run — don't try to audit the whole codebase at once
- Always read actual files (not from memory) before filing anything
- Never write code or touch branches
- AUDIT task IDs must be monotonic — scan backlog for the last AUDIT-NNNN and increment
```

---

#### `prompts/main-ci-fixer.md` — skeleton

```markdown
# Main CI Fixer

You are the Main CI Fixer — a trigger-based agent that fires every 2 minutes but does nothing unless a CI failure signal exists. You investigate main branch CI failures, understand what was being built, and open a PR with a fix.

## Wake-up checklist
1. Check for trigger file at `/tmp/[your-project]-main-ci-failed`. If it does not exist, exit immediately — this is a no-op, no log needed.
2. Read the trigger file — it contains the run ID and SHA
3. Delete the trigger file so the next run doesn't double-fire
4. Fetch the failed CI run output: `gh run view <run_id> --log-failed`
5. Identify the breaking commit: read git log, read the commit message, find the corresponding task in backlog.md
6. Do detective work — read the PRD, the TRD if it exists, understand what feature was being built and what the test was trying to verify
7. Fix the failure — fix the test (if it was testing wrong assumptions) OR fix the code (if it's genuinely broken). Don't guess — understand intent first.
8. Branch from main, push the fix, open a PR: `gh pr create --title "fix: [description]" --body "..."`
9. Log and post Discord summary

## Hard rules
- Always understand intent before fixing — read the task, PRD, and original PR
- Never just delete a failing test — understand why it was written before deciding it's wrong
- Never merge — open a PR and let the Reviewer handle it
- Delete the trigger file at the start of the run (step 3) to prevent double-firing
```

---

#### `prompts/pr-ci-fixer.md` — skeleton

```markdown
# PR CI Fixer

You are the PR CI Fixer — a trigger-based agent that fires every 2 minutes but does nothing unless a PR CI failure signal exists. You investigate failing PR CI runs, understand the task being built, and push a fix directly to the PR branch.

## Wake-up checklist
1. Check for trigger files matching `/tmp/[your-project]-pr-ci-failed-*`. If none exist, exit immediately — no log needed.
2. Pick the oldest trigger file. Read it — it contains the PR number, branch name, and run ID.
3. Delete the trigger file so the next run doesn't double-fire.
4. Fetch the failed CI output: `gh run view <run_id> --log-failed`
5. Find the corresponding task in backlog.md. Read its type:
   - TASK: read the PRD for context on what was being built
   - AUDIT: read the Issue/Fix fields for what was being fixed
   - BUG: read Repro/Expected/Actual fields
6. Check out the branch, understand the failure, fix it, push directly to the PR branch
7. Log and post Discord summary

## Hard rules
- Always read the task before fixing — understand what was being built
- Never delete a failing test — understand why it was written
- Push directly to the PR branch (not a new branch)
- Delete the trigger file at the start of the run (step 3) to prevent double-firing
- If the fix is complex or uncertain, post a Discord message asking for human review instead
```

---

### Step 2C: The backlog format

`research/agents/backlog.md` is the single source of truth for all tasks. Start with this template and fill in your first 2–3 Ready tasks:

```markdown
# Backlog

_Single source of truth for all agent tasks. State machine: Ready → In Progress → In Review → Approved → Shipped. See agent system guide for who moves what._

---

## Ready

### TASK-0001: [first task title]
- **Goal:** Goal 1 — [goal name from roadmap]
- **PRD:** research/agents/prds/goal-01-short-title.md
- **Scope:** [what's in] / NOT: [what's explicitly out]
- **Acceptance:**
  - [ ] [testable criterion 1]
  - [ ] [testable criterion 2]
- **PR:** (filled by Developer)
- **Branch:** (filled by Developer)
- **TRD:** (filled by Developer)
- **Notes:**

---

## In Progress

_(empty at start)_

---

## In Review

_(empty at start)_

---

## TRD Changes Requested

_(empty at start)_

---

## Changes Requested

_(empty at start)_

---

## Approved

_(empty at start)_

---

## Blocked

_(empty at start)_

---

## Shipped

_(empty at start)_
```

**Task field reference:**
- `Goal` — which goal from the roadmap this traces to (cite the goal number and name)
- `PRD` — path to the PRD file in `prds/`. Required before Developer can start.
- `Scope` — explicit in/out scope boundary. "NOT:" items prevent scope creep.
- `Acceptance` — testable checkbox criteria. Reviewer uses these to verify completeness.
- `PR` — filled by Developer when they open the draft PR
- `Branch` — filled by Developer when they create the branch
- `TRD` — filled by Developer: `research/plans/<branch>-trd.md — awaiting-review` → `approved` → (after code review) stays approved
- `Notes` — anything the reviewer should know; merge watcher appends ship dates here

**Task ID convention:** monotonic integers, zero-padded to 4 digits. Project Manager picks the next number.

---

### Step 2D: The implementation roadmap format

`research/implementation-roadmap.md` (or name it `implementation-roadmap-v2.md` etc.) is the Project Manager's law. Every task it creates must trace to a goal here. Keep it updated as priorities shift.

```markdown
# Implementation Roadmap

_Last updated: YYYY-MM-DD_
_Source of truth for goals in priority order. PM creates tasks only from goals listed here._

---

## Phase 1 — [Phase name, e.g. "Foundation"]

### Goal 1 — [Goal title]
**Why:** [one sentence on why this matters to the user]
**Scope:** [what this goal covers]
**Dependencies:** none
**Status:** In Progress / Not Started / Complete

### Goal 2 — [Goal title]
**Why:**
**Scope:**
**Dependencies:** Goal 1 complete
**Status:** Not Started

---

## Phase 2 — [Phase name, e.g. "Core Features"]

### Goal 3 — [Goal title]
...
```

---

### Step 2E: Standards memory files

Agents read a set of memory files before every run that define your coding standards. These files live in `workspace/memory/[your-project]-context/` and are referenced in each agent's prompt.

Suggested files to create:

- `feedback_backend_standards.md` — controller patterns, service object shapes, query patterns, test conventions
- `feedback_frontend_standards.md` — component conventions, hook rules, API module structure, styling rules
- `feedback_separation_of_concerns.md` — what lives where; what's allowed to cross which boundaries
- `feedback_pull_requests.md` — PR policy (open draft early, strip WIP when done, commit message format)
- `feedback_plan_files.md` — plan file format and conventions
- `project_[your-project].md` — current goal status, architecture overview, key file index, infra notes

The Reviewer cross-checks every PR diff against these. The Developer reads them before writing any code. When standards drift, update one file — all agents pick it up on the next run.

---

## Part 3: Cron Configuration

All jobs live in `workspace/crons/jobs.json`. This file is read by `cron-runner.js` on startup and polled periodically for changes.

### Full jobs.json structure

```json
{
  "jobs": [
    {
      "id": "your-project-agent-developer",
      "name": "Your Project — Developer (every 10 min)",
      "schedule": "0,10,20,30,40,50 * * * *",
      "tz": "America/New_York",
      "enabled": true,
      "timeoutSeconds": 1500,
      "model": "sonnet",
      "discordChannel": "YOUR_CHANNEL_ID",
      "announceResult": false,
      "message": "You are the Developer agent. Read your instructions at ./[your-project]/research/agents/prompts/developer.md and follow them exactly. Wake-up order: PAUSE check → DEV_LOCK check → fix-up changes-requested → resume In Progress (check TRD status) → in-flight cap → top Ready task. Never write feature code until TRD is approved. Never merge anything. Log the run and post a Discord summary."
    }
  ]
}
```

**Field reference:**

- `id` — unique string identifier for the job. Used in logs. Convention: `[project]-agent-[role]`.
- `name` — human-readable label shown in cron runner logs and the watch script.
- `schedule` — standard 5-field crontab syntax (`minute hour day month weekday`). Uses 24-hour time. Examples below.
- `tz` — IANA timezone for schedule interpretation (e.g. `America/New_York`, `America/Los_Angeles`, `UTC`).
- `enabled` — `true` to run, `false` to disable without deleting.
- `timeoutSeconds` — max runtime before the Claude process is killed. Set generously for complex agents (Developer: 1500s, Reviewer: 1200s). Fast agents (TRD Watcher, Merge Watcher: 300–360s).
- `model` — `"opus"` or `"sonnet"`. See model assignment guidance below.
- `discordChannel` — the channel ID where this agent posts its summary. Right-click a channel in Discord → Copy Channel ID.
- `announceResult` — `true` posts Claude's raw response to Discord after the run completes. `false` lets the agent handle its own Discord posts (preferred — gives the agent control over format).
- `message` — the full prompt sent to Claude Code. Can be short (point to a prompt file) or verbose. Pointing to a file is better — it separates concerns and lets you edit prompts without touching the JSON.
- `skipOnPreCommandFailure` — optional. If `true`, Claude is not spawned if `preCommand` exits non-zero. Useful to skip expensive Claude runs when there's nothing to do.
- `preCommand` — optional bash command that runs before Claude. Use it to check preconditions cheaply (shell scripts cost nothing; Claude spawns cost tokens). If it exits 0, Claude runs. If it exits 1 and `skipOnPreCommandFailure` is true, the entire job is skipped.

### Recommended schedule for each agent type

```
Developer:        0,10,20,30,40,50 * * * *    (every 10 min, on the :00)
Reviewer:         5,15,25,35,45,55 * * * *    (every 10 min, offset :05 — interleaves with Developer)
TRD Watcher:      2,7,12,17,22,27,32,37,42,47,52,57 * * * *  (every 5 min, offset :02)
Merge Watcher:    */5 * * * *                  (every 5 min, on the :00/:05/:10...)
Project Manager:  2,32 * * * *                 (every 30 min, offset :02)
Product Manager:  0 */4 * * *                  (every 4 hours)
Researcher:       0 7 * * *                    (daily at 7am)
System Reviewer:  0 21 * * *                   (daily at 9pm)
Codebase Auditor: 0 */3 * * *                  (every 3 hours)
Log Trim:         0 */4 * * *                  (every 4 hours)
Main CI Fixer:    */2 * * * *                  (every 2 min — trigger-based, almost always a no-op)
PR CI Fixer:      */2 * * * *                  (every 2 min — trigger-based, almost always a no-op)
```

**Why the offsets matter:** The Developer fires at :00 and writes a TRD. The TRD Watcher fires at :02 — it sees the TRD almost immediately and can approve it in the same 10-minute window. The Developer fires again at :10, sees the TRD approved, and starts building. Without the offset, the TRD Watcher and Developer would fire at the same time and race.

### Model assignments

**Use Opus for agents that make complex planning or judgment decisions:**
- Product Manager — writing PRDs requires reasoning about user needs, scope, and strategy
- Project Manager — grooming decisions require roadmap reasoning
- System Reviewer — auditing logs and scoring 7 dimensions requires careful analysis
- Reviewer — evaluating code correctness against nuanced standards requires careful judgment
- Codebase Auditor — finding real violations (not false positives) requires careful reading and judgment

**Use Sonnet for agents that execute against clear instructions:**
- Developer — coding is structured execution once the TRD is approved
- TRD Watcher — checking architectural soundness against a document
- Merge Watcher — mechanical git operations with some judgment
- Industry Researcher — search and summarization
- Log Trim — mechanical file operations
- Main CI Fixer — the context is handed to it (failed log + task + PRD); fixing tests is structured execution
- PR CI Fixer — same as Main CI Fixer

Sonnet is faster and cheaper. Opus catches subtle planning mistakes. Mismatch in either direction costs you: Sonnet on Product Manager produces weak PRDs; Sonnet on Reviewer misses subtle standards violations; Opus on Log Trim is pure waste.

### The preCommand pattern

`preCommand` lets you run a bash check before Claude is spawned. If the check fails (exits 1) and `skipOnPreCommandFailure: true`, the entire job is skipped — no Claude process, no tokens spent.

**When to use it:**
- When an agent has a frequent no-op case you can detect cheaply in bash
- When the work is conditional on a file existing or a grep match

**TRD Watcher example** — only spawn Claude if there's an `awaiting-review` TRD in the backlog:

```bash
"preCommand": "if [ -f /path/to/[your-project]/research/agents/PAUSE ]; then exit 1; fi; if grep -q 'awaiting-review' /path/to/[your-project]/research/agents/backlog.md 2>/dev/null; then exit 0; else TS=$(TZ=America/New_York date '+%Y-%m-%d %H:%M ET'); { echo ''; echo \"## $TS TRD-WATCHER\"; echo '- did: preCommand no-op — no TRDs awaiting-review'; echo '- metrics: run_type=no-op | reason=preCommand-skip'; } >> /path/to/[your-project]/research/agents/agent-log.md; exit 1; fi"
```

Note: in the preCommand, use absolute paths — the working directory is not guaranteed.

**Merge Watcher example** — only spawn Claude if there are recent merges to main or open PRs need syncing. Also implements auto-pause after N consecutive idle fires to prevent pointless runs during slow periods.

The preCommand writes to agent-log.md directly (as a no-op entry) so the log stays complete even when Claude is skipped. This is important — if you only have Claude-written log entries, gaps in the log are ambiguous (did the agent not run? or did it run and skip?).

---

## Part 4: SQLite + Gemini Semantic Memory System

The harness ships with a semantic memory system at `workspace/memory/`. It stores facts, people, preferences, and events with vector embeddings, then retrieves them by semantic similarity.

### What it does

- Stores arbitrary memory entries in SQLite (`knowledge.db`)
- Generates a vector embedding for each entry using the Gemini embedding API (`models/gemini-embedding-001`)
- On recall, embeds the query and finds the most similar memories by cosine similarity
- Also supports named entities (people, places) and relationships between entities

### Setup

```bash
# Install the SQLite native binding
cd workspace && npm install better-sqlite3

# Initialize the database (creates the tables and FTS index)
node workspace/memory/setup-db.js
```

This creates `workspace/memory/knowledge.db` with three tables:
- `memories` — arbitrary fact storage with embeddings
- `entities` — named people, places, organizations
- `relationships` — edges between entities (e.g. "Keith" ↔ "Marcus" → "friends")

### Writing memories

```bash
# Store a fact
node workspace/memory/remember.js "Keith prefers async/await over .then() chains" --type preference --tags "keith,coding" --importance 4

# Store a named entity
node workspace/memory/remember.js --entity "Marcus" --type person --notes "Keith's friend, works at biotech startup in SF"

# Store a relationship
node workspace/memory/remember.js --relationship "Keith" "Marcus" "friends" --notes "met at conference"
```

**Options:**
- `--type` — `fact` | `person` | `place` | `event` | `preference` (default: `fact`)
- `--tags` — comma-separated tags for filtering
- `--importance` — 1–5 (default: 3); 5 = critical context
- `--entity` — store as a named entity instead of a raw memory
- `--relationship` — store a relationship between two named entities

### Recalling memories

```bash
# Semantic search — returns top 5 most relevant memories
node workspace/memory/recall.js "what does Keith like about frontend development?"

# More results
node workspace/memory/recall.js "Keith's background" --limit 10

# Most recent entries
node workspace/memory/recall.js --recent 20

# List all named entities
node workspace/memory/recall.js --list-entities

# List all relationships
node workspace/memory/recall.js --list-relationships
```

### How to reference it in CLAUDE.md

In your `workspace/CLAUDE.md`, add a section that tells the main Claude session to use memory naturally:

```markdown
## Long-Term Memory (SQLite + Gemini embeddings)

You have a semantic memory system at `memory/knowledge.db`.

Write a memory when you learn something worth keeping:
  node memory/remember.js "content" --type fact|person|preference --tags "tag1,tag2" --importance 1-5

Recall memories when context from past conversations would help:
  node memory/recall.js "what is Keith working on?"
  node memory/recall.js --recent 20

When to write: facts about Keith's life, work, preferences, people he mentions, decisions he makes.
When to recall: at the start of a new topic, when Keith mentions a person or place.
Don't write: ephemeral task details, project technical state (that belongs in vetware-context/).
```

---

## Part 5: CLAUDE.md Setup

`workspace/CLAUDE.md` is the identity file for your main Discord Claude session (the one in tmux window 0, not the cron agents). It defines who that session is, what it knows, and what it does on startup.

### Key sections to include

**1. Identity**
Give the session a name and a clear role. Something like:
```markdown
# CLAUDE.md — [YourBotName] Identity & Session Init

You are **[YourBotName]** — [your name]'s AI assistant running on Claude Code, connected to Discord.
```

**2. Long-term memory** — see Part 4 above

**3. Primary project context**
Tell the session about your project, where the repo lives, and which files to always read before doing project work:
```markdown
## Primary Project: [Your Project]

[Project description]. The repo is symlinked at `./[your-project]/`.

Before doing any [your-project] work, always read:
- `memory/[your-project]-context/MEMORY.md` — index of everything you know
- `memory/[your-project]-context/project_[your-project].md` — current goals and progress
- `memory/[your-project]-context/feedback_*.md` — coding standards
```

**4. The agent system**
Make the session aware that there is an autonomous agent loop running in the background, what files it touches, and how to interact with it:
```markdown
## Agent System

A separate autonomous loop runs in `[your-project]/research/agents/`. It is NOT you — it's scoped cron agents working on the project around the clock.

Key files:
- `backlog.md` — task queue; only PM reorders Ready
- `agent-log.md` — append-only audit log, read from the bottom
- `PAUSE` — kill switch; when present every agent exits immediately
- `prompts/` — each agent's instructions (edit these to tune behavior)

When asked to update `agent-log.md`: append a new block at the END of the file.
When asked to pause agents: `touch [your-project]/research/agents/PAUSE`. Resume: remove the file.
```

**5. Session start protocol**
Tell the session what to do at the start of every conversation:
```markdown
## On Every Session Start

1. Read `memory/active-threads.md` if it exists — what's in progress
2. Read today's daily note `memory/YYYY-MM-DD.md` if it exists
3. If task is project-related, read files in `memory/[your-project]-context/`
4. Check if anything urgent needs attention
```

**6. Behavior policy**
```markdown
## How You Behave

- Do the work, then report. Don't narrate what you're about to do.
- Long tasks: post a "Done — [summary]" when finished.
- Write a brief daily note to `memory/YYYY-MM-DD.md` after completing anything important.
- Match the user's energy — direct, no corporate speak, no filler.

## Action Policy

- Internal (files, commands, memory): do it freely, no confirmation needed
- External (sending emails, posting to services): ask first unless it's an established workflow
- Destructive (deleting files, overwriting things): always warn first

## Formatting for Discord

- No markdown tables — use bullet lists (Discord doesn't render tables)
- Wrap URLs in <> to prevent link previews
- Keep responses concise
```

**7. Custom slash commands**
Register the agent-trigger commands so you can run agents on demand from Discord:
```markdown
## Custom Workflows

**/dev** — Execute the Developer agent. Read `[your-project]/research/agents/prompts/developer.md` and follow it exactly.

**/reviewer** — Execute the Reviewer agent. Check REVIEWER_LOCK first — if held < 20 min, report "reviewer is already running" and stop.

**/trd** — Execute the TRD Watcher. If no TRDs awaiting review, report that and exit.

**/pm** — Execute the Project Manager.

**/pause** — `touch [your-project]/research/agents/PAUSE` and report paused.

**/unpause** — Remove PAUSE and all per-agent pause files and idle counters. Report resumed.
```

---

## Part 6: Operational Runbook

### Pausing and resuming the agents

**Pause everything immediately:**
```bash
touch [your-project]/research/agents/PAUSE
```
Every agent checks for this file on wake-up and exits after logging one line. They will not touch the repo while this file exists.

**Resume:**
```bash
rm -f [your-project]/research/agents/PAUSE
# Also clear per-agent auto-pause files if any accumulated:
rm -f [your-project]/research/agents/DEV_PAUSE
rm -f [your-project]/research/agents/REV_PAUSE
rm -f [your-project]/research/agents/TRD_PAUSE
rm -f [your-project]/research/agents/MW_PAUSE
rm -f [your-project]/research/agents/PM_PAUSE
# Reset idle counters:
echo 0 > [your-project]/research/agents/REV_IDLE
echo 0 > [your-project]/research/agents/TRD_IDLE
echo 0 > [your-project]/research/agents/MW_IDLE
```

**When to pause:** before doing manual git operations on branches the agent is working on, before a big migration, before sensitive refactors that would conflict with agent commits.

### Watching the live schedule

```bash
node workspace/scripts/agent-watch.js
```

This renders a live dashboard in your terminal, refreshing every second. For each agent it shows:
- Time until next fire
- Time of next fire
- A progress bar showing how far through the interval you are
- `● RUNNING` badge if the agent's lock file is held and fresh
- `⏸ PAUSED` badge if PAUSE file or a per-agent pause file exists
- `⚠ STUCK` badge if the lock file is stale (> lock threshold) — means a prior run probably crashed or timed out
- A `↳ next:` hint showing the last "next:" line from the agent's log entry — tells you what the agent expected to do next

To add a new agent to the watch display, edit the `AGENTS` array at the top of `agent-watch.js`:

```javascript
{ name: 'My Agent', logRole: 'MY-AGENT', schedule: [0, 30], maxSecs: 1800, color: '\x1b[36m', lockFile: MY_LOCK, lockMaxMs: 15*60*1000 }
```

`schedule` is minutes-past-the-hour. For daily agents use `daily: { hour: N }`. For every-N-hours use `everyNHours: N`.

### Reading the agent log

`research/agents/agent-log.md` is append-only. Read from the bottom (newest entries last). Agents read only the last 75 lines on each wake-up to keep token usage bounded.

Each entry looks like:
```
## 2026-04-14 10:00 ET DEVELOPER
- did: started TASK-0042 — wrote TRD for goal-05-notifications branch
- task: TASK-0042
- trd: written, awaiting-review
- tests: skipped (TRD-only run, no implementation yet)
- metrics: run_type=productive | commits=1 | tests_added=0 | trd_cycles=0
- next: resume building after TRD approval
```

What to look for:
- `run_type=no-op` — agent fired but had nothing to do. Frequent no-ops are normal for TRD Watcher and Merge Watcher. Frequent Developer no-ops mean the queue is empty or the in-flight cap is blocking.
- `run_type=productive` — agent did real work. Look at `commits=N` and `tests_added=N`.
- Missing entries for a time slot — agent either didn't fire (check cron-runner logs) or preCommand skipped it.
- STUCK in the watch script — look for the last log entry from that agent, check if it logged "started run" but never logged "released lock". That's a timeout or crash.

Log entries older than 24–48 hours are archived by the Log Trim agent to `agent-log-archive-YYYY-MM.md`. You rarely need to read archives; they exist so the active log stays small.

### Reading system-health.md

`research/agents/system-health.md` receives a new scorecard every night from the System Reviewer. Format:

```
## System Health — 2026-04-14

Developer throughput:  4/5 — completed TASK-0041 and TASK-0042, no idle cycles
TRD review speed:      5/5 — both TRDs approved within 2 minutes of commit
Code review quality:   3/5 — TASK-0041 approval lacked specific standards citations
Backlog health:        4/5 — 2 tasks in Ready, 1 in flight, no blockers
PRD coverage:          5/5 — Product Manager has PRDs through Goal 7
Token efficiency:      2/5 — Developer fired 6 times with no-op (TRD awaiting approval)
Overall:               4/5

Issues filed to proposals.md:
- [MEDIUM] Developer fires while TRD pending — add preCommand to skip if TRD is awaiting-review and no other work exists
```

Scores below 3 with specific evidence should prompt you to read the proposals.md entry and decide whether to implement the fix.

### Debugging a stuck run

1. Check the watch script — is there a `⚠ STUCK` badge?
2. If yes, look at the lock file: `cat research/agents/DEV_LOCK` — it shows which task was claimed and when.
3. Look at the log: read the last few entries from that agent in agent-log.md. Did it log "started run" without a closing entry?
4. Check the cron runner logs: `tail -100 workspace/crons/logs/cron-runner.log` — did the process start? Did it time out?
5. If confirmed stuck: manually delete the lock file: `rm -f research/agents/DEV_LOCK`. The next run will start fresh.
6. Check the branch state in git — is there a pending commit that needs pushing? Is the branch in a bad merge state?
7. If the branch is dirty and you're not sure what the agent was doing: `cd [your-project] && git status` and `git diff`. Clean it up manually, then touch PAUSE, reset the state, and remove PAUSE.

### Handling merge conflicts

The Merge Watcher automatically tries to merge main into all open PR branches. If it can't resolve a conflict, it aborts the merge and posts to Discord with the branch name and conflicting files. To resolve manually:

```bash
cd [your-project]
git checkout [branch-name]
git merge origin/main
# Resolve conflicts in your editor
git add [resolved-files]
git commit --no-edit
git push
```

Then the Merge Watcher will find the branch clean on the next run and skip it.

---

## Part 7: Tuning and Customization

### Tuning agent behavior

The fastest way to change how an agent behaves is to edit its prompt file. Changes take effect on the next cron fire — no restarts needed.

Common tuning scenarios:

**Agent is being too aggressive (approving too fast):**
Find the prompt's `Hard rules` section and add an explicit rule. E.g. for the Reviewer: "Always run `gh pr checks <num>` before approving. Never approve if any check is failing or pending."

**Agent is being too conservative (blocking everything with changes-requested):**
Add explicit guidance about what deserves a pass. E.g.: "Minor style issues that don't violate a written standard should not block approval. Only request changes for violations of documented rules in the standards memory files."

**Agent is doing something it shouldn't:**
Add a "NEVER" rule. Clear prohibitions in the Hard Rules section are the most reliable way to stop unwanted behavior. E.g.: "NEVER create a branch from staging — always from main."

**Agent is missing context:**
Add the file to its "Read these before doing anything" section. The agent reads these on every wake-up, so any new context file is immediately available.

**Agent's log entries are hard to parse:**
Update the "Log format" section in the prompt with the exact format you want, including an example.

### Adding a new agent

1. Create the prompt file at `research/agents/prompts/[new-agent].md`. Include the four standard sections: identity/context, read-first list, wake-up checklist (with PAUSE check and mutex lock), log format.

2. Add a lock file constant to `agent-watch.js` and add the agent to the `AGENTS` array.

3. Add the job to `workspace/crons/jobs.json`:
```json
{
  "id": "your-project-agent-[new-agent]",
  "name": "Your Project — [New Agent] ([schedule description])",
  "schedule": "CRON_EXPRESSION",
  "tz": "America/New_York",
  "enabled": true,
  "timeoutSeconds": 600,
  "model": "sonnet",
  "discordChannel": "YOUR_CHANNEL_ID",
  "announceResult": false,
  "message": "You are the [New Agent] in the [your-project] multi-agent cron system. Read your full instructions at ./[your-project]/research/agents/prompts/[new-agent].md and follow them exactly. [One sentence summary of the job]. Log the run and post a Discord summary."
}
```

4. If the new agent has a common no-op case (it often fires and finds nothing to do), write a `preCommand` that detects that cheaply in bash and skips the Claude spawn.

5. Enable it in jobs.json (`"enabled": true`) and watch the first few runs in agent-watch.js.

### Adjusting schedules

Edit the `schedule` field in `jobs.json`. The cron runner picks up changes on the next reload (it polls the file). Standard 5-field crontab syntax:

```
┌──────── minute (0–59)
│ ┌────── hour (0–23)
│ │ ┌──── day of month (1–31)
│ │ │ ┌── month (1–12)
│ │ │ │ ┌ day of week (0–7, 0 and 7 are Sunday)
│ │ │ │ │
* * * * *

Examples:
"0,10,20,30,40,50 * * * *"   every 10 minutes
"5,15,25,35,45,55 * * * *"   every 10 minutes, offset by 5
"*/5 * * * *"                every 5 minutes
"2,32 * * * *"               every 30 minutes at :02 and :32
"0 */4 * * *"                every 4 hours at the top of the hour
"0 7 * * *"                  daily at 7am
"0 21 * * *"                 daily at 9pm
"0 18 * * 5"                 Fridays at 6pm
"0 9 * * 1-5"                weekdays at 9am
```

To disable a job without deleting it: set `"enabled": false`. It will still appear in the watch script but won't fire.

### Token optimization tips

- Use `preCommand` + `skipOnPreCommandFailure` for agents that frequently have nothing to do (TRD Watcher, Merge Watcher). A preCommand costs nothing; a Claude spawn costs tokens even for a no-op.
- Don't run Opus where Sonnet is sufficient. The gap is meaningful at scale.
- Keep `timeoutSeconds` proportional to the agent's actual runtime. A 1500s timeout on the Log Trim job just means a stuck run stays stuck longer.
- The Log Trim agent is important — a large `agent-log.md` causes every agent to burn tokens re-reading history. Keep it running.
- The `announceResult: false` pattern (agent handles its own Discord posts) prevents double-posting and lets the agent decide whether a run is worth reporting.

---

## Getting Started Checklist

**Discord & harness:**
- [ ] Create Discord bot (developer portal → new app → bot token → enable Message Content Intent → invite to server)
- [ ] Clone claude-code-discord-starter, fill in `config/.env`
- [ ] Symlink or clone your project into `workspace/`
- [ ] Write `workspace/CLAUDE.md` (identity file for the main Discord session)

**Project repo setup:**
- [ ] Create the directory structure in `research/agents/` (see Part 2A)
- [ ] Add agent state files to `.gitignore` (see Part 2A gitignore block)
- [ ] Write your standards memory files in `workspace/memory/[your-project]-context/`
- [ ] Write your implementation roadmap at `research/implementation-roadmap.md`
- [ ] Write all 11 agent prompt files (developer, reviewer, trd-watcher, project-manager, product-manager, industry-researcher, merge-watcher, system-reviewer, codebase-auditor, main-ci-fixer, pr-ci-fixer)
- [ ] Have the Product Manager write PRDs for the first 2–3 goals manually before enabling the Developer
- [ ] Create `research/agents/backlog.md` with 2–3 Ready tasks (with PRD fields filled in)

**Cron configuration:**
- [ ] Configure `workspace/crons/jobs.json` with all agents, your channel IDs, and schedules
- [ ] Add all agents to `agent-watch.js` AGENTS array

**VPS & CI (optional but recommended):**
- [ ] Provision a Linux VPS (4GB RAM minimum for one project with CI)
- [ ] Deploy the discord starter to the VPS; start it inside a persistent tmux session
- [ ] Install and register a self-hosted GitHub Actions runner on the VPS
- [ ] Add the `fix-main-on-failure` and `signal-pr-failure` jobs to your CI workflow (see Part 1E)
- [ ] Verify the runner is active: `cd /path/to/actions-runner && ./run.sh` or run as a service

**Memory (optional):**
- [ ] `node workspace/memory/setup-db.js` and add GEMINI_API_KEY to config/.env

**Launch:**
- [ ] Run `bash start-local.sh` (or the equivalent on your VPS) to start the system
- [ ] Run `node workspace/scripts/agent-watch.js` in a terminal to monitor
- [ ] Touch `research/agents/PAUSE` before any manual repo changes; remove when done
- [ ] After the first full cycle, read `agent-log.md` and verify each agent is logging correctly

---

## Recent System Health

_(updated nightly by System Reviewer)_

| Date | Dev | TRD | Review | Backlog | PRD | Tokens | Overall |
|------|-----|-----|--------|---------|-----|--------|---------|
| 2026-04-12 | 4/5 | 4/5 | 3/5 | 3/5 | 4/5 | 2/5 | **3/5** |
| 2026-04-13 | 4/5 | 5/5 | 3/5 | 4/5 | 5/5 | 1/5 | **3/5** |
| 2026-04-14 | 5/5 | 5/5 | 3/5 | 4/5 | 4/5 | 3/5 | **4/5** |
| 2026-04-17 | 5/5 | 5/5 | 3/5 | 5/5 | 5/5 | 3/5 | **4/5** |
| 2026-04-18 | 4/5 | 3/5 | 3/5 | 1/5 | 5/5 | 2/5 | **3/5** |
| 2026-04-19 | 4/5 | 3/5 | 2/5 | 3/5 | 4/5 | 1/5 | **3/5** |
| 2026-04-20 | 5/5 | 4/5 | 4/5 | 4/5 | 4/5 | 3/5 | **4/5** |
| 2026-04-23 | 5/5 | 4/5 | 4/5 | 4/5 | 5/5 | 2/5 | **4/5** |
| 2026-04-24 | 5/5 | 5/5 | 4/5 | 4/5 | 5/5 | 3/5 | **4/5** |
| 2026-04-25 | 5/5 | 5/5 | 2/5 | 3/5 | 5/5 | 4/5 | **3/5** |
