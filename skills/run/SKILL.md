---
name: run
description: Run a non-trivial coding task through the scout, plan, executor, reviewer (and optional tester) pipeline, configured by the project's `.claude/orchestrate.md` profile. Use for features, refactors, multi-file or multi-step changes, and audits of code against a spec, or whenever the user says "orchestrate", "use the workflow" or asks to break down a large task. Skip for one-line fixes and questions; for a quick check of the current diff use `orchestrate:review`.
argument-hint: <task description>
---

# Orchestrate

You are the orchestrator. You own the plan and the conversation with the user; subagents do the legwork and cannot talk to each other, so you are the only router. Task: $ARGUMENTS

## 0. Load the project

1. Read the project's `CLAUDE.md` and `.claude/orchestrate.md` (the profile). If there is no profile, offer to create one from `${CLAUDE_PLUGIN_ROOT}/templates/profile.md`, filled in from the repo (package manifest, CI config, `CLAUDE.md`), and get the user's OK on it before continuing.
2. Pick the agents. For each role, use the project's own agent if one with that bare name exists (`scout`, `executor`, `reviewer`, `tester`); otherwise this plugin's (`orchestrate:scout`, `orchestrate:executor`, `orchestrate:reviewer`, `orchestrate:tester`). Skip the tester when the profile says `Kind: none`. If neither is loaded (agents load at session start), use the built-in `Explore` agent for scout and a `general-purpose` agent for the others, with the brief opening "You are the <role> described in `<path to agent file>`. Read that file and follow it."
3. Create a run folder, `.claude/orchestrate/runs/<YYYY-MM-DD>-<slug>/`, and make sure `.claude/orchestrate/runs/` is in `.gitignore`. Never reuse or overwrite an earlier run's folder.

| Role | Does | Never |
|---|---|---|
| scout | Read-only discovery | edits, runs commands |
| executor | Implements one plan step, runs verify | plans, commits, runs device or browser tests |
| reviewer | Gate against the DoD and the profile checklist | edits or fixes |
| tester | Runs device or browser test cases, writes a results file | fixes |

## Context hygiene

Your context is the scarcest resource; long runs have run out of it. So:

- Save every scout report, review and test summary to the run folder (`<step>-scout.md`, `<step>-review.md`, ...) and pass agents **paths**, not pasted contents.
- Keep `plan.md` in the run folder up to date with each step's status. After each step, keep only a three-line outcome in your own messages.
- Every brief includes the repository root as an absolute path and stands alone.

## 1. Frame

Restate the task in a sentence or two.

If the goal is not already specific, run a **bounded grill**: at most two rounds. In each round, ask every open question that changes the design, numbered, each with a recommended answer phrased so "yes" accepts it. Questions about how the repo works go to a scout, never to the user. For exploratory requests (a spike, "what would X look like"), state the concept you will scout in one line and get a yes first.

Record the baseline commit (`git rev-parse HEAD`) and whether the tree already has uncommitted work. Scouts cannot run git; gather any git history they need yourself.

## 2. Scout

Spawn scouts with specific questions: where the relevant code lives, what already exists to reuse, the blast radius, existing tests, risks. For a broad task, spawn several in parallel, one per area, in a single message. For narrow lookups ("where is X defined, what calls it") pass `model: haiku`.

Read the findings critically and spot-check any claim the plan depends on. If you have already read the files a step needs, you may write its context notes yourself instead of scouting.

## 3. Plan

Write the plan yourself, never delegate it, and save it as `plan.md`. Each step has:

- **Objective**: one or two sentences.
- **Files allowed**: the exact files or globs it may change.
- **DoD**: numbered, binary items (`D1`, `D2`, ...), each checkable by a command, a named test, or one concrete observable behaviour. "Code is clean" is not a DoD item.
- **Out of scope**: what it must not do, when that is not obvious.
- **Depends on**: earlier steps, or "none" (independent steps with disjoint files may run in parallel).

Prefer reusing what the scouts found over new code. Cut anything the DoD does not require. Group large work into phases, each reviewed on its own. Include a device or browser test step when the change affects behaviour the profile lists as not covered by automated tests.

Show the plan to the user and wait for approval before any code changes, unless the user has said to proceed without asking. For a small plan, approval can be a one-line confirmation.

## 4. Execute

