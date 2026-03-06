---
name: quick
description: Bridge skill that delegates simple OpenSpec changes to Oh My ClaudeCode (OMC) autopilot mode. Triggered when user says "implement quickly", "apply now", "delegate to autopilot", "apply opsx", "start spec implementation", or when an OpenSpec change folder has tasks.md ready with only straightforward implementation remaining. Best for 3 or fewer tasks, single domain, no parallelization needed.
---

# OpenSpec → OMC Quick Bridge

Bridge skill that hands off simple OpenSpec planning artifacts to OMC autopilot.
Eliminates the manual copy-paste of tasks.md and context re-explanation.

## When to use

- OpenSpec change folder has tasks.md with 3 or fewer tasks
- Single domain change (narrow file modification scope)
- Linear work that does not require parallel agents

## Execution flow

### Step 1: Identify target change

Scan `openspec/changes/` directory for active changes.

```bash
ls -d openspec/changes/*/tasks.md 2>/dev/null | grep -v archive
```

- If exactly 1 change exists, select it automatically
- If multiple exist, ask the user which change to implement
- If none exist, guide user to run `/opsx:propose` or `/opsx:ff` first and stop

### Step 2: Validate artifacts

Check minimum required files in the selected change folder:

```bash
CHANGE_DIR="openspec/changes/<change-name>"
# Required: tasks.md
test -f "$CHANGE_DIR/tasks.md" || echo "MISSING: tasks.md"
# Recommended: proposal.md (for context)
test -f "$CHANGE_DIR/proposal.md" || echo "WARNING: proposal.md missing - context may be insufficient"
```

If tasks.md is missing, stop. If proposal.md is missing, warn and proceed.

### Step 3: Gather context

Read files in order to assemble implementation context:

1. `proposal.md` — Motivation and scope (Why/What)
2. `design.md` — Technical approach (How) — read if present, skip if absent
3. `tasks.md` — Implementation checklist (read in full)

Extract the following:
- **Goal summary**: 1-2 sentences from the Intent/Why section of proposal.md
- **Technical constraints**: Key decisions from design.md (engine choice, patterns, etc.)
- **Pending tasks**: Filter only `- [ ]` items from tasks.md

### Step 4: Delegate to OMC autopilot

Compose gathered context into a single autopilot instruction. Key principles:

- Reference tasks.md verbatim (do not summarize)
- Explicitly pass design.md constraints
- Maintain checklist format so OMC can track progress

Prompt structure to compose:

```
autopilot Implement the following changes.

## Background
[1-2 sentence summary extracted from proposal.md]

## Technical Constraints
[Key decisions from design.md. Omit this section if design.md is absent]

## Tasks
[Full text of pending items from tasks.md]

## Guidelines
- Check off each task as [x] in tasks.md upon completion
- Strictly follow architectural decisions from design.md
- Verify existing tests are not broken
```

### Step 5: Post-completion guidance

Once OMC autopilot completes, guide the user to next steps:

- "Implementation complete. You can run `/opsx:verify` to validate against the spec, or `/opsx:archive` to archive."

## Notes

- This skill is not a replacement for OpenSpec's `/opsx:apply` — it is an alternative path leveraging OMC's autonomous execution.
- If tasks.md has 4+ tasks or changes span multiple domains, recommend the `deploy` skill instead.
- If OMC is not installed, this skill cannot run. Guide user to standard `/opsx:apply`.
