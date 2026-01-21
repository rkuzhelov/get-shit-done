# Feature Plan: `gsd:file-bug`

## Overview

Add a new command `/gsd:file-bug <phase> <description>` that files a bug against a completed phase, automatically diagnoses the root cause, and creates a verified fix plan ready for execution.

**Motivation:** The current `/gsd:verify-work` command couples two processes: manual UAT testing and gap filing. When a user already knows a bug exists (discovered outside of formal testing), they must either:
1. Run through the full UAT flow just to log one known issue
2. Manually create gap entries and trigger diagnosis separately

This command provides a direct path: describe the bug → get diagnosis → get fix plan. It reuses the existing diagnosis and planning infrastructure from `verify-work` while skipping the interactive testing ceremony.

---

## Requirements Summary

| Requirement | Details |
|-------------|---------|
| **Command name** | `gsd:file-bug` |
| **Arguments** | `<phase> <description>` (phase number + bug description) |
| **Phase validation** | Must be completed phase (SUMMARY.md exists) |
| **UAT file handling** | Create if missing, append if exists (regardless of status) |
| **Test case generation** | Auto-generate test entry, propose to user for review before proceeding |
| **Test numbering** | Continue from last test number in existing UAT file |
| **Severity inference** | Inferred from description (same logic as verify-work) |
| **Bug count** | One bug at a time |
| **Diagnosis** | Automatic - spawn debug agent (same as verify-work) |
| **Planning** | Automatic - spawn gsd-planner in --gaps mode |
| **Plan verification** | Automatic - spawn gsd-plan-checker with iteration loop (max 3) |
| **Git behavior** | Commit UAT file after gap entry, commit again after diagnosis |

---

## Core Flow

```
┌─────────────────────────────────────────────────────────────────┐
│  /gsd:file-bug 3 "Button doesn't respond when clicked"         │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  1. VALIDATE PHASE                                              │
│     • Phase directory exists?                                   │
│     • SUMMARY.md exists? (phase completed)                      │
│     • If not → error and exit                                   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  2. GENERATE & PROPOSE TEST CASE                                │
│     • Auto-generate test from description                       │
│     • Present to user for review/editing                        │
│     • Wait for user confirmation                                │
│     • User can modify test name and expected behavior           │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  3. CREATE/UPDATE UAT FILE                                      │
│     • If no UAT file → create from template                     │
│     • If exists → append (regardless of current status)         │
│     • Add test entry (continue numbering from last)             │
│     • Add gap entry (YAML format, severity inferred)            │
│     • Commit UAT file                                           │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  4. DIAGNOSE (automatic)                                        │
│     • Spawn debug agent for the single gap                      │
│     • Agent investigates root cause autonomously                │
│     • Update UAT.md gaps with diagnosis results                 │
│     • Commit updated UAT file                                   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  5. PLAN FIX (automatic)                                        │
│     • Spawn gsd-planner in --gaps mode                          │
│     • Create fix plan based on diagnosed root cause             │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  6. VERIFY PLAN (automatic)                                     │
│     • Spawn gsd-plan-checker                                    │
│     • If issues found → revision loop (max 3 iterations)        │
│     • Iterate planner ↔ checker until plans pass                │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  7. PRESENT RESULT                                              │
│     • Route C: Fix plans ready → suggest execute-phase --gaps   │
│     • Route D: Planning blocked → manual intervention needed    │
└─────────────────────────────────────────────────────────────────┘
```

---

## Validation Rules

### Phase Existence Check

```bash
# Find phase directory (match both zero-padded and unpadded)
PADDED_PHASE=$(printf "%02d" ${PHASE_ARG} 2>/dev/null || echo "${PHASE_ARG}")
PHASE_DIR=$(ls -d .planning/phases/${PADDED_PHASE}-* .planning/phases/${PHASE_ARG}-* 2>/dev/null | head -1)

if [ -z "$PHASE_DIR" ]; then
  echo "ERROR: Phase ${PHASE_ARG} not found"
  exit 1
fi
```

### Phase Completion Check

A phase is "completed" if any `*-SUMMARY.md` files exist in its directory:

```bash
if ! ls "$PHASE_DIR"/*-SUMMARY.md 2>/dev/null | head -1 > /dev/null; then
  echo "ERROR: Phase ${PHASE_ARG} is not completed yet"
  echo ""
  echo "Bugs can only be filed against completed phases."
  echo "A phase is completed when it has at least one SUMMARY.md file."
  exit 1
fi
```

