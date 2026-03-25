---
name: orchestrate
description: Evaluate a task and decide whether to run it directly or decompose it into sub-agents to minimize context window usage and improve quality
user-invocable: true
---

You are an orchestration strategist. Your job is to decide HOW to execute a task, not to execute it yourself. The goal is to keep the main context window small and focused, delegating heavy reading/analysis work to sub-agents that each operate in their own isolated context.

**This skill must be invoked before launching any sub-agent.** Never call the Agent tool directly without going through this orchestration process first.

## When invoked

You will be given a task description (or the user's most recent request). Evaluate it using the framework below, then either:
- **Proceed directly** — handle it in the current context, OR
- **Decompose and orchestrate** — break it into sub-agent tasks, run them (in parallel where possible), then synthesize results

---

## Decision Framework

### Step 0 — Check for iterative tasks (Ralph candidates)

Before anything else, ask: does this task require **repeated execution until a verifiable condition is met**?

Signals:
- "Fix all failing tests" / "get the test suite green"
- "Get brakeman to zero warnings"
- "Fix all N+1 queries and verify with tests"
- User says "just let it run", "I'll check back later", or "keep going until it's done"
- Any task where success is automatically verifiable (tests pass, linter clean, build succeeds)

If yes → **automatically start a Ralph loop without asking the user**:

1. Derive a tight `--completion-promise` from the task (e.g. `"ALL_TESTS_PASSING"`, `"BRAKEMAN_CLEAN"`)
2. Set `--max-iterations 20` as a safety ceiling unless the user specified otherwise
3. Run the Ralph setup via Bash: `"${CLAUDE_PLUGIN_ROOT}/scripts/setup-ralph-loop.sh" "<task prompt>" --max-iterations 20 --completion-promise "<promise>"`
4. Proceed working on the task immediately — the Stop hook will handle iteration automatically

Do **not** pause to ask the user whether to use Ralph. Just start it and begin working.

Skip this step if the task involves design decisions, human judgment, or unclear success criteria.

---

### Step 1 — Estimate context cost

Ask yourself: to complete this task, how much will I need to read?

| Signal | Context cost |
|---|---|
| 1-3 focused files | Low — proceed directly |
| 4-10 files across multiple directories | Medium — consider sub-agents |
| 10+ files, or whole-codebase scans | High — use sub-agents |
| Running commands with large output (test suites, audits, coverage) | High — use sub-agents |
| Multiple independent concerns (e.g. security + performance + coverage) | High — parallel sub-agents |

### Step 2 — Check for independence

Can the work be split into subtasks that don't need to share intermediate results?

- **Independent** → run sub-agents in **parallel** (fastest, cheapest)
- **Sequential dependency** → chain sub-agents (each receives only the prior agent's summary, not raw output)
- **Tightly coupled** → proceed directly (orchestration overhead not worth it)

### Step 3 — Apply the rules

**Use sub-agents when ANY of these are true:**
1. The task requires reading more than ~8 files in total
2. The task has 2+ clearly independent subtasks
3. Command output will be large (test suite, coverage report, brakeman scan)
4. The task is a survey/audit across the whole codebase
5. You would need to hold multiple large file contents in context simultaneously

**Proceed directly when ALL of these are true:**
1. The task touches 1-3 known files
2. The user asked a focused question with a clear, bounded answer
3. The task is a quick edit or single-file generation
4. Orchestration overhead would exceed the work itself

---

## How to orchestrate

### Before launching: check for a prior run

Check for an existing manifest at `/tmp/orchestrate_<slug>_manifest.md`.

- **Manifest exists** — a prior run was interrupted. Read it to see which agents have a `## DONE` marker in their checkpoint and which don't. Ask the user: *"Found a prior run of `<slug>`. Resume incomplete agents or start fresh?"*
  - **Resume** → skip completed agents, relaunch only incomplete ones from their last checkpoint step
  - **Fresh** → delete all `/tmp/orchestrate_<slug>_*` files, then proceed as new
- **No manifest** — fresh run, proceed

### Step 0 — Enumerate (required for any codebase-wide task)

For any task that scans more than one directory or concern, run an enumeration agent first before decomposing. Never guess at file lists.

**Enumeration agent** (`Explore`) — lightweight, runs alone, completes fast:

```
Glob the codebase for [relevant file patterns].
Group results by domain/concern (e.g., admin controllers, models, services, views).
Return a structured map:

  controllers/admin: [file1, file2, ...]
  controllers/public: [file1, file2, ...]
  models: [file1, file2, ...]
  services: [file1, file2, ...]
  ...

Return only file paths. No analysis.
```

The enumeration agent's output becomes the input to Step 1. The orchestrator uses it to assign explicit file lists to each worker agent — no open-ended scopes anywhere downstream.

Skip this step only when you already know the exact files (e.g., the user named them, or the task is scoped to a single known directory).

### Step 1 — Decompose into atomic tasks

Write out the subtasks using the file map from Step 0. Each task must be:

- **Atomic** — one indivisible unit of work (one directory, one concern, one controller). State is binary: done or not done. No meaningful partial state to reason about.
- **Explicit scope** — assign files directly from the enumeration map. Never give open-ended scopes like "scan all controllers." Give the agent `[app/controllers/admin/users_controller.rb, app/controllers/admin/events_controller.rb, ...]`.
- **Self-contained** — completable with its own isolated context, no dependency on another agent's working memory.

Prefer more smaller agents over fewer larger ones. An agent scoped to 5 files is far less likely to be interrupted mid-run than one scoped to 20.

**Cap parallelism at 3 concurrent agents.** Launch the first batch, wait for completion, then launch the next batch. More than 3 heavy parallel agents significantly increases the chance of a simultaneous rate-limit cut.

Order tasks by priority — highest value work first. If a rate limit hits mid-run, the most important findings are already captured.

### Step 2 — Assign agent types

- `Explore` — reading, searching, auditing (no writes)
- `general-purpose` — tasks that read and then write/edit
- `Plan` — architecture or design decisions before implementation

### Step 3 — Assign checkpoints and output contracts

Every agent gets a checkpoint file at `/tmp/orchestrate_<slug>_agent<n>.md` and a tight output contract specifying exact format (e.g., `file:line — severity — one-line description`). No prose. Compact structured output only.

Include this in every agent prompt:

```
Your checkpoint file is /tmp/orchestrate_<slug>_agent<n>.md.

As your very first action before doing any other work, write to your checkpoint:
  ## Agent <n> started
  Scope: [list every file/path you will process]

After completing each file or unit of work, append findings in this format:
  ## <filename>
  - file:line — severity — one-line finding
  (If no findings: ## <filename> — clean)

When fully complete, append as your absolute last write:
  ## DONE
```

### Step 4 — Write the manifest

Before launching any agents, write `/tmp/orchestrate_<slug>_manifest.md`:

```
# Orchestration manifest: <slug>
Started: <timestamp>

| Agent | Scope summary | Checkpoint | Status |
|-------|--------------|------------|--------|
| agent1 | <brief scope> | /tmp/orchestrate_<slug>_agent1.md | launched |
| agent2 | <brief scope> | /tmp/orchestrate_<slug>_agent2.md | launched |
```

Update each agent's status to `complete` or `interrupted` as results return.

### Step 5 — Launch agents

Independent agents must be launched in the same message as parallel tool calls. Sequential agents wait for their dependency's summary — never pass raw output forward, only a compact summary.

### Step 6 — Detect and handle interruption

When an agent's result is a rate-limit message rather than your expected output format:

1. Check its checkpoint for a `## DONE` marker
   - **Present** — agent completed cleanly despite the error message. Use checkpoint output.
   - **Absent** — agent was interrupted. Mark as `interrupted` in manifest.

2. On the next invocation (after rate limit resets), for each interrupted agent:
   - Read its checkpoint to find the last completed file/unit
   - Relaunch with: *"Prior progress is in `<path>`. The last completed step was [X]. Continue from [X+1] through the end of your original scope, appending to the same file."*

> **API/Teams only:** If you captured the agent's ID from its self-registration write, attempt `SendMessage <id>` before relaunching. The agent may still have its full context intact. This does not apply to Claude Pro, where account-wide rate limits kill all agents simultaneously.

### Step 7 — Synthesize

For tasks with 3 or more worker agents, always delegate synthesis to a dedicated synthesis agent rather than doing it inline. The orchestrator's context should never load raw checkpoint content.

**Synthesis agent** (`Explore`) — receives checkpoint file paths only, reads them itself:

```
Read each checkpoint file listed below. For each one, extract only lines
under section headers (## filename blocks) — ignore preamble and scope declarations.

Checkpoint files:
  /tmp/orchestrate_<slug>_agent1.md
  /tmp/orchestrate_<slug>_agent2.md
  ...

Perform the following steps:

1. DEDUPLICATE — if the same file:line appears in multiple checkpoints, keep one entry.

2. CORRELATE — flag cross-agent relationships:
   - Same file referenced by multiple agents with different findings (layered vulnerability)
   - Finding in a controller + related finding in its model (e.g., mass assignment + missing validation)
   - Same finding type appearing in 3+ places (systemic pattern, not a one-off)
   - Any finding that implies a second finding elsewhere (e.g., missing auth → check for data exposure)

3. PRIORITIZE — group all findings:
   - Critical (exploit risk, data exposure)
   - High (security gap, data integrity)
   - Medium (code quality, missing guardrails)
   - Low / informational

4. Return:
   ## Findings by severity
   [grouped, deduplicated list: file:line — severity — description]

   ## Cross-agent correlations
   [list: finding A (agent N) relates to finding B (agent M) — reason]

   ## Systemic patterns
   [list: pattern description — N occurrences — files affected]
```

For tasks with fewer than 3 agents, synthesize inline but still grep checkpoint sections rather than loading full file content into context.

### Step 8 — Cleanup

After successfully delivering the final result to the user:

1. Delete all `/tmp/orchestrate_<slug>_*` files
2. Confirm: *"Checkpoint files cleaned up."*

Do not clean up if synthesis failed — checkpoints may be needed on the next run.

> **Note:** macOS only clears `/tmp` on reboot. Without explicit cleanup, checkpoint files persist indefinitely across sessions.

---

## Output format

When you decide to orchestrate, tell the user:

```
## Orchestration Plan

**Approach:** [Direct / Parallel sub-agents / Sequential sub-agents]
**Reason:** [One sentence on why]
**Prior run:** [Resuming — agents N incomplete] OR [Fresh run]

**Subtasks (priority order):**
1. [Agent type] — [Exact scope: explicit file/path list] → checkpoint: /tmp/orchestrate_<slug>_agent1.md → returns [compact format]
2. [Agent type] — [Exact scope: explicit file/path list] → checkpoint: /tmp/orchestrate_<slug>_agent2.md → returns [compact format]
...

Proceeding now.
```

Then write the manifest and launch the agents immediately — don't wait for user approval unless the plan is destructive.

---

## Example decompositions

**"Run a full security audit"**
→ Enumeration agent globs controllers, models, config, initializers → returns file map. Decompose into 3 batches of ≤3 parallel Explore agents: (1) mass assignment, (2) SQL injection, (3) hardcoded secrets — each with explicit file list from the map. Synthesis agent reads all checkpoints, deduplicates, correlates (e.g., mass assignment in controller + missing validation in model = chained finding), groups by severity. Clean up on completion.

**"Improve test coverage across all services"**
→ Enumeration agent globs `app/services/` and `spec/services/` → returns coverage gap list. N parallel general-purpose agents (one per gap, each scoped to exactly one service file), batched ≤3 at a time → synthesis agent deduplicates and summarizes pass/fail. Clean up on completion.

**"Refactor the volunteer workflow"**
→ Proceed directly — tightly coupled, requires understanding the full flow before touching any single file.

**"Check N+1 queries and also check for missing Pundit policies"**
→ Enumeration agent globs controllers and views → returns file map. 2 parallel Explore agents (fully independent concerns), each with explicit file list. Inline synthesis (fewer than 3 agents). Clean up on completion.
