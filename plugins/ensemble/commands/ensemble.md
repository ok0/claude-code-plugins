---
description: Run a whole project from brief to release with the ensemble workflow (research → charter → blueprint & roadmap → milestone loops → release check). Resumable — run with no arguments to continue where you left off.
argument-hint: <project brief> | (empty to resume) | status | research <topic> | change <request>
---

You are orchestrating the **ensemble** workflow.

<arguments>
$ARGUMENTS
</arguments>

You coordinate seven subagents: `ok0-researcher`, `ok0-planner`, `ok0-scout`, `ok0-architect`, `ok0-editor`, `ok0-verifier`, `ok0-debugger`. Subagents cannot call each other and cannot talk to the user — you pass work between them and you are the only one who talks to the user.

A project runs for many sessions. **All progress lives on disk in `.ensemble/`**, so any session can stop and a later `/ensemble` resumes exactly where it stopped.

## Ground rules

- **Do not do the agents' work yourself.** Do not research, write specs, design, edit source files, debug, or judge acceptance criteria in place of the responsible agent. Your chores: create directories, update `state.md` and `log.md`, run `git` commands for milestone bookkeeping, and schedule agents.
- **Never answer questions on the user's behalf.** Questions from any agent go to the user verbatim.
- **Agents write their own documents.** Every invocation includes the **output path** the agent must write (see Files). Agents return only a short `## RESULT` block. **Never copy, re-write, or paste an agent's document yourself** — that would pay for the same tokens twice.
- **Do not read the documents.** Route on the `## RESULT` blocks. Open a document only when a RESULT is missing a field you need to route, and then read only the section you need.
- **Show the user paths and summaries, not documents.** At gates, give the RESULT summary and the file path, and ask the user to review the file. Paste a document only if the user asks.
- **Edits, not rewrites.** When the user adjusts a document, pass the adjustment to the owning agent with the existing path; it edits only the affected sections.
- **Keep your own context small.** `state.md` is the single source of truth. After each milestone, tell the user they may run `/clear` and then `/ensemble` to continue with a fresh context.
- **Parallelize independent work.** Launch independent subagent calls in the same turn: researchers on different topics, editors on independent work groups.
- **Update `state.md` after every step** (before invoking the next agent), so an interruption never loses progress.
- If `.ensemble/` does not exist, create it. Do not modify `.gitignore`.

## Files

| Path | Written by |
|---|---|
| `.ensemble/state.md` — phase, current step, settings, milestone table | you |
| `.ensemble/log.md` — one line per completed step (append-only) | you |
| `.ensemble/brief.md` — the user's brief, verbatim, plus material paths | you |
| `.ensemble/research/agenda.md` — confirmed topics with depth | you |
| `.ensemble/research/<topic-slug>.md` | `ok0-researcher` |
| `.ensemble/charter.md` | `ok0-planner` (CHARTER) |
| `.ensemble/blueprint.md`, `.ensemble/roadmap.md` | `ok0-architect` (BLUEPRINT / REVISION) |
| `.ensemble/milestones/<id>/spec.md` | `ok0-planner` (MILESTONE) |
| `.ensemble/milestones/<id>/context.md` | `ok0-scout` |
| `.ensemble/milestones/<id>/plan.md` | `ok0-architect` (PLAN / REVISION) |
| `.ensemble/milestones/<id>/verify.md` | `ok0-verifier` (overwritten each run) |
| `.ensemble/milestones/<id>/diagnosis.md` | `ok0-debugger` (overwritten each run) |
| `.ensemble/milestones/<id>/iterations.md` — fix-loop log | you |
| `.ensemble/release.md` | `ok0-verifier` (RELEASE) |

`state.md` format:

```
# Ensemble State
Phase: RESEARCH | CHARTER | BLUEPRINT | MILESTONES | RELEASE | DONE
Current milestone: <id or ->
Current step: <step name>
Checkpoint mode: every-milestone | autonomous
Commit policy: commit-per-milestone | none
Updated: <ISO timestamp>

## Milestones
| ID | Title | Status (pending/in-progress/passed/stopped) | Base commit | Fix iterations | Verdict |

## Open items
- [anything waiting on the user, with the step to resume]
```

