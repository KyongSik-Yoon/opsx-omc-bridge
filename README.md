# opsx-omc-bridge

[한국어](README.ko.md)

A Claude Code plugin that automatically bridges **OpenSpec** planning artifacts to **Oh My ClaudeCode (OMC)** execution modes and orchestration strategies — eliminating the manual handoff between spec preparation and implementation.

## Problem

After preparing specs with OpenSpec (`proposal.md`, `design.md`, `tasks.md`), delegating implementation to OMC requires repetitive manual steps:

1. Read `tasks.md` and extract pending items
2. Summarize constraints from `design.md`
3. Choose the right OMC mode
4. Compose a context-rich prompt

This plugin automates the entire handoff.

## Skills

| Skill | Use Case | OMC Mode / Strategy | Trigger Examples |
|-------|----------|----------------------|-----------------|
| **quick** | ≤ 3 tasks, single domain | `autopilot` | "implement quickly", "apply now" |
| **deploy** | 4+ tasks, multi-phase/domain | `ralph` by default, `hybrid` for sequential phases with parallelizable segments, `team` for mostly independent work | "run with team", "delegate to ralph" |

### quick

Best for straightforward changes. Reads OpenSpec artifacts, assembles context, and delegates to OMC `autopilot` for linear execution.

### deploy

Best for complex changes. Performs a preflight pass, prefers a compacted execution brief when present, analyzes task structure (phase count, dependencies, domain spread), and automatically selects the optimal execution strategy:

| Condition | OMC Mode | Rationale |
|-----------|----------|-----------|
| Sequential phases with strong dependencies | `ralph` | Step-by-step with verification |
| Independent tasks within a phase | `team N:executor` | Parallel processing |
| Single phase, 4+ tasks | `team N:executor` | Simple parallelization |
| Refactoring requiring iterative verification | `ralph` | Repeats until architect approval |
| Sequential phases + parallelizable tasks within | `ralph` + `team` per phase | Hybrid approach with phase gates |

### Preflight and Compaction

Before delegating, `deploy` should:

1. Resolve the canonical artifact set for the target change
2. Prefer a compacted execution brief when one exists, such as `implementation-brief.md`, `brief.md`, or another user-designated handoff doc
3. Treat `proposal.md`, `design.md`, delta specs, and `tasks.md` as source-of-truth evidence behind that brief
4. Extract a phase graph and mark which phases are safe to parallelize
5. Present the recommended execution plan before delegation

## Workflow

```
/opsx:propose "feature description"   ← Prepare spec with OpenSpec
        ↓
/opsx:ff                              ← Generate all artifacts
        ↓
"implement quickly"                   ← quick skill → autopilot
or
"run with team"                       ← deploy skill → ralph / hybrid / team
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