**Rationale:** You can't have bugs in phases that haven't been executed yet. The phase must have produced work that can contain bugs.

### Error Messages

**Phase doesn't exist:**
```
ERROR: Phase 7 not found

Available phases: 1, 2, 3, 4, 5
```

**Phase not completed:**
```
ERROR: Phase 4 is not completed yet

Bugs can only be filed against completed phases.
A phase is completed when it has at least one SUMMARY.md file.

Current phase 4 status: Planning in progress
```

---

## Test Case Generation

### Auto-Generation Logic

From the user's description, generate:

1. **Test name**: First ~50 characters of description (or full if shorter), cleaned up
2. **Expected behavior**: "Should work correctly: {full description}"

### Proposal Format

Present to user in editable format:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD ► FILING BUG FOR PHASE {N}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Proposed test case:**

Test name: {generated_name}
Expected: {generated_expected}
Severity: {inferred_severity}

───────────────────────────────────────────────────────────────

Does this look correct?

- Reply "yes" or "y" to proceed
- Or provide corrections (e.g., "change expected to: should show error message")

───────────────────────────────────────────────────────────────
```

### User Corrections

If user provides corrections, parse and apply:
- "change name to: ..." → update test name
- "change expected to: ..." → update expected behavior
- "severity should be blocker" → update severity
- Full replacement text → replace entirely

After corrections applied, proceed automatically (don't re-confirm unless user explicitly asks).

---

## UAT File Handling

### File Location

```
.planning/phases/{NN}-{slug}/{phase}-UAT.md
```

Example: `.planning/phases/03-authentication/03-UAT.md`

### Creating New UAT File

If no UAT file exists, create using the template from `~/.claude/get-shit-done/templates/UAT.md`:

```markdown
---
status: diagnosed
phase: {NN}-{slug}
source: [manual bug report]
started: {ISO timestamp}
updated: {ISO timestamp}
---

## Current Test

[testing complete]

## Tests

### 1. {Test Name}
expected: {Expected behavior}
result: issue
reported: "{User's original description}"
severity: {inferred}

## Summary

total: 1
passed: 0
issues: 1
pending: 0
skipped: 0

## Gaps

- truth: "{Expected behavior}"
  status: failed
  reason: "User reported: {original description}"
  severity: {inferred}
  test: 1
  root_cause: ""
  artifacts: []
  missing: []
  debug_session: ""
```

### Appending to Existing UAT File

If UAT file exists (regardless of status):

1. Parse existing tests to find last test number
2. Add new test with number = last + 1
3. Update Summary counts (increment total and issues)
4. Append to Gaps section
5. Update frontmatter.updated timestamp

**Do NOT change existing status** - the diagnosis step will update it to "diagnosed" after completion.

---

## Severity Inference

Same logic as verify-work - infer from user's natural language:

| User describes | Infer |
|----------------|-------|
| crash, error, exception, fails completely, unusable, broken | **blocker** |
| doesn't work, nothing happens, wrong behavior, missing, can't | **major** |
| works but..., slow, weird, minor, small issue | **minor** |
| color, font, spacing, alignment, visual, looks off | **cosmetic** |

**Default:** major (safe default if unclear)

**Never ask the user for severity** - just infer and move on.

---

## Diagnosis Step

### Single-Agent Variant

Unlike verify-work which may diagnose multiple gaps in parallel, file-bug always has exactly one gap. Spawn a single debug agent:

```
Task(
  prompt="""
<debug_context>

**Symptom:** {expected_behavior}
**Actual:** {user_description}
**Severity:** {severity}
**Phase:** {phase_number} - {phase_name}

**Project State:**
@.planning/STATE.md

**Phase Artifacts:**
@.planning/phases/{phase_dir}/

</debug_context>

<goal>
find_root_cause_only

Investigate this single issue. Find the root cause with evidence.
Do NOT propose fixes - just diagnose.
</goal>

<output_format>
Return:
## ROOT CAUSE FOUND

**Root Cause:** {specific cause with evidence}

**Evidence Summary:**
- {key finding 1}
- {key finding 2}

**Files Involved:**
- {file1}: {what's wrong}

**Suggested Fix Direction:** {brief hint for planner}
</output_format>
""",
  subagent_type="general-purpose",
  description="Debug: {short_description}"
)
```

### Update UAT After Diagnosis

After agent returns, update the gap entry in UAT.md:

```yaml
- truth: "{expected behavior}"
  status: failed
  reason: "User reported: {description}"
  severity: {severity}
  test: {N}
  root_cause: "{from debug agent}"
  artifacts:
    - path: "{file}"
      issue: "{what's wrong}"
  missing:
    - "{what needs to be done}"
  debug_session: ".planning/debug/{slug}.md"