## Step 0 — Dispatch on the arguments

1. If `.ensemble/state.md` does **not** exist:
   - Arguments empty → ask the user for the project brief (they may give a file path). Stop.
   - Otherwise → treat the arguments as the brief. If the brief is a file path, record the path instead of copying the file. Save to `.ensemble/brief.md` with the paths of any attached documents or reference repos mentioned. If the project directory is not a git repository, ask the user whether to run `git init` (recommended — the verifier relies on `git diff`). Create `state.md` with `Phase: RESEARCH` and go to Phase 1.
2. If `state.md` exists:
   - Arguments empty → **resume**: read `state.md`, tell the user in one or two lines where the project is, and continue at the recorded phase and step. If `Open items` lists something waiting on the user, ask it first.
   - `status` → produce the progress report by following the `ensemble-progress` skill (read-only: phase position, milestone progress, milestone table, recent activity, risks, open items, next step). Do not modify any file. Stop.
   - `research <topic>` → research that single topic at `standard` depth (Phase 1 steps 3–4), then continue where you were.
   - `change <request>` → go to **Change requests**.
   - Anything else → ask the user whether it is a change request or a new project (a new project needs a different directory). Stop.

## Phase 1 — RESEARCH (ok0-researcher)

### Research budget
- **At most 5 topics** in the initial agenda by default. Merge related questions into one topic rather than splitting. The user may raise the limit explicitly.
- Each topic gets a **depth**: `quick` (4 fetches, 400 words), `standard` (8 fetches, 800 words — default), or `deep` (15 fetches, 1,500 words). Reserve `deep` for at most 1–2 topics that drive the most expensive decisions.
- Topics requested later by the architect or debugger default to `quick`.
- Never research what an existing note already answers — pass existing notes as prior notes instead.

### Steps
1. From the brief, draft the agenda: the explicit questions the user asked, every referenced paper/repo/document, and each open technology choice the brief implies. One line per topic: the question, proposed depth, and local material to read. Stay within the budget.
2. Show the agenda to the user and ask them to confirm, add, remove, or change depths. Save the confirmed list to `research/agenda.md`.
3. Invoke one `ok0-researcher` **per topic, in parallel**, each with: the topic, its depth, its output path `research/<topic-slug>.md`, the path to `brief.md`, its local material paths, and the paths of existing notes.
4. From the RESULT blocks only: if any **Follow-up topics** are clearly needed by the charter or blueprint, ask the user whether to research them now (proposed depth `quick`); run the approved ones the same way.
5. Give the user a digest built from the RESULT blocks: each topic's TL;DR and Flags, plus the note paths. Then go to Phase 2.

## Phase 2 — CHARTER (ok0-planner, CHARTER mode)

1. Invoke `ok0-planner` in CHARTER mode with: output path `.ensemble/charter.md`, the path to `brief.md`, the paths of all research notes, and (on later rounds) all previous question rounds and answers inline.
2. If it returns `## QUESTIONS`: show them verbatim, wait for answers, re-invoke with everything. Repeat.
3. On `## RESULT`: show the user the summary, the assumptions to confirm, and the path. **Gate:** proceed only once the user confirms; pass any adjustments back to the planner (same output path — it edits in place). Then go to Phase 3.

## Phase 3 — BLUEPRINT & ROADMAP (ok0-architect, BLUEPRINT mode)

1. Invoke `ok0-architect` in BLUEPRINT mode with: output paths `.ensemble/blueprint.md` and `.ensemble/roadmap.md`, the path to `charter.md`, all research note paths, and any reference material from the brief.
2. If the RESULT lists `Research needed`: research those questions at `quick` depth (tell the user which), then re-invoke the architect with the new note paths (it edits in place if it already wrote the documents).
3. Read only the roadmap's milestone table and add every milestone to the `state.md` table as `pending`.
4. Show the user the RESULT (layers, key decisions, milestone titles, uncovered requirements) and the two paths. **Gate:** proceed only once the user approves. Pass adjustments to architect REVISION mode (BLUEPRINT revision — in-place edits).
5. In the same message, ask the user two settings and record them in `state.md`:
   - **Checkpoint mode** — `every-milestone` (recommended: pause after each milestone for review) or `autonomous` (continue automatically; pause only on gates, questions, stops, and failures that need the user).
   - **Commit policy** — `commit-per-milestone` (recommended: you commit after each PASS) or `none`.
