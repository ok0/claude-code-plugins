# claude-code-plugins

Reusable [Claude Code](https://claude.ai/code) agent collections.

## Install

### 1. Add the marketplace

In Claude Code, run:
```
/install trio
```

When prompted to add the marketplace `github:ok0/claude-code-plugins`, confirm it.

Or add the marketplace first:
```
/plugin marketplace add github:ok0/claude-code-plugins
```

Then install a specific plugin:
```
/install ensemble
/install quartet
/install triad
/install trio
/install duo
```

## Collections

### duo

A two-agent workflow separating design and implementation. Uses `opus` for complex planning and `haiku` for mechanical edits to save tokens.

| Agent | Model | Role |
|-------|-------|------|
| [`ok0-architect`](plugins/duo/agents/ok0-architect.md) | opus | Analyzes code and creates a **file-by-file change plan**. Never modifies files. |
| [`ok0-editor`](plugins/duo/agents/ok0-editor.md) | haiku | **Mechanically executes** the plan from `ok0-architect` without making design decisions. |

**Workflow:**
```
User → @ok0-architect "Add a login feature"
       ↓ (Outputs change plan)
User → @ok0-editor (Hand over the plan)
       ↓ (Edits code + runs build/checks)
```

### trio

A three-agent workflow adding a planning layer before design and implementation. The planner clarifies vague requests into structured specs before any code is read — saving `opus` tokens by eliminating ambiguity upfront.

| Agent | Model | Role |
|-------|-------|------|
| [`ok0-planner`](plugins/trio/agents/ok0-planner.md) | sonnet | Asks clarifying questions and produces a **structured spec (PRD)**. No code access. |
| [`ok0-architect`](plugins/trio/agents/ok0-architect.md) | opus | Reads code and turns the spec into a **file-by-file change plan**. Never modifies files. |
| [`ok0-editor`](plugins/trio/agents/ok0-editor.md) | haiku | **Mechanically executes** the plan from `ok0-architect` without making design decisions. |

**Workflow:**
```
User → @ok0-planner "Add a login feature"
       ↓ (Clarifies → Outputs spec)
User → @ok0-architect (Hand over the spec)
       ↓ (Reads code → Outputs change plan)
User → @ok0-editor (Hand over the plan)
       ↓ (Edits code + runs build/checks)
```

### triad

A three-agent workflow that adds an **independent verifier** without a separate planning step — for requests that are clear enough to go straight to design. The architect derives requirements and testable acceptance criteria from the request itself (asking clarifying questions only when it must), and the verifier also checks that those criteria actually cover the request. The `/triad` command orchestrates the whole flow.

| Agent | Model | Role |
|-------|-------|------|
| [`ok0-architect`](plugins/triad/agents/ok0-architect.md) | opus | Reads code, derives **requirements + acceptance criteria** from the request, and produces a **file-by-file change plan** with a **verification section**. Returns **clarifying questions** instead if the request is too ambiguous. Never modifies files. |
| [`ok0-editor`](plugins/triad/agents/ok0-editor.md) | haiku | **Mechanically executes** the plan. Never weakens tests and never judges the result. |
| [`ok0-verifier`](plugins/triad/agents/ok0-verifier.md) | sonnet | Checks **request coverage**, runs checks, judges each acceptance criterion **PASS/FAIL**, checks plan conformance, and classifies failures. Never modifies files. |

**Workflow:**
```
User → /triad "Add a login feature"
  architect ─ QUESTIONS ⇄ User          → .triad/plan.md
            (assumptions? → confirm with User)
  editor
  verifier                              → .triad/verify.md
     (skipped for small changes: ≤2 files, simple criteria, clean editor report)
     PASS → report
     FAIL → IMPLEMENTATION → editor → verifier
            DESIGN         → architect → editor → verifier
            SPEC           → stop, ask User
     (stops on no progress; budgets: IMPLEMENTATION 2, DESIGN 1, total 3;
      a repeated IMPLEMENTATION failure escalates to DESIGN)
```

Hand-off files are written to `.triad/` in your project. Add it to your project's `.gitignore` if you don't want to commit them.

### quartet

A four-agent workflow that closes the loop with an **independent verifier**. The agent that writes the code never judges it: the verifier checks every acceptance criterion and routes failures back to the right agent. The `/quartet` command orchestrates the whole flow.

| Agent | Model | Role |
|-------|-------|------|
| [`ok0-planner`](plugins/quartet/agents/ok0-planner.md) | sonnet | Returns either **clarifying questions** (relayed to you verbatim) or a **structured spec** with testable acceptance criteria. No code access. |
| [`ok0-architect`](plugins/quartet/agents/ok0-architect.md) | opus | Turns the spec into a **file-by-file change plan** plus a **verification section** mapping each acceptance criterion to a concrete check. Never modifies files. |
| [`ok0-editor`](plugins/quartet/agents/ok0-editor.md) | haiku | **Mechanically executes** the plan. Never weakens tests and never judges the result. |
| [`ok0-verifier`](plugins/quartet/agents/ok0-verifier.md) | sonnet | Runs checks, judges each acceptance criterion **PASS/FAIL**, checks plan conformance, and classifies failures. Never modifies files. |

**Workflow:**
```
User → /quartet "Add a login feature"
  planner   ─ QUESTIONS ⇄ User          → .quartet/spec.md
  architect                             → .quartet/plan.md
  editor
  verifier                              → .quartet/verify.md
     (skipped for small changes: ≤2 files, simple criteria, clean editor report)
     PASS → report
     FAIL → IMPLEMENTATION → editor → verifier
            DESIGN         → architect → editor → verifier
            SPEC           → stop, ask User
     (stops on no progress; budgets: IMPLEMENTATION 2, DESIGN 1, total 3;
      a repeated IMPLEMENTATION failure escalates to DESIGN)
```

Hand-off files are written to `.quartet/` in your project. Add it to your project's `.gitignore` if you don't want to commit them.

### ensemble

A seven-agent workflow for taking a **whole project from brief to release** — not just one change. It adds sourced web research before any decision, a project-level charter, a blueprint with decision records and a milestone roadmap, a verified loop per milestone (with parallel editors and a debugger), and a final release check against the charter. All state lives on disk, so `/ensemble` resumes across sessions.

| Agent | Model | Role |
|-------|-------|------|
| [`ok0-researcher`](plugins/ensemble/agents/ok0-researcher.md) | sonnet | Researches **one topic** (web + local docs/reference code) within a **depth budget** (quick/standard/deep) and writes a note where every claim has sources and **external/internal credibility** ratings. Never decides the design. |
| [`ok0-planner`](plugins/ensemble/agents/ok0-planner.md) | sonnet | **CHARTER** mode: project-wide requirements + project acceptance criteria. **MILESTONE** mode: per-milestone spec. Reads planning docs only, never code. |
| [`ok0-architect`](plugins/ensemble/agents/ok0-architect.md) | opus | **BLUEPRINT** mode: layers, modules, tech stack, ADRs, milestone roadmap. **PLAN** mode: file-by-file plan in **parallelizable work groups**. **REVISION** mode edits only the changed sections. Never modifies project source. |
| [`ok0-scout`](plugins/ensemble/agents/ok0-scout.md) | haiku | Builds a **fact-only context map** (files, signatures, conventions, commands) so opus doesn't spend tokens exploring. |
| [`ok0-editor`](plugins/ensemble/agents/ok0-editor.md) | haiku | **Mechanically executes** one work group; several run in parallel on disjoint files. |
| [`ok0-verifier`](plugins/ensemble/agents/ok0-verifier.md) | sonnet | **MILESTONE** mode: AC PASS/FAIL, regression, plan + blueprint conformance, failure class. **RELEASE** mode: judges every charter PAC end-to-end. Never modifies files. |
| [`ok0-debugger`](plugins/ensemble/agents/ok0-debugger.md) | sonnet | Reproduces a failure, finds the root cause, and returns a **small fix plan** or an escalation (DESIGN / SPEC / BLUEPRINT). |

**Workflow:**
```
User → /ensemble "<project brief>"          (later: /ensemble to resume)
  1 RESEARCH   agenda ⇄ User → researcher ×N (parallel)   → .ensemble/research/*.md
  2 CHARTER    planner ─ QUESTIONS ⇄ User                 → .ensemble/charter.md      [User approves]
  3 BLUEPRINT  architect (+ research on demand)           → blueprint.md, roadmap.md  [User approves]
  4 MILESTONES for each milestone in dependency order:
       planner → scout → architect → editor ×N (parallel waves) → verifier
         PASS → commit, next milestone (pause or autonomous)
         FAIL → IMPLEMENTATION → editor → verifier
                DESIGN         → debugger → FIX PLAN → editor
                                          → ESCALATE → architect → editor
                SPEC           → stop, ask User
                BLUEPRINT      → stop, User approves → architect revises blueprint/roadmap
         (stops on no progress; budgets per milestone: IMPLEMENTATION 3, DESIGN 2, total 4)
  5 RELEASE    verifier checks every charter PAC end-to-end
         FAIL → architect appends fix milestones → 4
         PASS → DONE
```

Other commands: `/ensemble status`, `/ensemble research <topic>`, `/ensemble change <request>`.

**Token discipline:** each agent writes its own document under `.ensemble/` and returns only a short `RESULT` block, so the orchestrating session never copies or re-reads documents; revisions are in-place edits of only the changed sections; research defaults to at most 5 topics with per-topic fetch and length budgets.

Project state is written to `.ensemble/` in your project. Committing it keeps the project resumable on any machine; add it to `.gitignore` if you prefer not to.

## Structure

```text
claude-code-plugins/
├── .claude-plugin/
│   └── marketplace.json      # Marketplace registry
├── README.md
└── plugins/
    ├── duo/
    │   ├── .claude-plugin/
    │   │   └── plugin.json   # Plugin metadata
    │   └── agents/
    │       ├── ok0-architect.md
    │       └── ok0-editor.md
    ├── trio/
    │   ├── .claude-plugin/
    │   │   └── plugin.json   # Plugin metadata
    │   └── agents/
    │       ├── ok0-planner.md
    │       ├── ok0-architect.md
    │       └── ok0-editor.md
    ├── triad/
    │   ├── .claude-plugin/
    │   │   └── plugin.json   # Plugin metadata
    │   ├── agents/
    │   │   ├── ok0-architect.md
    │   │   ├── ok0-editor.md
    │   │   └── ok0-verifier.md
    │   └── commands/
    │       └── triad.md      # /triad orchestration
    ├── quartet/
    │   ├── .claude-plugin/
    │   │   └── plugin.json   # Plugin metadata
    │   ├── agents/
    │   │   ├── ok0-planner.md
    │   │   ├── ok0-architect.md
    │   │   ├── ok0-editor.md
    │   │   └── ok0-verifier.md
    │   └── commands/
    │       └── quartet.md    # /quartet orchestration
    └── ensemble/
        ├── .claude-plugin/
        │   └── plugin.json   # Plugin metadata
        ├── agents/
        │   ├── ok0-researcher.md
        │   ├── ok0-planner.md
        │   ├── ok0-architect.md
        │   ├── ok0-scout.md
        │   ├── ok0-editor.md
        │   ├── ok0-verifier.md
        │   └── ok0-debugger.md
        └── commands/
            └── ensemble.md   # /ensemble orchestration
```
