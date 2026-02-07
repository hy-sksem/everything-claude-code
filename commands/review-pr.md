---
description: Analyze PR reviews, implement fixes with agent collaboration, and trigger Gemini review
argument-hint: [pr-number] [--interactive] [--max-cycles N]
allowed-tools: [Read, Grep, Glob, Bash, Edit, Write, Task, AskUserQuestion]
model: sonnet
---

# PR Review Response

Automatically handle code review feedback on pull requests.

## Modes

- **Auto (Default)**: Fully autonomous. AI validates correctness, auto-fixes valid CRITICAL/HIGH, skips invalid reviews.
- **Interactive (`--interactive`)**: User confirms all comments. CRITICAL/HIGH pre-selected, MEDIUM/LOW optional.
- **Mixed (`--interactive-medium`)**: Auto-fix CRITICAL/HIGH, ask user about MEDIUM/LOW.

## Arguments

Parse from $ARGUMENTS:
- `[pr-number]`: PR number (default: current branch's PR)
- `--interactive`: Enable user confirmation mode
- `--max-cycles N`: Repeat review-fix cycle up to N times (default: 1)
- `--interactive-medium`: Auto-fix CRITICAL/HIGH, ask about MEDIUM/LOW
- `--auto-merge`: Auto-merge PR when reviews are clean (off by default in standalone mode)

```bash
/review-pr                          # Auto mode, current PR, 1 cycle
/review-pr 123                      # Auto mode, PR #123
/review-pr --max-cycles 3           # Auto mode, up to 3 cycles
/review-pr --interactive            # Interactive mode
/review-pr --auto-merge             # Auto-merge if reviews clean
/review-pr 123 --interactive --max-cycles 2
```

## Workflow

### 1. Identify PR and Fetch Reviews

```bash
BRANCH=$(git rev-parse --abbrev-ref HEAD)
PR_NUMBER=${1:-$(gh pr view --json number -q .number 2>/dev/null)}
```

If no PR found, inform user and exit.

```bash
gh pr view $PR_NUMBER --json reviews,reviewThreads,comments,title,url
```

### 2. Categorize Review Comments

Parse each comment by priority:

| Priority | Criteria | Action |
|----------|----------|--------|
| **CRITICAL** | Security vulns, breaking bugs, data corruption, auth issues | Must fix |
| **HIGH** | Code quality, performance, missing error handling, test gaps | Important |
| **MEDIUM** | Style, minor refactoring, docs, optimization | Should address |
| **LOW** | Nitpicks, alternatives, nice-to-haves | Optional |

For each actionable comment: extract file/line, read code, understand concern, determine if code change needed (skip discussions).

### 2a. Validate Review Correctness (Auto Mode - Default)

**Skipped when `--interactive` is present.**

For each review, AI reads actual code and validates:

1. **Technical Accuracy**: Does the issue actually exist? (e.g., "missing null check" → verify)
2. **Code Context Match**: Does review apply to current code? (not outdated references)
3. **Scope Appropriateness**: Within PR scope? (OUT_OF_SCOPE → create issue)
4. **Priority Validation**: Is severity justified? (re-categorize if not)
5. **Duplicate/Contradiction Check**: Conflicts with other reviews?

**Validation outcomes**: VALID (will address), INVALID (skip + reason), UNCLEAR (skip in auto), OUT_OF_SCOPE (create issue + skip)

**Decision Matrix:**

| Priority | Validation | Action |
|----------|-----------|---------|
| CRITICAL/HIGH | VALID | Auto-fix |
| CRITICAL/HIGH | INVALID | Skip + document why |
| CRITICAL/HIGH | UNCLEAR | Skip + note for manual review |
| CRITICAL/HIGH | OUT_OF_SCOPE | Skip + create issue |
| MEDIUM/LOW | VALID | Skip (lower priority) |
| MEDIUM/LOW | INVALID | Skip |

### 3. User Confirmation (Interactive Mode Only)

**Skipped by default. Only runs with `--interactive`.**

Present ALL comments for user selection via AskUserQuestion (multiSelect: true):
- CRITICAL/HIGH → Pre-selected (user can deselect)
- MEDIUM/LOW → Not selected (user can select)
- Show: priority, reviewer, file:line, concern
- If user deselects CRITICAL/HIGH, ask for confirmation

### 4. Plan Implementation

Group related fixes and select agents:

| Issue Type | Agent |
|-----------|-------|
| Security | `security-reviewer` |
| Missing tests | `tdd-guide` |
| Code quality | `code-reviewer` |
| Refactoring | `refactor-cleaner` |
| Build issues | `build-error-resolver` |

**Parallel**: Independent fixes in different files → launch agents concurrently in single message.
**Sequential**: Dependent changes (e.g., refactor → test) → launch in order.

### 5. Implement Changes

For each fix: launch appropriate agent via Task tool with review context (comment, file:line, concern). Run independent agents in PARALLEL (single message with multiple Task calls).

Document deselected CRITICAL/HIGH items with reason (incorrect, out of scope, separate PR).

### 6. Verify All Changes

```bash
npm run build || yarn build || pnpm build
npm test || yarn test || pnpm test
```

If failures: launch `build-error-resolver` / `tdd-guide`, fix, retry.

### 7. Commit and Push

```bash
git add <modified-files>
git commit -m "$(cat <<'EOF'
fix: address PR review feedback (#$PR_NUMBER)

Addressed:
- [@reviewer] Category: Description of fix

Skipped/Deferred:
- [@reviewer] Reason (INVALID/OUT_OF_SCOPE/deferred)

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
EOF
)"
git push origin HEAD
```

### 8. Trigger Gemini Review

```bash
gh pr comment $PR_NUMBER --body "/gemini review"
```

Note: Copilot cannot be triggered via CLI. Use "Review new pushes" ruleset or manual UI re-request.

### 8a. Review Cycle Repetition (with --max-cycles)

**Only runs when `--max-cycles N` where N > 1.**

**CRITICAL: You MUST actually wait using Bash `sleep` command. DO NOT skip the wait or poll once and give up.**

```
current_cycle = 1
while current_cycle <= max_cycles:
  1. WAIT for reviews (MANDATORY):
     Run: sleep 300  (5 minutes via Bash tool)
     This is REAL waiting. You MUST execute the sleep command.

  2. After wait, fetch new reviews:
     gh pr view $PR_NUMBER --json reviews,reviewThreads,comments
     Compare with previously seen comments.

  3. If no new reviews:
     Run: sleep 300 (another 5 min, 10 min total)
     Check again. If still none → exit loop.

  4. Validate new reviews (same as step 2a)
  5. If no VALID reviews → exit loop
  6. Fix VALID reviews (steps 4-7)
  7. Trigger /gemini review
  8. current_cycle++
```

**Exit conditions**: Max cycles reached, no new reviews after 10-min wait, no VALID reviews, build/test failure.

### 8b. Auto-Merge PR (with --auto-merge)

**Only runs when `--auto-merge` flag is present.**

```bash
# Check PR review status and CI
PR_STATE=$(gh pr view $PR_NUMBER --json reviewDecision -q .reviewDecision)
gh pr checks $PR_NUMBER
```

**Merge if ALL conditions met:**
- No unresolved review comments
- No pending CRITICAL/HIGH VALID reviews left unfixed
- CI checks passing
- No merge conflicts

```bash
gh pr merge $PR_NUMBER --squash --delete-branch
```

If conditions not met: document blockers in PR comment, leave open.

### 9. Summary Report

```
✅ PR Review Response Complete ({Mode} - {cycles} cycle(s))

📋 PR #N: [Title]
🔗 URL

📊 Reviews: X CRITICAL, Y HIGH, Z MEDIUM, W LOW
🤖 Validation: N VALID → fixed, M INVALID → skipped, K OUT_OF_SCOPE → issues created
🔧 Changes: [list of fixes with agent used]
⏭️ Skipped: [list with reasons]

✅ Build: Passing | Tests: Passing (N% coverage)
✅ Committed, pushed, Gemini review triggered

🔄 Cycle summary (if multi-cycle):
  Cycle 1: N fixed, M skipped
  Cycle 2: N fixed, M skipped
  ...
```

## Error Handling

- **No PR found**: Exit with message, suggest `/review-pr 123`
- **Build failures**: Launch `build-error-resolver`, fix incrementally
- **Test failures**: Launch `tdd-guide`, fix without regressions
- **Unclear reviews**: Skip, list for manual review
- **Can't auto-fix**: Create TODO list with guidance

## Best Practices

**Auto Mode**: Validate rigorously (read actual code), trust validation, create issues for out-of-scope, document reasons, use `--max-cycles 3` for iteration.

**Interactive Mode**: Summarize before implementing, present ALL for selection, pre-select but don't force, respect deselections, document clearly.

**Both**: Batch related fixes, always verify build/tests, use specialized agents, run independent agents in parallel.