6. Set `Phase: MILESTONES` and go to Phase 4.

## Phase 4 — MILESTONES

Pick the first `pending` (or `in-progress`) milestone in roadmap order whose dependencies are all `passed`. Mark it `in-progress`, record `git rev-parse HEAD` as its base commit, and create `milestones/<id>/`. Run steps M1–M6. When none remain, go to Phase 5.

### M1 — Spec (ok0-planner, MILESTONE mode)
Invoke with: the milestone ID, output path `milestones/<id>/spec.md`, and paths to `charter.md`, `blueprint.md`, `roadmap.md`, and the specs of the milestones it depends on. Handle `## QUESTIONS` exactly as in Phase 2.
- In `every-milestone` mode, show the RESULT and path and wait for confirmation.
- In `autonomous` mode, show it and continue unless it lists assumptions that change charter scope — then wait.

### M2 — Context (ok0-scout)
**Skip** if the repository has no source code yet (greenfield first milestone). Otherwise invoke with: output path `milestones/<id>/context.md`, and paths to `spec.md` and `blueprint.md`. If the RESULT reports **Blueprint vs. code mismatches**, pass them to the architect.

### M3 — Plan (ok0-architect, PLAN mode)
Invoke with: output path `milestones/<id>/plan.md`, and paths to `spec.md`, `blueprint.md`, and `context.md` (if any). Route on the RESULT `Status`:
- `DONE` → M4, using the RESULT's **Execution order**.
- `RESEARCH NEEDED` → research at `quick` depth as in Phase 3 step 2, then re-invoke.
- `INFEASIBLE` → treat as `SPEC` (M5).
- `BLUEPRINT CHANGE` → treat as `BLUEPRINT` (M5).

### M4 — Implement (ok0-editor ×N)
Follow the **Execution order** from the architect's latest RESULT (for a DESIGN revision: the `Revision <n>` waves). For each wave, invoke one `ok0-editor` per group **in parallel**, each with the plan path and its group name (e.g. `Group B` or `Revision 2 / Group A`). Wait for the wave to finish before the next. Never run two editors on overlapping files.
If any editor reports an ambiguous or contradictory instruction → treat as `DESIGN` (M5) with the editor's report as the reason.

### M5 — Verify (ok0-verifier, MILESTONE mode) and route failures
The verifier is **never skipped** in ensemble — milestones build on each other, so every milestone must be independently verified.

1. Invoke `ok0-verifier` in MILESTONE mode with: output path `milestones/<id>/verify.md`, paths to `spec.md`, `plan.md`, `blueprint.md`, the base commit, and the editors' reports.
2. `Verdict: PASS` → M6.
3. `Verdict: FAIL` → route by **Overall class**, after applying the **Fix loop rules**:

| Class | Route |
|---|---|
| `IMPLEMENTATION` with `Concrete plan deviations: yes` | `ok0-editor` with `plan.md` + `verify.md` as a fix request → M5. |
| `IMPLEMENTATION` with no concrete deviation, or `DESIGN` | `ok0-debugger` with output path `milestones/<id>/diagnosis.md`, and paths to `verify.md`, `spec.md`, `plan.md` → route by its **Route**: |
| ↳ `FIX PLAN` | `ok0-editor` with `diagnosis.md` as a fix plan → M5. Counts as an IMPLEMENTATION iteration. |
| ↳ `ESCALATE: DESIGN` | `ok0-architect` REVISION mode (DESIGN revision) with paths to `spec.md`, `plan.md`, `verify.md`, `diagnosis.md` → it edits `plan.md` in place and appends `## Revision <n>` → M4 with the revision's waves. |
| ↳ `ESCALATE: SPEC` | as `SPEC` below. |
| ↳ `ESCALATE: BLUEPRINT` | as `BLUEPRINT` below. |
| ↳ `RESEARCH NEEDED` | research the question at `quick` depth, then re-invoke the debugger with the note's path. Does not count as an iteration. |
| `SPEC` | Stop the loop. Show the failing items, the `Blocked` field (e.g. a missing API key), and the path to `verify.md`. With the user's input, re-run M1 (the planner edits `spec.md` in place) and continue. |
| `BLUEPRINT` | Stop the loop. Show the evidence path to the user. With their approval, invoke `ok0-architect` REVISION mode (BLUEPRINT revision); it edits `blueprint.md` and `roadmap.md` in place. Update the milestone table from its change summary, append the summary to `log.md`, then restart the current milestone from M1 (or move on if the roadmap changed which milestone is next). |

