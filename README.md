# claudecode-plugins

Reusable [Claude Code](https://claude.ai/code) agent collections.

## Install

Install all agents:
```bash
/plugin install <path-to-this-repo>
```

Install a specific group (e.g., `duo`):
```bash
/plugin install <path-to-this-repo>/agents/duo
```
*(Each group has its own `plugin.json` for independent installation.)*

## Collections

### duo

A two-agent workflow separating design and implementation. Uses `opus` for complex planning and `haiku` for mechanical edits to save tokens.

| Agent | Model | Role |
|-------|-------|------|
| [`ok0-architect`](agents/duo/ok0-architect.md) | opus | Analyzes code and creates a **file-by-file change plan**. Never modifies files. |
| [`ok0-editor`](agents/duo/ok0-editor.md) | haiku | **Mechanically executes** the plan from `ok0-architect` without making design decisions. |

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
| [`ok0-planner`](agents/trio/ok0-planner.md) | sonnet | Asks clarifying questions and produces a **structured spec (PRD)**. No code access. |
| [`ok0-architect`](agents/trio/ok0-architect.md) | opus | Reads code and turns the spec into a **file-by-file change plan**. Never modifies files. |
| [`ok0-editor`](agents/trio/ok0-editor.md) | haiku | **Mechanically executes** the plan from `ok0-architect` without making design decisions. |

**Workflow:**
```
User → @ok0-planner "Add a login feature"
       ↓ (Clarifies → Outputs spec)
User → @ok0-architect (Hand over the spec)
       ↓ (Reads code → Outputs change plan)
User → @ok0-editor (Hand over the plan)
       ↓ (Edits code + runs build/checks)
```

## Structure

```text
claudecode-plugins/
├── plugin.json
├── README.md
└── agents/
    ├── duo/
    │   ├── plugin.json
    │   ├── ok0-architect.md
    │   └── ok0-editor.md
    └── trio/
        ├── plugin.json
        ├── ok0-planner.md
        ├── ok0-architect.md
        └── ok0-editor.md
```
