---
name: ensemble-progress
description: Report the progress and status of an in-flight ensemble project (read-only). Use when the user asks about project progress, status, completion rate, what's done or left, what's blocked, or "where are we" in a project that has a `.ensemble/` directory. Triggers include "진행률", "진행상황", "현재 어디까지 했어", "progress report", "project status".
---

# Ensemble Progress Report

Read-only report on an ensemble project. **Never modify any file, invoke any agent, or run git commands that change state.** To continue work, tell the user to run `/ensemble`.

## Sources (read in this order, stop when you have enough)

1. `.ensemble/state.md` — phase, current milestone/step, settings, milestone table, open items. **Primary source.**
2. `.ensemble/log.md` — completed-step history (use the last ~10 lines, plus counts of blueprint revisions).
3. `.ensemble/roadmap.md` — only the milestone table, for titles/dependencies if `state.md` lacks them.
4. For the in-progress milestone only: `milestones/<id>/iterations.md` (fix-loop history) and the `Verdict`/`Failing items` header of `verify.md` if present.
5. `.ensemble/release.md` — only when Phase is `RELEASE` or `DONE` (verdict + manual checks).

Do not read charter, blueprint, research notes, specs, or plans unless the user asks for detail. If `.ensemble/state.md` does not exist, say no ensemble project was found in this directory and suggest `/ensemble <brief>`.

## Compute

- **Phase position**: RESEARCH → CHARTER → BLUEPRINT → MILESTONES → RELEASE → DONE (mark the current one).
- **Milestone progress**: `passed / total` and percentage. Count `stopped` and `in-progress` separately.
- **Overall estimate** (state it is a rough estimate): pre-milestone phases count as done once passed; during MILESTONES use passed/total; RELEASE ≈ 95%; DONE = 100%.
- **Effort signals**: total fix iterations, milestones needing more than 1 iteration, blueprint revisions.
- **Risks**: `stopped` milestones, milestones near the fix budget (IMPLEMENTATION 3, DESIGN 2, total 4), open items waiting on the user.

## Output format

Respond in the user's language (Korean if they wrote Korean). Keep it compact:

```
## 📊 <project name> 진행 현황
**Phase**: RESEARCH ✅ → CHARTER ✅ → BLUEPRINT ✅ → **MILESTONES** → RELEASE → DONE
**진행률**: ▓▓▓▓▓░░░░░ 50% (3/6 milestones passed) — 대략적 추정
**현재**: <milestone id/title> — <step> (fix iteration n)

| ID | Title | Status | Fix iter | Verdict |
|----|-------|--------|----------|---------|

### 최근 활동
- <last few log lines>

### ⚠️ 주의 / 사용자 대기 항목
- <stopped milestones, open items, budget warnings — or "없음">

### 다음 단계
<one line: what happens next, e.g. run `/ensemble` to continue>
```

Keep status values exactly as recorded in `state.md`. Report facts from the files only — do not guess about work not recorded there. If `Updated` is old, mention how long ago the state was last updated.
