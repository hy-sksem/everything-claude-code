---
description: Autonomous Night Shift mode - Executes tasks, creates PR, and handles AI review feedback (3 cycles default)
---

# Night Shift Command

Autonomous task execution mode that works through `tasks.md` in order, linking to GitHub Issues and following TDD practices.

## Overview

Night Shift operates **completely unsupervised**, automatically:
- Reading tasks from `tasks.md` in order
- Matching tasks to GitHub Issues
- Implementing with TDD cycle (via agents)
- Committing and pushing changes
- Creating PR when complete
- Handling AI review feedback (default: 3 cycles via `/review-pr` logic)
- **Auto-merging PR** when reviews are clean


## CRITICAL AUTONOMOUS EXECUTION RULES

1. **NEVER STOP BETWEEN TASKS** - After completing a task, IMMEDIATELY proceed to the next
2. **NEVER ASK FOR CONFIRMATION** - No user approval, acknowledgment, or input
3. **NEVER ANNOUNCE AND WAIT** - "Proceeding to next task" MUST immediately execute it
4. **CONTINUOUS LOOP** - Complete task → Update tasks.md → Commit → IMMEDIATELY start next
5. **ALWAYS USE AGENTS** (unless `--no-agents`):
   - **MANDATORY**: `tdd-guide` for ALL implementations (manual implementation PROHIBITED)
   - **MANDATORY**: `code-reviewer` BEFORE every commit
   - **RECOMMENDED**: `planner` for complex tasks (>3 steps or architectural changes)
   - **RECOMMENDED**: `security-reviewer` for auth/API/payment/data tasks
6. **ONLY STOP WHEN**: All tasks in tasks.md are `[x]` AND final PR merged (or merge blocked), max-tasks reached, or unrecoverable error

## Prerequisites

