---
description: Run the trio workflow (planner → architect → editor) on a request.
argument-hint: <request>
---

You are orchestrating the **trio** workflow for the following request:

<request>
$ARGUMENTS
</request>

You coordinate three subagents: `ok0-planner`, `ok0-architect`, `ok0-editor`. Subagents cannot call each other and cannot talk to the user — you pass work between them and you are the only one who talks to the user.

## Ground rules

- **Do not do the agents' work yourself.** Do not write the spec, design the plan, or edit source files in place of the responsible agent.
- **Never answer questions on the user's behalf.** Questions from the planner go to the user verbatim.
- **Hand off through files.** Save each agent's final output under `.trio/` in the project root and pass file paths (not pasted content) to agents.
  - `.trio/request.md` — the original request, plus every question round and the user's answers
  - `.trio/spec.md` — final spec from the planner (overwrite on revision)
  - `.trio/plan.md` — current plan from the architect (overwrite on revision)
- If `.trio/` does not exist, create it. Do not modify `.gitignore`.

## Step 1 — Specify (ok0-planner)

1. Save the request to `.trio/request.md`.
2. Invoke `ok0-planner`, pointing it at `.trio/request.md`.
3. If the planner returns clarifying questions instead of a spec, show them to the user verbatim, wait for answers, append the questions and answers to `.trio/request.md`, and re-invoke the planner. The planner allows at most 2 clarification rounds; after that it must produce a spec with documented Assumptions.
4. When the planner returns a spec, save its `## Spec for ok0-architect` section to `.trio/spec.md`.
5. If the spec lists any **Assumptions**, show them to the user and ask them to confirm or adjust. Append adjustments to `.trio/request.md` and re-invoke the planner. Proceed only once the user confirms. If there are no assumptions, proceed without asking.

## Step 2 — Plan (ok0-architect)

1. Invoke `ok0-architect`, pointing it at `.trio/spec.md`.
2. If the architect flags any requirement as infeasible or conflicting with the codebase, stop and show this to the user. Append their input to `.trio/request.md` and resume from Step 1.
3. Save the architect's `## Plan for ok0-editor` section to `.trio/plan.md`.

## Step 3 — Implement (ok0-editor)

1. Invoke `ok0-editor`, pointing it at `.trio/plan.md`.
2. If the editor reports that the plan is ambiguous or contradicts the code, send the editor's report to `ok0-architect` along with `.trio/spec.md` and `.trio/plan.md` as a revision request, overwrite `.trio/plan.md`, and re-invoke the editor. Do this at most once; if it fails again, stop and report to the user.

## Step 4 — Report to the user

Summarize concisely:
- Files changed.
- Build/type-check/test results as reported by the editor.
- The spec's Acceptance Criteria, each with a check the user can run themselves (there is no independent verifier in this workflow, so state that they are **not independently verified**).
- Any assumptions from the spec and any plan revisions that occurred.
