---
name: autoresearch
description: Use when the user wants Claude to run an autonomous, long-running, iterative research/experimentation loop — karpathy-style autoresearch but upgraded for the Claude Code ecosystem. Triggers include phrases like "autoresearch", "自动迭代研究", "自主跑实验", "claude --autoresearch", "让你自己迭代", "代替 deepseek/openai 当研究员". Provides: three-phase state machine (interactive planning → user confirms → autonomous execution until budget exhausted), A+B hybrid architecture (main session coordinator + per-iteration fresh subagent for clean context + parallel subagents during waits), MD-as-single-source-of-truth state externalization, crash/disconnect resilience (tmux + durable cron + SessionStart hook), and the hard platform constraint that subagents cannot spawn sub-subagents (depth capped at 1).
---

# Autoresearch Framework

Use this skill when the user asks you to **run autonomous iterative research** — not a one-shot task. You are the researcher, not just the coder. The user will give you a goal and a time budget; you drive the loop until the budget is exhausted or the goal is met.

This skill is an upgrade of karpathy/autoresearch: instead of a dumb "LLM proposes patch → apply → score → reset" loop, leverage Claude Code's full toolset (subagents, memory, skills, scheduled wake-ups, hooks) and externalize all state to Markdown so iterations can hand off cleanly.

