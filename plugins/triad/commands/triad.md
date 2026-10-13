---
description: Run the triad workflow (architect → editor → verifier) on a request, routing verification failures back to the right agent.
argument-hint: <request>
---

You are orchestrating the **triad** workflow for the following request:

<request>
$ARGUMENTS
</request>

You coordinate three subagents: `ok0-architect`, `ok0-editor`, `ok0-verifier`. Subagents cannot call each other and cannot talk to the user — you pass work between them and you are the only one who talks to the user. There is no planner: the architect derives requirements and acceptance criteria directly from the request.

## Ground rules

- **Do not do the agents' work yourself.** Do not design the plan, write acceptance criteria, edit source files, or judge acceptance criteria in place of the responsible agent.
- **Never answer questions on the user's behalf.** Questions from the architect go to the user verbatim.
- **Hand off through files.** Save each agent's final output under `.triad/` in the project root and pass file paths (not pasted content) to agents.
  - `.triad/request.md` — the original request, plus every question round and the user's answers
  - `.triad/plan.md` — current plan (overwrite on revision)
  - `.triad/verify.md` — latest verification report (overwrite each run)
- If `.triad/` does not exist, create it. Do not modify `.gitignore`.

## Step 1 — Plan (ok0-architect)

1. Save the request to `.triad/request.md`.
2. Invoke `ok0-architect`, pointing it at `.triad/request.md`.
3. If the output starts with `## QUESTIONS`: show the questions to the user verbatim, wait for their answers, append the questions and answers to `.triad/request.md`, and re-invoke the architect. Repeat.
4. When the output starts with `## PLAN`: save its `## Plan for ok0-editor` section to `.triad/plan.md`.
5. If the architect flags any part of the request as infeasible or conflicting with the codebase, stop and show this to the user. Resume from Step 1 with the user's input.
6. If the plan lists any **Assumptions** (anything other than "None"), show the Requirements, Acceptance Criteria, and Assumptions to the user and ask them to confirm or adjust. Append adjustments to `.triad/request.md` and re-invoke the architect. Proceed only once the user confirms. If there are no assumptions, proceed without asking.

## Step 2 — Implement (ok0-editor)

1. Invoke `ok0-editor`, pointing it at `.triad/plan.md`.
2. If the editor reports that the plan is ambiguous or contradicts the code, treat it as a `DESIGN` failure: go to Step 4 with the editor's report as the reason.

## Step 3 — Verify (ok0-verifier)

**Skip the verifier** only on the first pass (never after a fix in Step 4), and only if ALL of the following hold:
- The plan changes or creates **at most 2 files**.
- Every acceptance criterion is **simple**: its check in the plan's Verification section is a single command or a single inspection with an unambiguous expected result.
- The editor reported **no** failing build, type-check, or test, and **no** plan discrepancy.

If skipped, do not judge the acceptance criteria yourself. Go to Step 5 and mark the verdict as `SKIPPED`.

Otherwise:
1. Invoke `ok0-verifier`, pointing it at `.triad/request.md` and `.triad/plan.md`, and include the editor's report.
2. Save its report to `.triad/verify.md`.
3. If the verdict is `PASS`, go to Step 5.
4. If the verdict is `FAIL`, go to Step 4.

## Step 4 — Route the failure

Use the verifier's **Overall class** (or `DESIGN` for an editor-reported plan problem). Before routing, apply the **Fix loop rules** below — they may change the class or stop the loop.

| Class | Route |
|---|---|
| `IMPLEMENTATION` | Invoke `ok0-editor` with `.triad/plan.md` and `.triad/verify.md` as a fix request. Then go to Step 3. |
| `DESIGN` | Invoke `ok0-architect` with `.triad/request.md`, `.triad/plan.md`, and `.triad/verify.md` (or the editor's report) as a revision request. Overwrite `.triad/plan.md` with the revised plan. Then go to Step 2. |
| `SPEC` | Stop. Show the failing criteria and the verifier's evidence to the user. Append the user's input to `.triad/request.md` and resume from Step 1. |

### Fix loop rules

Keep a log in `.triad/iterations.md`: one entry per pass through Step 4 with the iteration number, the class routed, and the list of failing items (failing AC#s, request-coverage gaps, and failing project-wide checks). Compare each new FAIL against the previous entry. Apply these rules in order:

1. **Escalate repeated implementation failures.** If the class is `IMPLEMENTATION` and the same failing item was also routed as `IMPLEMENTATION` in the previous iteration, change the class to `DESIGN` — the editor failing twice on the same item means the plan is not literal enough. Tell the architect this in the revision request.
2. **Stop when there is no progress.** Stop if either holds. Skip this check on the first FAIL, when rule 1 just escalated, and on the first routing after the user said to continue:
   - The same failing item failed with the same class in the previous iteration (after rule 1 has been applied).
   - The number of failing items did not decrease compared to the previous iteration.
3. **Stop when a budget is exhausted.** Per run, at most:

   | Budget | Limit |
   |---|---|
   | `IMPLEMENTATION` iterations | 2 |
   | `DESIGN` iterations (including escalations and editor-reported plan problems) | 1 |
   | Total iterations | 3 |

   If routing would exceed any limit, stop instead.

**When stopping**, go to Step 5 with the remaining failures, the rule that stopped the loop, and ask the user whether to continue. If the user says to continue, reset all budgets to zero and resume routing from the current failure.

## Step 5 — Report to the user

Summarize concisely:
- Final verdict (PASS, FAIL with remaining failures and their class, or SKIPPED).
- The acceptance criteria table from the latest verification report. If the verifier was skipped, say so explicitly, state that the acceptance criteria were **not independently verified**, and list each AC with the check the user can run themselves (from the plan's Verification section).
- Files changed.
- Fix iterations used, what each one routed to, and any escalation from `IMPLEMENTATION` to `DESIGN`.
- If the loop stopped early: which rule stopped it, and the question whether to continue.
- Any assumptions from the plan the user should double-check.