1. **tasks.md** exists (search order: `.kiro/specs/[project]/tasks.md` → `.kiro/tasks.md` → `tasks.md`)
2. **spec/** folder with specifications (`.kiro/specs/[project]/` or `spec/`)
3. GitHub CLI (`gh`) authenticated
4. On a feature branch (not main)

## Execution Loop

### 1. IDENTIFY NEXT TASK
- Read `tasks.md` (pull latest from main if just merged a PR)
- Find first unchecked `- [ ]` item
- If none remain → proceed to FINISH (step 8)

### 2. FIND MATCHING ISSUE
```bash
gh issue list --state open --limit 50 --json number,title,body
```
Fuzzy match task name to issues. Proceed without link if no match.

### 3. GATHER CONTEXT
Create `CURRENT_TASK.md` with: task description, GitHub Issue body, relevant spec files, mini implementation plan.

### 4. PLAN (Complex tasks only)
For tasks with >3 steps or architectural changes: MUST delegate to `planner` agent. Save plan to CURRENT_TASK.md.

### 5. EXECUTE (MANDATORY: tdd-guide agent)
Delegate ENTIRE TDD cycle to `tdd-guide` agent: tests first (RED) → implement (GREEN) → refactor → 80%+ coverage.

**Error handling**: If agent fails → `build-error-resolver` agent. Max 3 retries. If still fails: revert, add `<!-- FAILED: [reason] -->` to tasks.md, skip to next.

### 6. REVIEW (MANDATORY before commit)
1. **Code Review (ALL commits)**: `code-reviewer` agent → fix all issues → re-review if significant changes
2. **Security Review (auth/API/payment/data)**: `security-reviewer` agent → fix all vulnerabilities

### 7. COMPLETE & UPDATE
1. Delete `CURRENT_TASK.md`
2. Update tasks.md: `- [ ]` → `- [x]`
3. Commit: `git add . && git commit -m "feat(auto): <Task> (Fixes #<N>)" && git push origin HEAD`
4. Run `/compact` if context growing (every ~5 tasks)
5. **IMMEDIATELY** loop back to step 1 (NO PAUSE)

### 8. FINISH & PR (then check for remaining tasks)

**Step 8.1: Create Pull Request**
```bash
gh pr create --base main --head <branch> \
  --title "Night Shift Report: $(date +%Y-%m-%d)" \
  --body "## Tasks Completed\n[list]\n## Tests Status\nAll passing ✓"
PR_NUMBER=$(gh pr view --json number -q .number)
```

**Step 8.2: Auto-Review Cycles (Default: 3 cycles)**

Runs by default. Skip with `--skip-auto-review` or `--auto-review-cycles 0`.

Uses the same logic as `/review-pr --max-cycles N` (auto mode):

**CRITICAL: You MUST actually wait using Bash `sleep` command. DO NOT skip the wait or just check once and give up.**

```
for cycle in 1..N (default 3):
  1. Trigger /gemini review (cycle 1 only)

  2. **WAIT for reviews (MANDATORY - DO NOT SKIP)**
     Run: sleep 300  (5 minutes)
     This is REAL waiting. You MUST execute the sleep command via Bash tool.
     DO NOT just poll once and exit. DO NOT say "no reviews found" without waiting.

  3. After wait, fetch reviews:
     gh pr view $PR_NUMBER --json reviews,reviewThreads,comments
     Compare with previously seen comments to find NEW ones.

  4. If no new reviews after 5 min wait:
     - Run one more: sleep 300 (another 5 min, 10 min total)
     - Check again. If still none → exit loop.

  5. AI validates each new review (read actual code):
     - VALID → auto-fix with appropriate agent
     - INVALID → skip + document reason
     - UNCLEAR → skip in auto mode
     - OUT_OF_SCOPE → create GitHub issue

  6. Fix VALID CRITICAL/HIGH issues, commit, push
  7. Trigger /gemini review for next cycle
  8. Exit when: max cycles, no new reviews after 10 min, no VALID issues, or build failure
```

**Step 8.3: Auto-Merge PR (default: enabled, unless --no-auto-merge)**

After review cycles complete, check if PR is ready to merge (skip if `--no-auto-merge` flag).

```bash
# Check review decision and CI status
REVIEW_DECISION=$(gh pr view $PR_NUMBER --json reviewDecision -q .reviewDecision)

# Exit early if not approved
if [ "$REVIEW_DECISION" != "APPROVED" ]; then
  echo "⏸️ PR not approved (state: $REVIEW_DECISION). Skipping auto-merge."
  gh pr comment $PR_NUMBER --body "Auto-merge skipped: PR requires approval"
  # Continue to Step 8.4 without merging
  return
fi

# Check if all CI checks pass (non-zero exit if failing)
if ! gh pr checks $PR_NUMBER 2>/dev/null; then
  echo "⏸️ CI checks failing. Skipping auto-merge."
  gh pr comment $PR_NUMBER --body "Auto-merge skipped: CI checks failing"
  # Continue to Step 8.4 without merging
  return
fi
```

**Merge conditions (ALL must be true):**
- `reviewDecision` is APPROVED
- No pending CRITICAL/HIGH VALID reviews left unfixed
- CI checks passing (build + tests green)
- No merge conflicts

**If merge conditions met:**
```bash
gh pr merge $PR_NUMBER --squash --delete-branch
```

**If merge conditions NOT met:**
- Document what's blocking in PR comment
- Continue to Step 8.4 without merging

**Step 8.4: Check for Remaining Tasks (after merge)**

**CRITICAL: After merging, DO NOT stop. Check tasks.md for remaining unchecked tasks.**

```bash
# Switch back to main and pull merged changes
git checkout main && git pull origin main
```

Read `tasks.md` and check for remaining `- [ ]` items.

**If unchecked tasks remain:**
1. Create a new branch: `git checkout -b night-shift-$(date +%Y-%m-%d)-cont`
2. **IMMEDIATELY** loop back to step 1 (Execution Loop)
3. Continue the same autonomous flow: implement → commit → PR → review → merge → check again

**If all tasks are `[x]`:**
- Night Shift is complete. Output final summary and stop.

**This creates a continuous cycle:**
```
Tasks → PR → Review → Merge → Check tasks.md → More tasks? → New branch → Tasks → PR → ...
```

## Arguments

- `--dry-run`: Preview without changes
- `--max-tasks N`: Stop after N tasks (default: all)
- `--skip-pr`: Don't create PR
- `--branch NAME`: Use specific branch
- `--no-agents`: Disable agents (NOT RECOMMENDED)
- `--skip-review`: Skip review agents
- `--quality-mode [fast|standard|thorough]`: fast=tdd only, standard=tdd+review (default), thorough=plan+tdd+review+security
- `--interactive`: Enable confirmation prompts (NOT RECOMMENDED)
- `--auto-review-cycles N`: Review cycles after PR (default: 3)
- `--skip-auto-review`: Disable review cycles
- `--auto-merge`: Auto-merge PR when reviews are clean (default: true)
- `--no-auto-merge`: Skip auto-merge, leave PR open for manual review

## Safety Guards

- **Branch check**: Must NOT be on `main`. Auto-create `night-shift-YYYY-MM-DD` if needed.
- **Test validation**: All tests must pass before committing.
- **Retry limit**: Max 3 attempts per task, auto-skip on persistent failure.
- **Context management**: `/compact` every ~5 tasks.

## File Structure

```
project/
├── .kiro/specs/[project]/    # Kiro structure (preferred)
│   ├── tasks.md              # Task checklist
│   ├── database.md           # Specs...
│   └── api.md
├── spec/                     # Fallback structure
├── tasks.md                  # Fallback location
├── CURRENT_TASK.md           # Temporary (auto-deleted)
└── commands/night-shift.md
```

## tasks.md Format

```markdown
# Project Tasks

## Phase 1: Setup
- [x] Initialize Project Structure
- [ ] Setup Database Schema
- [ ] Configure Authentication

## Phase 2: Features
- [ ] Implement User Registration
<!-- FAILED: Auth service not responding -->
- [ ] Create Dashboard
```

## GitHub Issue Matching

Fuzzy match task names to issue titles by word overlap.
**Task**: `- [ ] Setup Database Schema` → **Issue**: `#15 - Database Schema Setup`
**Commit**: `feat(auto): Setup Database Schema (Fixes #15)`

## Error Recovery

- **Failed test**: Analyze → fix → retry (max 3). On persistent failure → skip + comment in tasks.md.
- **Lost context**: Run `/compact`, resume from current unchecked task.
- **Interruption recovery**: Check `CURRENT_TASK.md` → `tasks.md` → git log → resume with `/night-shift`.

## Examples

```bash
/night-shift                              # Standard: all tasks + 3 review cycles
/night-shift --quality-mode thorough      # All agents + 3 review cycles
/night-shift --max-tasks 5                # Limited tasks + 3 review cycles
/night-shift --auto-review-cycles 5       # Custom review cycles
/night-shift --skip-auto-review           # Tasks only, no auto-review
/night-shift --no-agents                  # Manual mode (not recommended)
/night-shift --dry-run                    # Preview only
```

## Best Practices

**Before**: Order tasks.md, update specs, create GitHub Issues, commit pending work.
**During**: Monitor progress, check test results, review commits, verify AI validation decisions.
**After**: Review PR, verify tests, check implementation quality, validate skipped reviews.
**Auto-review**: Default 3 cycles is thorough. First cycle catches most issues. Check created issues for OUT_OF_SCOPE items. Use `--skip-auto-review` only for manual review preference.

## Integration

- `/review-pr`: Standalone PR review (same logic used in step 8.2)
- `/compact`: Context management during long sessions
- `/verify`: Validation checks
- `/checkpoint`: State saving

## Monitoring

```bash
git log --oneline --grep="feat(auto)"    # Track commits
git diff HEAD~10 tasks.md                # Check progress
npm test || yarn test || pnpm test       # Verify tests
```
