# orchestrate

A Claude Code plugin that runs multi-step coding tasks through a scout → plan → executor → reviewer (→ tester) pipeline. The process lives here; each project supplies a short profile.

## Contents

- `skills/run/`: the full pipeline (`/orchestrate:run <task>`), including an audit mode for checking code against a spec.
- `skills/review/`: a quick project-aware review of the current diff (`/orchestrate:review`), for work done outside the pipeline.
- `agents/`: `scout`, `executor`, `reviewer`, `tester` (an automated device or browser suite, or manual cases on an Android emulator, iOS Simulator or local dev server; the profile's `Kind` picks the mobile or web rules).
- `templates/profile.md`: the per-project profile template.

## How it works

- Each step in a plan has allowed files and numbered, binary DoD items. The executor reports per DoD item and stops with a **context gap** instead of guessing; the reviewer re-runs the checks itself.
- The reviewer rejects only on evidenced defects, never on style or a different valid design, and from the second attempt only on unfixed findings or regressions. A repeated failure or three attempts escalate to a plan change instead of another loop.
- Scout reports, reviews and test results are saved under `.claude/orchestrate/runs/<date>-<slug>/` (gitignored) and passed to agents by path, to keep the main session's context small.
- Testers write each case's result as they go, so a turn limit never loses work.

## Install

From an interactive `claude` terminal:

```
/plugin marketplace add /Users/troyanderson/projects/personal/claude-orchestrate
/plugin install orchestrate@troy-plugins
```

After editing the plugin, run `/plugin marketplace update troy-plugins` and start a new session (agents load at session start).

## Use in a project

1. Add `.claude/orchestrate.md` (copy `templates/profile.md`, or let `/orchestrate:run` draft it).
2. Optionally override a role by adding a project agent with the bare name (`.claude/agents/reviewer.md`, `tester.md`, ...). The skills prefer project agents over the plugin's.
3. Run `/orchestrate:run <task>`, or `/orchestrate:review` before a commit.

## Not enforced (yet)

The rules (allowed files, read-only reviewer, no destructive git) are instructions, not hooks. An earlier hook-based setup showed that guards which also block reads (`grep`, `git log`, reading `.env.example`) cost more than they save; if enforcement is added, it should block writes and destructive commands only.