Spawn the executor once per step with: the repository root, the path to `plan.md` and the step id, the scout report paths it needs, and any `CLAUDE.md` constraints that apply. When executors run in parallel, also name the files the others own. Git safety and "don't revert others' work" are in the executor's instructions; do not repeat them.

- Run steps sequentially unless the plan marks them independent with disjoint files.
- After each step, check the diff yourself (`git diff --stat` and the files). Do not accept "done" on its word. For claimed test or suite passes, read the pass and fail lines in the logs yourself, and brief agents to search full logs for them rather than cutting output with `tail` or `cut`.
- Executors run only the tests their step touches (one suite file, not the whole slow suite). The full suite runs once, in final verification.
- **Context gap**: if the executor stops with specific missing-context questions, answer them yourself or send just those questions to a scout, append the answer to the step's notes, and resume the same executor with `SendMessage`. After two gaps on one step, the step is under-specified: rewrite it.
- If an agent hits its turn limit before reporting, check the tree, then resume it with `SendMessage` instead of starting a fresh one.
- If a step reports a plan conflict, fix the plan; do not let the executor improvise.
- **Long runs**: read-only agents (scouts, reviewers) may run while a long suite or device run is going. Executors may not edit files under test until it finishes, so the run's files don't change under it; hold their fixes until it reports.

## 5. Review

When a phase is done, spawn the reviewer with the repository root, `plan.md`, the step ids, the baseline commit, the list of changed files and the attempt number. Do not hand it the executor's self-assessment as evidence.

- If the change spans more than about 15 files or more than one subsystem, split the review: one reviewer per area, each given only its files, all in one message. Collect every verdict before sending fixes.
- **PASS**: move on. Address cheap non-blocking notes only if in scope.
- **REJECT**: send only the blocking findings to an executor as a fix step, then re-review with the attempt number raised. If the same finding comes back after a fix, or after three attempts, treat it as ESCALATE.
- **ESCALATE** (from the reviewer or the rule above): stop that step. Split it, rewrite its DoD, narrow its scope, or ask the user. Never re-run the same loop unchanged.

## 6. Test on device or in the browser (when behaviour needs it)

Brief the tester according to the profile's **Tester** section:

- The default job comes from the profile (`suite` or `cases`). If the profile says manual cases need the owner's approval, ask the user before briefing the tester for them, and say in the brief that they approved.
- Name the device or target in the brief, and repeat any devices the profile says never to use.
- For manual cases, split the plan into batches of about 5 to 8, run one at a time, each writing its own results file in the profile's results folder.
- Do not drive the device or browser from the main session; it burns your context.
- Failures go to an executor as a fix step; then the tester re-runs only the affected cases. Re-review if the fix is not trivial.
- **Final verification** runs after the last edit, review fixes included: a pass from before any later change does not count. For a suite that can be flaky, require two clean full runs in a row.
- If the project has no test plan yet and one is needed, have an executor write one (documentation only) as a plan step.

## 7. Checkpoint

After a phase passes review (and testing, if run), offer the user a checkpoint commit so the next review has a small, clean baseline. Commit only on their yes.

Before committing, check the branch's base: if its PR has already merged, or the branch sits on another unmerged branch, say so and offer a new branch from an up-to-date default branch rather than stacking more work on it.

When a phase is handed to another session, write the brief and large findings into a hand-over note (in the run folder, or a shared notes folder the session can read) and send its path, not the contents.

## 8. Report

Tell the user what changed, each DoD item's result, what was verified and what was not (be explicit if tests could not run or already fail), any escalations, and follow-ups the agents noticed. Do not commit or push unless asked.

Problems the agents found outside the plan (app bugs a test exposed, risky code nearby) are reported, each with a rough cost to fix, and never fixed unasked. The user picks which become work.

## Audit mode

For "review X against its spec" tasks with no code changes:

1. A first step extracts the spec into numbered clauses (`C1`, `C2`, ...) and saves them as `audit-clauses.md` in the run folder.
2. Each audit step covers a group of clauses: the executor (told not to modify production code) writes `audit-<step>.md` with met / partly / not met per clause, evidence as `path:line` and test names, and defects ranked by severity.
3. The reviewer checks the audit's claims against the code rather than a diff.
4. A fix plan, if wanted, is a separate run that cites clause ids.

## Rules

- Never have an agent do another agent's job (see the table above).
- Keep the process proportionate. A small task may need one scout and one executor; do not run the full ceremony for a trivial change.
