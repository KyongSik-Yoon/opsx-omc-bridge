# opsx-omc-bridge

[한국어](README.ko.md)

A Claude Code plugin that automatically bridges **OpenSpec** planning artifacts to **Oh My ClaudeCode (OMC)** execution modes — eliminating the manual handoff between spec preparation and implementation.

## Problem

After preparing specs with OpenSpec (`proposal.md`, `design.md`, `tasks.md`), delegating implementation to OMC requires repetitive manual steps:

1. Read `tasks.md` and extract pending items
2. Summarize constraints from `design.md`
3. Choose the right OMC mode
4. Compose a context-rich prompt

This plugin automates the entire handoff.

## Skills

| Skill | Use Case | OMC Mode | Trigger Examples |
|-------|----------|----------|-----------------|
| **quick** | ≤ 3 tasks, single domain | `autopilot` | "implement quickly", "apply now" |
| **deploy** | 4+ tasks, multi-phase/domain | `team` or `ralph` (auto-selected) | "run with team", "delegate to ralph" |

### quick

Best for straightforward changes. Reads OpenSpec artifacts, assembles context, and delegates to OMC `autopilot` for linear execution.

### deploy

Best for complex changes. Analyzes task structure (phase count, dependencies, domain spread) and automatically selects the optimal execution strategy:

| Condition | OMC Mode | Rationale |
|-----------|----------|-----------|
| Sequential phases with strong dependencies | `ralph` | Step-by-step with verification |
| Independent tasks within a phase | `team N:executor` | Parallel processing |
| Single phase, 4+ tasks | `team N:executor` | Simple parallelization |
| Refactoring requiring iterative verification | `ralph` | Repeats until architect approval |
| Sequential phases + parallelizable tasks within | `ralph` + `team` per phase | Hybrid approach |

## Workflow

```
/opsx:propose "feature description"   ← Prepare spec with OpenSpec
        ↓
/opsx:ff                              ← Generate all artifacts
        ↓
"implement quickly"                   ← quick skill → autopilot
or
"run with team"                       ← deploy skill → team/ralph
        ↓
/opsx:verify                          ← Verify against spec
        ↓
/opsx:archive                         ← Archive completed change
```

## Installation

```bash
cd /path/to/opsx-omc-bridge
claude /plugin install --plugin-dir .
```

## Prerequisites

- **OpenSpec** initialized in the target project (`openspec/` directory)
- **Oh My ClaudeCode** plugin installed
- **tmux** session active (required for `team` mode in deploy)

## Project Structure

```
opsx-omc-bridge/
├── .claude-plugin/
│   └── plugin.json
├── skills/
│   ├── quick/
│   │   └── SKILL.md
│   └── deploy/
│       └── SKILL.md
├── README.md
└── README.ko.md
```

## License

MIT