**Prerequisite skill**: [experiment-ops-playbook](https://github.com/guangxiangdebizi/experiment-ops-playbook-skill) — use its versioning rules (Experiment N / vNN), its two canonical docs (`EXPERIMENTS.md` + `CURRENT_STATUS.md`), and its templates (`references/templates.md`). Do not re-invent that layer; this skill composes on top of it.

## Hard platform constraints (discovered by testing, not guessing)

- **Subagent recursion is disabled**: a `general-purpose` subagent does **not** have the `Agent` tool in its runtime toolset, even though the top-level description implies it does. Maximum spawn depth is **1** (main → subagent; subagent cannot spawn grandchild). Design every architectural decision around this.
- **Claude Code is turn-based, not daemonic**: nothing runs when Claude is not in an active REPL turn. "Self-driving" means leaving a wake-up cron/schedule at the end of a turn, not a background thread.
- **`ScheduleWakeup` is session-only**; **`CronCreate(durable: true)`** survives process restarts. For any loop that must survive disconnects, use durable crons.

## Three-phase state machine

### Phase 1 — Interactive planning (do NOT run research tools)

When this skill activates, you are in planning mode. Your only job is conversation: extract a clean research contract and write it to `plan.md`. Do not Edit, Bash (beyond reading), or start any experiment. Keep asking until all seven fields are nailed:

| Field | What to ask |
|---|---|
| `research_dir` | Where should the research folder live? (Default suggestion: `~/.claude/autoresearch/<session_id>/`, but ALWAYS offer to place it inside the user's project instead. Let them pick.) |
| `goal` | What is the target? One sentence. |
| `success_criteria` | How do we know we won? Quantify if possible. |
| `budget_hours` | Hard time limit. |
| `editable_surface` | Which files/dirs can you touch? Which must stay frozen? |
| `gates` | Validation ladder, env-specific (e.g. `syntax → local_import → kaggle_kernel_complete → kaggle_public_score`). Names come from the project, not a fixed vocabulary. |
| `stop_conditions` | All reasons to halt: budget exhausted, quota exhausted, goal reached, N consecutive regressions, user says stop. |
| `known_context` | What has already been tried, what is forbidden, any prior failure modes to avoid. |

After drafting `plan.md`, show it to the user. They can edit verbally ("改第 4 条的预算为 6 小时"). Iterate until they explicitly say **"开始"** / **"start"** / **"confirm"** (or any unambiguous go-signal). That's the Phase 1 → 2 transition.

### Phase 2 — Autonomous execution (the loop)

This phase is the A+B hybrid. **Do not try to do everything in one big turn; externalize state and hand off.**

**Role split**:
- **Main session (A)** = coordinator. Holds the dialogue with the user, maintains `CURRENT_STATUS.md` + `EXPERIMENTS.md`, schedules wake-ups, launches subagents. Stays long-lived.
- **Per-iteration subagent (B)** = worker. Each experiment iteration is delegated to a fresh `general-purpose` subagent. That subagent reads the MDs, runs one attempt, updates the MDs, returns a short report. Its context dies with it — the MDs are the hand-off medium.
- **Parallel research subagents** = bonus. While a slow gate is running (e.g. 40-min Kaggle kernel), the main session can spawn additional subagents for data analysis, literature scan, or old-log mining. Their reports feed the NEXT iteration's design.

**Each iteration (the subagent's 9 steps)**:
1. Read `plan.md`, `CURRENT_STATUS.md`, `EXPERIMENTS.md`, and the latest gate log.
2. Decide: same experiment family (new `vNN`) or new family (new `expN`)? Follow the experiment-ops-playbook versioning rules.
3. Create the versioned artifact file (never overwrite a file that has produced an official result).
4. Run the weakest gate (local sanity / syntax / import).
5. Run the next gate up. Stop climbing if it fails.
6. Append an entry to `EXPERIMENTS.md` using the template — success AND failure both logged, with concrete IDs (run id, submission id, timestamps, deltas).
7. Update `CURRENT_STATUS.md` to the new latest truth: current best-on-weak-gate, current best-on-strongest-gate, next recommended direction.
8. Write one learning to auto-memory if it's a durable constraint (e.g. "X flag crashes model Y on this runner").
9. Report back to main session: `{iteration_n, outcome, next_delay_seconds, rationale}`.

**Main session then**:
- Decides next wake-up delay based on state (e.g. 20 min if a Kaggle kernel is running, 3 min if just analyzing).
- Schedules via `CronCreate(durable: true, recurring: false)` with a prompt like `"继续 autoresearch 迭代 in <research_dir>"`.
- Checks total elapsed against `budget_hours`. If exhausted → Phase 3.

### Phase 3 — Wrap-up

Write `final_report.md` with: summary of all iterations, best result achieved vs success criteria, what worked / what didn't, recommendations if research continues. Then stop scheduling new wake-ups.

## File structure (the state is ALL on disk)

```
<research_dir>/
  plan.md             # immutable after Phase 1 confirm, unless user re-opens planning
  state.json          # {session_id, phase, started_at, budget_hours, iteration_count, best_score, stop_reason_if_set}
  EXPERIMENTS.md      # append-only log, templates from experiment-ops-playbook
  CURRENT_STATUS.md   # single-page latest truth, templates from experiment-ops-playbook
  log.md              # append-only narrative of what main session did each turn
  artifacts/          # versioned experiment files, result JSONs (*_expN_vNN.py, etc.)
  final_report.md     # only exists after Phase 3
```

The rule: **if it's not in these files, it doesn't exist**. Never rely on session context to remember anything across iterations. A subagent launched five hours from now must be able to pick up the research from nothing but this folder.

## Durability stack (against terminal close / SSH disconnect / reboot)

Layer 1 (required): **tmux**. The wrapper `claude --autoresearch` (or user's launcher) should `exec tmux new-session -A -s autoresearch claude ...`. Closing the terminal detaches; `tmux attach -t autoresearch` resumes.

Layer 2 (required): **durable crons for wake-ups** via `CronCreate(durable: true)`. Survives Claude process restarts; auto-catches-up missed one-shots.

Layer 3 (recommended): **`SessionStart` hook** that detects unfinished autoresearch runs (scan `~/.claude/autoresearch/*/state.json` for `phase: running` with `started_at + budget_hours > now`) and surfaces them so the user can type "resume" to continue.

Layer 4 (optional): **@reboot crontab or systemd user unit** to auto-launch tmux+claude on machine boot. Only add when the user explicitly wants "machine reboots itself doesn't stop the research." Requires `loginctl enable-linger <user>` for the user session to persist without SSH.

Layer 5 (free, built-in): the MD files. Even if every process dies and every cron evaporates, a fresh `claude --autoresearch` pointed at the same `research_dir` can read `plan.md` + `CURRENT_STATUS.md` + `EXPERIMENTS.md` and resume with at most one iteration lost.

## Operating principles

- **State externalization > context persistence**. Never "remember" something by keeping it in the conversation. Write it to a MD file.
- **One change per iteration**. If a result moves, you must be able to name the cause. Batch changes hide signal.
- **Failures are data**. Log them with the same rigor as successes. Include exact error signatures, IDs, gate names.
- **Never re-try a direction already marked forbidden in `plan.md` → known_context** without first surfacing the contradiction to the user.
- **Respect the editable surface**. If `plan.md` says only file X can change, touching anything else is a bug, not a feature.
- **Budget is a hard limit**, not a suggestion. Stop means stop. Write `final_report.md` and return control.

## When NOT to use this skill

- Single-shot tasks ("fix this bug", "add this feature"): overkill.
- Research with no gate / no measurable outcome: you have no signal to iterate on.
- User wants to watch every step and approve each change manually: use plan mode or normal conversation, not autoresearch.