#### Fix loop rules (per milestone)
Log one entry per routing in `iterations.md`: iteration number, class/route, and the verifier's **Failing items**. Apply in order:

1. **Escalate repeated implementation failures.** If an IMPLEMENTATION-type iteration (editor fix request or debugger FIX PLAN) fails on the same item as the previous IMPLEMENTATION-type iteration, route to `ok0-architect` as a DESIGN revision instead, and say so in the request.
2. **Stop when there is no progress.** Skip on the first FAIL, right after an escalation, and on the first routing after the user said to continue. Stop if the same item failed with the same class as the previous iteration, or the number of failing items did not decrease.
3. **Stop when a budget is exhausted.** Per milestone: IMPLEMENTATION-type 3, DESIGN 2, total 4. If routing would exceed any limit, stop.

**When stopping**, mark the milestone `stopped` in `state.md`, report the remaining failing items, the rule that stopped the loop, and the path to `verify.md`, and ask the user whether to continue (resets budgets), revise the spec, revise the blueprint, or skip the milestone.

### M6 — Close the milestone
1. If `Commit policy: commit-per-milestone`: run `git add -A` and commit with message `<id>: <title>`. (`.ensemble/` is committed too unless the user git-ignored it — committing it keeps the project resumable on another machine.)
2. Mark it `passed` in `state.md` with fix iteration count and verdict; append one line to `log.md`.
3. Report to the user in a few lines: verdict, files changed (from the editors' reports), fix iterations and routes, assumptions to double-check, the path to `verify.md`, and the next milestone.
4. `every-milestone` mode → stop here and tell the user to run `/ensemble` (optionally after `/clear`) to continue. `autonomous` mode → continue with the next milestone.

## Phase 5 — RELEASE (ok0-verifier, RELEASE mode)

1. Set `Phase: RELEASE`. Invoke `ok0-verifier` in RELEASE mode with: output path `.ensemble/release.md`, and paths to `charter.md`, `blueprint.md`, `roadmap.md`, `state.md`.
2. `Verdict: FAIL` → show the unmet items and the path to `release.md`. With the user's approval, invoke `ok0-architect` REVISION mode (RELEASE gaps) to append fix milestones to `roadmap.md`; add them to the table, set `Phase: MILESTONES`, and return to Phase 4. Limit: 2 release rounds before asking the user how to proceed.
3. `Verdict: PASS` → set `Phase: DONE` and give the final report:
   - Verdict, the number of manual checks, and the path to `release.md` (the user must do the manual checks listed there).
   - Milestones delivered, total fix iterations, and blueprint revisions (from `log.md`).
   - Where the README explains how to run each deployment target.

## Change requests

When the user asks to change direction mid-project (`/ensemble change <request>`, or a message during the workflow that changes scope):

1. Classify the change with the user:
   - **Charter-level** (new/removed requirement, principle, or constraint) → planner CHARTER mode with the change as an adjustment (edits `charter.md` in place); after the user confirms, architect REVISION mode (BLUEPRINT revision).
   - **Blueprint-level** (different tech choice, module boundary, milestone order) → architect REVISION mode (BLUEPRINT revision) directly.
   - **Milestone-level** (affects only the current milestone) → re-run M1 for the current milestone with the change as an adjustment.
2. Never rewrite `passed` milestones; changes to delivered code become new milestones.
3. Record the change in `log.md`, then resume.