```

Update frontmatter status to "diagnosed".

Commit:
```bash
git add ".planning/phases/{phase_dir}/{phase}-UAT.md"
git commit -m "docs({phase}): diagnose bug - {short_description}"
```

---

## Planning Step

### Spawn Planner

Same as verify-work - spawn gsd-planner in --gaps mode:

```
Task(
  prompt="""
<planning_context>

**Phase:** {phase_number}
**Mode:** gap_closure

**UAT with diagnosis:**
@.planning/phases/{phase_dir}/{phase}-UAT.md

**Project State:**
@.planning/STATE.md

**Roadmap:**
@.planning/ROADMAP.md

</planning_context>

<downstream_consumer>
Output consumed by /gsd:execute-phase --gaps-only
Plans must be executable prompts.
</downstream_consumer>
""",
  subagent_type="gsd-planner",
  description="Plan fix for Phase {phase}"
)
```

---

## Plan Verification Step

### Spawn Checker

```
Task(
  prompt="""
<verification_context>

**Phase:** {phase_number}
**Phase Goal:** Close diagnosed gap from manual bug report

**Plans to verify:**
@.planning/phases/{phase_dir}/*-PLAN.md

</verification_context>

<expected_output>
Return one of:
- ## VERIFICATION PASSED — all checks pass
- ## ISSUES FOUND — structured issue list
</expected_output>
""",
  subagent_type="gsd-plan-checker",
  description="Verify Phase {phase} fix plan"
)
```

### Revision Loop

If checker returns issues, iterate (max 3 times):

1. Send issues back to planner with revision context
2. Planner updates plan
3. Re-run checker
4. Repeat until pass or max iterations

If max iterations reached without passing:
- Report blocking issues
- Offer manual intervention options

---

## Output Routing

### Route C: Fix Plans Ready

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD ► BUG FILED & FIX READY ✓
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Phase {N}: {Name}**

Bug: {short_description}
Root cause: {root_cause}
Fix plan: {plan_file}

───────────────────────────────────────────────────────────────

## ▶ Next Up

**Execute fix** — run the fix plan

`/clear` then `/gsd:execute-phase {N} --gaps-only`

───────────────────────────────────────────────────────────────

**Also available:**
- `cat {plan_file}` — review fix plan
- `/gsd:plan-phase {N} --gaps` — regenerate fix plan

───────────────────────────────────────────────────────────────
```

### Route D: Planning Blocked

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD ► BUG FILED BUT FIX BLOCKED ✗
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Phase {N}: {Name}**

Bug: {short_description}
Root cause: {root_cause}
Fix planning blocked after {X} iterations

### Unresolved Issues

{issues from planner/checker}

───────────────────────────────────────────────────────────────

## ▶ Next Up

**Manual intervention required**

Review the issues above and either:
1. Provide guidance for fix planning
2. Manually address blockers
3. Accept current state and continue

───────────────────────────────────────────────────────────────

**Options:**
- `/gsd:plan-phase {N} --gaps` — retry fix planning with guidance
- `/gsd:discuss-phase {N}` — gather more context

