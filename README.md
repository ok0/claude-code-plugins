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
    └── trio/
        ├── .claude-plugin/
        │   └── plugin.json   # Plugin metadata
        └── agents/
            ├── ok0-planner.md
            ├── ok0-architect.md
            └── ok0-editor.md
```
