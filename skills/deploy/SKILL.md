---
name: deploy
description: Bridge skill that delegates complex OpenSpec changes to Oh My ClaudeCode (OMC) team/ralph mode. Triggered when user says "run with team", "parallelize", "delegate to ralph", "start big task", "delegate full implementation", or when an OpenSpec change has 4+ tasks, multiple domains, or phased structure. Best for large-scale parallelizable implementations and refactoring requiring iterative verification.
---

# OpenSpec → OMC Deploy Bridge

Bridge skill that hands off complex OpenSpec planning artifacts to OMC team or ralph mode. Automatically determines the optimal execution strategy by analyzing task structure.

## When to use

- tasks.md has 4 or more tasks
- Work is divided into multiple phases
- Changes span multiple domains (frontend + backend, DDL + service + tests, etc.)
- Refactoring requiring iterative verification until completion

## Execution flow

### Step 1: Identify target change

Scan for active changes, same as `quick`.

```bash
ls -d openspec/changes/*/tasks.md 2>/dev/null | grep -v archive
```

If multiple changes exist, ask the user to select one.

### Step 2: Validate all artifacts

Complex work requires the full artifact set:

```bash
CHANGE_DIR="openspec/changes/<change-name>"
MISSING=""
test -f "$CHANGE_DIR/proposal.md" || MISSING="$MISSING proposal.md"
test -f "$CHANGE_DIR/design.md"   || MISSING="$MISSING design.md"
test -f "$CHANGE_DIR/tasks.md"    || MISSING="$MISSING tasks.md"
# Check for delta specs
ls "$CHANGE_DIR/specs/" 2>/dev/null | head -1 || MISSING="$MISSING delta-specs"
```

- If tasks.md or design.md is missing, stop and guide user to run `/opsx:continue` or `/opsx:ff`
- If delta specs are missing, warn but proceed (not all changes require deltas)

### Step 3: Analyze tasks and determine execution strategy

Parse tasks.md to analyze the work structure:

**Analysis criteria:**
- Total task count (pending `- [ ]` items)
- Number of phases (`## Phase`, `## Stage`, or similar headers)
- Inter-task dependencies (whether Phase N tasks depend on Phase N-1 results)
- Domain distribution (DDL, Kotlin services, tests, frontend, etc.)

**Strategy decision matrix:**

| Condition | OMC Mode | Rationale |
|-----------|----------|-----------|
| Sequential phases with strong dependencies | `ralph` | Step-by-step with verification |
| Independent tasks within a phase | `team N:executor` | Parallel processing |
| Single phase, 4+ tasks | `team N:executor` | Simple parallelization |
| Refactoring requiring iterative verification | `ralph` | Repeats until architect approval |
| Sequential phases + parallelizable tasks within | `ralph` + `team` per phase | Hybrid approach |

Agent count calculation:
- Half the number of independent tasks (rounded up), minimum 2, maximum 5
- Example: 6 independent tasks → `team 3:executor`

### Step 4: Assemble rich context

Extract context from all artifacts:

1. **proposal.md** → Motivation, scope, constraints
2. **delta specs** → GIVEN/WHEN/THEN scenarios (used as verification criteria)
3. **design.md** → Architectural decisions, component design, data flow
4. **tasks.md** → Full checklist

Also read `openspec/project.md` to include project-wide context (tech stack, conventions).

### Step 5: Delegate by strategy

#### Strategy A: ralph mode (sequential + iterative verification)

```
ralph Implement the following changes. Do not stop until complete.

## Project Context
[Tech stack and conventions extracted from project.md]

## Change Background
[Summary from proposal.md]

## Architectural Decisions (must follow)
[Key content from design.md]

## Verification Scenarios
[GIVEN/WHEN/THEN from delta specs — criteria for Architect verification]

## Tasks (proceed in order)
[Full text of tasks.md]

## Guidelines
- Verify each phase against its GIVEN/WHEN/THEN scenarios upon completion
- Never deviate from design.md architectural decisions
- Update tasks.md with [x] as each task completes
- Fix immediately if existing tests break
- Run the full test suite after all phases are complete
```

#### Strategy B: team mode (parallel execution)

```
team [N]:executor Implement the following tasks in parallel.

## Project Context
[Extracted from project.md]

## Change Background
[Summary from proposal.md]

## Architectural Decisions (all agents must follow)
[Key content from design.md]

## Task Distribution
### Agent 1: [domain/role]
[Assigned task list]

### Agent 2: [domain/role]
[Assigned task list]

### Agent N: [domain/role]
[Assigned task list]

## Common Guidelines
- Check off completed tasks as [x] in tasks.md
- Strictly follow design.md architectural decisions
- Do not modify files owned by other agents
- Run tests for your assigned area upon completion
```

#### Strategy C: Hybrid (sequential phases, parallel within each)

Proceed through phases in order, using team mode within each phase.
Present the strategy to the user and let them choose between phase-by-phase execution or delegating everything to ralph.

### Step 6: User confirmation before execution

Show the analysis results and get user approval before delegating:

```
OpenSpec → OMC Delegation Analysis

Change: add-jmx-metrics-table
Tasks: 8 (3 phases)
Strategy: ralph (inter-phase dependencies detected)

Phase 1: DDL & Infrastructure (3 tasks)
Phase 2: Service Layer (3 tasks) — depends on Phase 1
Phase 3: Testing (2 tasks) — depends on Phase 2

Verification scenarios: 3 (from delta specs)

Proceed? [Y/n]
To change strategy, say "switch to team" or "use autopilot instead".
```

### Step 7: Post-completion guidance

Once OMC execution completes:

1. Check tasks.md completion status — report any remaining incomplete items
2. If incomplete items remain, inform the user and suggest running this skill again
3. If all tasks are complete, present the user with a choice for the next step:

```
All tasks complete. What would you like to do next?

1. /opsx:verify — Validate implementation against delta spec scenarios
2. /opsx:archive — Archive this change (skip verification)
3. Skip — Do nothing for now

Choose [1/2/3]:
```

Use `AskUserQuestion` to present this choice. Based on the user's selection:
- **1 (verify)**: Invoke the `/opsx:verify` skill
- **2 (archive)**: Invoke the `/opsx:archive` skill
- **3 (skip)**: End with no further action

## Notes

- Team mode requires an active tmux session. On Termux, verify tmux setup first.
- Too many agents may hit rate limits. 5 or fewer recommended.
- For changes without delta specs, use tasks.md items themselves as verification criteria.
- If the user disagrees with the strategy, adjust immediately. This skill's judgment is a suggestion, not a mandate.
