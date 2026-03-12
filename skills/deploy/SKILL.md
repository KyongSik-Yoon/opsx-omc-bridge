---
name: deploy
description: Bridge skill that delegates complex OpenSpec changes to Oh My ClaudeCode (OMC) team/ralph mode. Triggered when user says "run with team", "parallelize", "delegate to ralph", "start big task", "delegate full implementation", or when an OpenSpec change has 4+ tasks, multiple domains, or phased structure. Best for large-scale parallelizable implementations and refactoring requiring iterative verification.
---

# OpenSpec → OMC Deploy Bridge

Bridge skill that hands off complex OpenSpec planning artifacts to OMC team or ralph mode. Automatically determines the optimal execution strategy by running a preflight pass, resolving the canonical execution context, and analyzing task structure.

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

### Step 2: Preflight and validate artifacts

Complex work requires the full artifact set and a clear execution context:

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
- If a compacted execution brief exists, treat it as the primary implementation handoff and use the full artifacts as supporting evidence

Preferred compacted brief candidates:

```bash
ls "$CHANGE_DIR"/implementation-brief.md \
   "$CHANGE_DIR"/brief.md \
   "$CHANGE_DIR"/context.md \
   "$CHANGE_DIR"/handoff.md 2>/dev/null | head -1
```

Preflight checklist:

- Resolve the canonical artifact set for the change
- Detect whether a compacted brief is available
- Confirm tasks.md is current and still reflects pending work
- Identify phase boundaries and obvious dependency edges
- Mark phases that are safe to parallelize internally
- Surface any ambiguity before delegating to OMC

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
| Sequential phases + parallelizable tasks within | `ralph` + `team` per phase | Hybrid approach with phase gates |

Agent count calculation:
- Half the number of independent tasks (rounded up), minimum 2, maximum 5
- Example: 6 independent tasks → `team 3:executor`

Default recommendation rules:

- Choose `ralph` when the overall flow is sequential and mistakes in earlier phases will invalidate later work
- Choose `hybrid` when the global phase order is sequential, but one or more phases have internally parallelizable tasks
- Choose plain `team` only when most work is independent and can safely proceed without serialized phase gates

### Step 4: Assemble rich context

Extract context from all artifacts with this precedence:

1. **Compacted brief** (if present) → Primary execution handoff, accepted scope, non-goals, phase notes
2. **proposal.md** → Motivation, scope, constraints
3. **delta specs** → GIVEN/WHEN/THEN scenarios (used as verification criteria)
4. **design.md** → Architectural decisions, component design, data flow
5. **tasks.md** → Full checklist

Also read `openspec/project.md` to include project-wide context (tech stack, conventions).

Important context handling rules:

- Treat the compacted brief as the execution-facing summary, not as a replacement for the source artifacts
- Preserve acceptance criteria, edge cases, non-goals, and explicit constraints from the full spec set
- If the compacted brief conflicts with source artifacts, stop and ask the user which one is canonical
- When no compacted brief exists, build a concise implementation handoff from the canonical artifacts before delegating

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

#### Strategy B: hybrid mode (ralph orchestration with team inside selected phases)

Use `ralph` as the outer orchestrator. For phases that are parallelizable internally, instruct `ralph` to delegate only that phase's independent work to `team`, wait for verification, then continue to the next phase.

Prompt structure to compose:

```text
ralph Implement the following phased change. Keep global phase order intact.

## Project Context
[Extracted from project.md]

## Change Background
[Summary from compacted brief or proposal.md]

## Architectural Decisions (must follow)
[Key content from design.md]

## Verification Scenarios
[GIVEN/WHEN/THEN from delta specs]

## Execution Plan
- Follow phases in order
- Use team mode only for the phases explicitly marked parallelizable
- Verify each phase before continuing
- If a phase fails verification, repair it before moving on
- If interrupted, resume from the last incomplete phase rather than restarting everything

## Parallelizable Phases
[Example: Phase 4 storage adapters, Phase 9 notification fan-out]

## Tasks
[Full text of tasks.md]
```

#### Strategy C: team mode (parallel execution)

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

### Step 6: User confirmation before execution

Show the analysis results and get user approval before delegating:

```
OpenSpec → OMC Delegation Analysis

Change: add-jmx-metrics-table
Tasks: 8 (3 phases)
Strategy: hybrid (ralph orchestrator with team inside Phase 4-5 and Phase 9-10)

Phase 1: DDL & Infrastructure (3 tasks)
Phase 2: Service Layer (3 tasks) — depends on Phase 1
Phase 3: Testing (2 tasks) — depends on Phase 2

Verification scenarios: 3 (from delta specs)
Resume support: start from last incomplete phase if interrupted

Proceed? [Y/n]
To change strategy, say "switch to team" or "use autopilot instead".
```

When the analysis shows a sequential global flow with only a few parallelizable phases, recommend `hybrid` instead of plain `team`.

### Step 7: Post-completion guidance

Once OMC execution completes:

1. Check tasks.md completion status — report any remaining incomplete items
2. If incomplete items remain, inform the user and suggest running this skill again
3. If execution was interrupted mid-plan, identify the last completed phase and offer to resume from the next incomplete phase
4. If all tasks are complete, present the user with a choice for the next step:

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
- For large multi-phase changes, `ralph` or `hybrid` should be the default. Plain `team` is for work that is genuinely independent.