───────────────────────────────────────────────────────────────
```

---

## Edge Cases

| Case | Handling |
|------|----------|
| **Phase doesn't exist** | ERROR with list of available phases |
| **Phase not completed** | ERROR explaining completion requirement |
| **No description provided** | ERROR: Description required |
| **Description is empty/whitespace** | ERROR: Description cannot be empty |
| **Very long description** | Accept (test name truncated to ~50 chars) |
| **Multi-line description** | Accept (use full text, generate short name) |
| **UAT file exists with status: testing** | Append anyway (don't interrupt active session) |
| **UAT file exists with status: complete** | Append and proceed |
| **UAT file exists with status: diagnosed** | Append and proceed |
| **Debug agent fails** | Mark as "needs manual review", offer /gsd:debug |
| **Planner fails** | Report and offer manual planning |
| **No .planning directory** | ERROR: No project initialized |

---

## Anti-Patterns

- Don't skip the test case confirmation step
- Don't ask for severity - always infer
- Don't allow filing bugs against incomplete phases
- Don't interrupt active UAT sessions (append is safe)
- Don't create duplicate gaps for same description
- Don't proceed to diagnosis without user confirming test case
- Don't auto-commit without completing the logical step
- Don't use AskUserQuestion for test case confirmation - plain text conversation

---

## Success Criteria

Bug filing is complete when:

- [ ] Arguments parsed (phase number + description)
- [ ] Phase validated (exists and completed)
- [ ] Test case generated and proposed to user
- [ ] User confirmed or edited test case
- [ ] UAT file created or updated with new test and gap
- [ ] UAT file committed
- [ ] Debug agent spawned and returned root cause
- [ ] UAT file updated with diagnosis
- [ ] Diagnosis committed
- [ ] gsd-planner spawned in --gaps mode
- [ ] gsd-plan-checker verified plan (with iteration if needed)
- [ ] User presented with next steps (Route C or D)
- [ ] Ready for `/gsd:execute-phase --gaps-only`

---

## Comparison with Related Commands

| Command | Purpose | Tests User? | Diagnoses? | Plans Fix? | Commits? |
|---------|---------|-------------|------------|------------|----------|
| `verify-work` | Full UAT flow | Yes (interactive) | Yes | Yes | Yes |
| `debug` | Investigate specific issue | No | Yes | No | No |
| `audit-milestone` | Find gaps at milestone end | No | No | No | Yes |
| `plan-milestone-gaps` | Plan fixes from audit | No | No | Yes | No |
| **`file-bug`** | Direct bug → fix plan | No (proposes) | Yes | Yes | Yes |

---

## Implementation Notes

### Command File Structure

Create: `commands/gsd/file-bug.md`

```yaml
---
name: gsd:file-bug
description: File a bug against a completed phase with automatic diagnosis and fix planning
argument-hint: "<phase> <description>"
allowed-tools:
  - Read
  - Bash
  - Glob
  - Grep
  - Edit
  - Write
  - Task
---
```

### Workflow File

Create: `get-shit-done/workflows/file-bug.md`

Detailed workflow steps following the same structure as `verify-work.md`.

### Execution Context

```yaml
execution_context:
  - ~/.claude/get-shit-done/templates/UAT.md
  - ~/.claude/get-shit-done/workflows/diagnose-issues.md
  - .planning/STATE.md
  - .planning/ROADMAP.md
  - .planning/phases/
```

### Reused Components

| Component | Source | Usage |
|-----------|--------|-------|
| UAT template | `~/.claude/get-shit-done/templates/UAT.md` | File structure |
| Severity inference | `verify-work.md` | Same logic |
| Debug agent spawn | `diagnose-issues.md` | Single-agent variant |
| gsd-planner | `verify-work.md` | --gaps mode |
| gsd-plan-checker | `verify-work.md` | With iteration loop |
| Route C/D output | `verify-work.md` | Same format |

---

## Files to Create

| File | Purpose |
|------|---------|
| `commands/gsd/file-bug.md` | Command definition with frontmatter |
| `get-shit-done/workflows/file-bug.md` | Detailed workflow steps |

---

## Example Usage

### Basic Usage

```
/gsd:file-bug 3 "Login button doesn't respond when clicked"
```

### Multi-line Description

```
/gsd:file-bug 3 "When user clicks the submit button on the registration form,
nothing happens. No error message, no loading indicator, just silent failure.
Console shows no errors."
```

### Expected Interaction

```
User: /gsd:file-bug 3 "Login button doesn't respond when clicked"

Claude:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD ► FILING BUG FOR PHASE 3
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Proposed test case:**

Test name: Login button responds when clicked
Expected: Login button should respond when clicked and initiate authentication flow
Severity: major

───────────────────────────────────────────────────────────────

Does this look correct? Reply "yes" to proceed or provide corrections.

───────────────────────────────────────────────────────────────

User: yes

Claude:
Bug logged. Spawning debug agent to diagnose root cause...

[Debug agent runs]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD ► DIAGNOSIS COMPLETE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Root cause: onClick handler missing from LoginButton component

Planning fix...

[Planner runs]
[Checker runs]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD ► BUG FILED & FIX READY ✓
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Phase 3: Authentication**

Bug: Login button doesn't respond when clicked
Root cause: onClick handler missing from LoginButton component
Fix plan: .planning/phases/03-authentication/03-04-PLAN.md

───────────────────────────────────────────────────────────────

## ▶ Next Up

**Execute fix** — run the fix plan

`/clear` then `/gsd:execute-phase 3 --gaps-only`

───────────────────────────────────────────────────────────────
```
