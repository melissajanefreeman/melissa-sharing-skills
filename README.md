# melissa-sharing-skills

A collection of Claude Code skills I've built and use personally.

## What are skills?

Skills are reusable prompt files for [Claude Code](https://claude.ai/code). You invoke them with `/skill-name` in the Claude Code CLI, and they expand into a full prompt that guides Claude's behavior for a specific task.

Skills live in `~/.claude/skills/<skill-name>/SKILL.md`.

---

## Skills

### `orchestrate`

**Evaluate a task and decide whether to run it directly or decompose it into sub-agents.**

Before launching any sub-agent, Claude runs this skill to decide the right execution strategy — keeping the main context window small and delegating heavy reading/analysis work to isolated sub-agents.

**Capabilities:**
- Estimates context cost and decides whether sub-agents are worth it
- Detects iterative tasks (e.g. "fix all failing tests") and auto-starts a Ralph loop
- Enumerates files before decomposing, so agents always get explicit file lists
- Runs agents in parallel batches (capped at 3 to avoid rate-limit cuts)
- Writes checkpoint files per agent so interrupted runs can resume
- Delegates synthesis to a dedicated agent to keep the orchestrator's context clean
- Cleans up `/tmp` checkpoint files on completion

**When it skips sub-agents:**
- Task touches 1–3 known files
- Focused question with a bounded answer
- Quick edit or single-file generation
- Orchestration overhead would exceed the work itself

---

## Handling rate limits with `/loop`

When orchestrating long-running multi-agent tasks, rate limits can interrupt agents mid-run. The skill handles resumption via checkpoint files — but you need something to periodically kick Claude back into action after a rate limit resets.

Use the `/loop` skill to send a resume prompt on a timer:

```
/loop 5h resume all agents. rate limit reset
```

This sends `"resume all agents. rate limit reset"` every 5 hours, which triggers the orchestrator to check its manifest, find any interrupted agents, and relaunch them from their last checkpoint.

Adjust the interval to match your plan's rate limit window (e.g. `/loop 3h` for API tier, `/loop 5h` for Pro).

---

## Installation

Copy the skill folder into your Claude Code skills directory:

```bash
mkdir -p ~/.claude/skills/orchestrate
curl -o ~/.claude/skills/orchestrate/SKILL.md \
  https://raw.githubusercontent.com/melissajanefreeman/melissa-sharing-skills/main/orchestrate/SKILL.md
```

Or clone the repo and symlink/copy manually:

```bash
git clone https://github.com/melissajanefreeman/melissa-sharing-skills.git
cp -r melissa-sharing-skills/orchestrate ~/.claude/skills/
```

Then invoke it in Claude Code with:

```
/orchestrate
```

Or add this to your `~/.claude/CLAUDE.md` to enforce it globally:

```
Always invoke the `orchestrate` skill before launching any sub-agent.
```
