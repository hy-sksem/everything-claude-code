---
description: Analyze PR reviews, implement fixes with agent collaboration, and trigger Gemini review
argument-hint: [pr-number] [--interactive] [--max-cycles N]
allowed-tools: [Read, Grep, Glob, Bash, Edit, Write, Task, AskUserQuestion]
model: sonnet
---

# PR Review Response

Automatically handle code review feedback on pull requests.

## Modes

### Auto Mode (Default)
- **FULLY AUTONOMOUS** - No user confirmation required
- AI validates each review comment's correctness
- Automatically addresses valid CRITICAL/HIGH issues
- Automatically skips invalid/incorrect reviews
- Can run multiple cycles with `--max-cycles N`
- **This is the default behavior**

### Interactive Mode (`--interactive`)
- User confirms which review comments to address
- CRITICAL/HIGH pre-selected, MEDIUM/LOW optional
- User can deselect any comment (including incorrect reviews)
- Manual control over what gets fixed

## Arguments

Parse from $ARGUMENTS:
- `[pr-number]`: Optional PR number (default: current branch's PR)
- `--interactive`: Enable interactive mode with user confirmation (default: off)
- `--max-cycles N`: Repeat review-fix cycle up to N times (default: 1)
- `--interactive-medium`: Auto-fix CRITICAL/HIGH, ask about MEDIUM/LOW

Examples:
```bash
/review-pr                          # Auto mode, current PR, 1 cycle
/review-pr 123                      # Auto mode, PR #123, 1 cycle
/review-pr --max-cycles 3           # Auto mode, up to 3 cycles
/review-pr --interactive            # Interactive mode with user confirmation
/review-pr 123 --interactive --max-cycles 2
```

## Workflow

### 1. Identify PR and Fetch Reviews

Get current branch and PR number:
```bash
BRANCH=$(git rev-parse --abbrev-ref HEAD)
PR_NUMBER=${1:-$(gh pr view --json number -q .number 2>/dev/null)}
```

If user provided PR number as argument ($ARGUMENTS), use that. Otherwise detect from current branch.

If no PR found, inform user and exit.

Fetch all review data:
```bash
gh pr view $PR_NUMBER --json reviews,reviewThreads,comments,title,url
```

### 2. Analyze and Categorize Review Comments

Parse each comment and categorize by priority:

**CRITICAL** - Must fix before merge:
- Security vulnerabilities
- Breaking bugs
- Data corruption risks
- Authentication/authorization issues

**HIGH** - Important to fix:
- Significant code quality issues
- Performance problems
- Missing error handling
- Inadequate test coverage

**MEDIUM** - Should address:
- Code style issues
- Minor refactoring suggestions
- Documentation improvements
- Optimization opportunities

**LOW** - Optional:
- Nitpicks
- Alternative approaches
- Nice-to-have enhancements

For each actionable comment:
1. Extract file path and line number
2. Read the relevant code section
3. Understand the reviewer's concern
4. Determine if code change is needed (skip questions/discussions)

### 2a. Validate Review Correctness (Auto Mode - Default)

**IMPORTANT: This step runs by DEFAULT. Only skipped when `--interactive` flag is present.**

For each review comment, AI validates its correctness by analyzing the actual code:

**Validation Criteria:**

1. **Technical Accuracy**
   - Read the file at the specified line
   - Verify if the issue actually exists
   - Example: Review says "missing null check" → Check if null check exists
   - Result: VALID if issue exists, INVALID if code is correct

2. **Code Context Match**
   - Ensure review applies to current code (not outdated)
   - Check if mentioned code/function names exist
   - Verify line numbers are accurate
   - Result: INVALID if reviewer is looking at old code

3. **Scope Appropriateness**
   - Is this within the scope of the current PR changes?
   - Should it be a separate PR or issue?
   - Example: "Refactor entire module" on bug fix PR → OUT_OF_SCOPE
   - Result: OUT_OF_SCOPE if not related to PR's purpose

4. **Priority Validation**
   - Is CRITICAL/HIGH severity justified?
   - Example: Style preference marked CRITICAL → Downgrade to LOW
   - Re-categorize if priority is incorrect

5. **Duplicate/Contradiction Check**
   - Does it conflict with other reviews?
   - Is it already addressed by another comment?
   - Result: UNCLEAR if contradictory

**Validation Outcomes:**

- ✅ **VALID**: Technically correct, in scope, priority accurate → Will address
- ❌ **INVALID**: Incorrect, code doesn't have the issue → Skip with reason
- ⚠️ **UNCLEAR**: Ambiguous, conflicting, needs context → Skip in auto mode
- 📋 **OUT_OF_SCOPE**: Valid but separate concern → Create GitHub issue, skip in PR

**Auto Mode Decision Matrix:**

| Priority | Validation | Action |
|----------|-----------|---------|
| CRITICAL/HIGH | VALID | Auto-fix |
| CRITICAL/HIGH | INVALID | Skip + document why |
| CRITICAL/HIGH | UNCLEAR | Skip + note for manual review |
| CRITICAL/HIGH | OUT_OF_SCOPE | Skip + create issue |
| MEDIUM/LOW | VALID | Skip (optional: add flag for auto-fix) |
| MEDIUM/LOW | INVALID | Skip |

**Output validation summary:**
```
📊 Review Validation (Auto Mode):
- 2 CRITICAL: 2 VALID → will fix
- 3 HIGH: 2 VALID → will fix, 1 INVALID (incorrect) → skip
- 2 MEDIUM: 1 VALID → skip, 1 OUT_OF_SCOPE → issue created
- 1 LOW: 1 INVALID → skip

✓ Will address: 4 reviews
⏭ Will skip: 3 reviews (with reasons)
```

### 3. User Confirmation (Interactive Mode Only)

**IMPORTANT: This step is SKIPPED by default. Only runs when `--interactive` flag is present.**

**In Interactive Mode, ALL review comments require user confirmation**, as even CRITICAL/HIGH priority reviews may be incorrect or not applicable.

Present review comments in priority groups with:
- CRITICAL/HIGH → Pre-selected by default (user can deselect)
- MEDIUM/LOW → Not selected by default (user can select)

Use AskUserQuestion to present ALL comments for selection:
- Show each comment with: priority, reviewer name, file path, line number, and concern
- Use multiSelect: true to allow selecting multiple comments
- Group by priority for clarity
- Pre-select CRITICAL and HIGH items (marked with "Recommended")

Example question format:
```
Found 7 review comments. Which should be addressed in this PR?

CRITICAL (Recommended - pre-selected):
- [CRITICAL] [@reviewer] XSS vulnerability in src/auth/login.ts:42 - Add input sanitization

HIGH (Recommended - pre-selected):
- [HIGH] [@reviewer] Missing tests for src/utils/parser.ts - Add unit tests
- [HIGH] [@reviewer] N+1 query in src/api/users.ts:89 - Optimize database query

MEDIUM:
- [MEDIUM] [@reviewer] Code style in src/hooks/useAuth.ts:45 - Apply immutability patterns
- [MEDIUM] [@reviewer] Documentation in src/api/client.ts:12 - Add JSDoc comments

LOW:
- [LOW] [@reviewer] Alternative approach in src/utils/helper.ts - Consider using lodash
```

**Important**: If user deselects a CRITICAL/HIGH item, ask for confirmation that they want to skip it, as this may indicate the review is incorrect or not applicable to the current PR scope.

### 4. Plan Implementation Strategy

**Auto Mode (Default)**: After AI validates which reviews are VALID and worth fixing
**Interactive Mode**: After user confirms which items to address

Group related fixes and identify required agents:

**Agent Selection Guide:**
- Security issues → Launch `security-reviewer` agent
- Missing tests → Launch `tdd-guide` agent
- Code quality → Launch `code-reviewer` agent
- Refactoring → Launch `refactor-cleaner` agent
- Build issues → Launch `build-error-resolver` agent

**Parallel vs Sequential:**
- Independent fixes in different files → Launch agents in PARALLEL
- Dependent changes (e.g., refactor then test) → Launch SEQUENTIALLY

Example parallel execution:
```
Comment 1: Fix XSS in auth.ts (CRITICAL - user confirmed)
Comment 2: Add tests for utils.ts (HIGH - user confirmed)
Comment 3: Apply immutability in useAuth.ts (MEDIUM - user selected)

→ Launch 3 Task tool calls in single message:
  1. Task(security-reviewer, "Fix XSS in auth.ts")
  2. Task(tdd-guide, "Add tests for utils.ts")
  3. Task(general-purpose, "Apply immutability in useAuth.ts")
```

Note: If user deselects a CRITICAL/HIGH item, respect their decision but note it in the summary report.

### 5. Implement Changes

For each user-selected fix (regardless of priority):
1. Use Task tool to launch appropriate agent
2. Provide agent with:
   - Review comment context
   - File path and line number
   - Specific concern to address
3. Let agent implement the fix
4. Verify fix addresses the review comment

**IMPORTANT**: When launching multiple agents for independent tasks, use a SINGLE message with multiple Task tool calls to run them in parallel.

**Note**: If user deselected a CRITICAL/HIGH item, document the reason in the summary (e.g., "incorrect review", "out of scope", "will address in separate PR").

### 6. Verify All Changes

After all fixes implemented:

Run build:
```bash
npm run build || yarn build || pnpm build
```

Run tests:
```bash
npm test || yarn test || pnpm test
```

If build/tests fail:
- Launch `build-error-resolver` agent
- Fix issues
- Re-run verification

### 7. Commit Changes

Create detailed commit message listing all addressed review comments:

```bash
git add <modified-files>

git commit -m "$(cat <<'EOF'
fix: address PR review feedback (#$PR_NUMBER)

Addressed review comments:
- [@reviewer1] Security: Fixed XSS vulnerability in src/auth/login.ts:42
- [@reviewer1] Security: Added input sanitization for user data
- [@reviewer2] Tests: Added missing test coverage for parser utility
- [@reviewer2] Tests: Implemented 12 new unit tests (coverage 85% → 94%)
- [@reviewer3] Performance: Optimized database query in src/api/users.ts:89
- [@reviewer3] Performance: Eliminated N+1 query, added proper indexing
- [@reviewer4] Style: Applied immutability patterns in src/hooks/useAuth.ts

Deferred/Not Applicable:
- [@reviewer5] Refactoring suggestion - Will address in separate PR
- [@reviewer6] Documentation - Out of scope for this PR

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
EOF
)"
```

Push changes:
```bash
git push
```

### 8. Trigger Gemini Review

After successful push, add comment to PR:
```bash
gh pr comment $PR_NUMBER --body "/gemini review"
```

Note: GitHub Copilot cannot be triggered via CLI. If "Review new pushes" is enabled in repository rulesets, Copilot will automatically review. Otherwise, manually click the re-request review button in the UI.

### 8a. Review Cycle Repetition (with --max-cycles)

**IMPORTANT: This step ONLY runs when `--max-cycles N` is specified (where N > 1).**

After triggering Gemini review, wait for new reviews and repeat the cycle:

**Cycle Loop Logic:**
```
current_cycle = 1
max_cycles = N (from --max-cycles argument, default 1)

while current_cycle <= max_cycles:
  1. Wait for AI reviews (Gemini/Copilot/Claude)
     - Poll every 2-3 minutes for new review comments
     - Timeout after 15 minutes if no reviews arrive
     - If timeout, assume no more reviews needed, exit loop

  2. Check if new reviews exist
     - Fetch latest review comments from PR
     - Compare with previously addressed comments
     - If no new reviews, exit loop (reviews complete)

  3. Validate new reviews (same as step 2a)
     - AI validates each new comment
     - Categorize as VALID/INVALID/UNCLEAR/OUT_OF_SCOPE

  4. If no VALID reviews to address:
     - All reviews are invalid or already addressed
     - Exit loop (cycle complete)

  5. Implement fixes for VALID reviews
     - Same process as steps 4-7
     - Launch agents, fix code, verify, commit, push

  6. Trigger Gemini review again
     - gh pr comment --body "/gemini review"

  7. Increment cycle counter
     - current_cycle++
     - Continue loop

end while
```

**Wait Strategy:**
```bash
# After triggering review, wait for AI response
sleep 180  # 3 minutes initial wait

# Poll for new reviews
for i in {1..5}; do
  NEW_REVIEWS=$(gh pr view $PR_NUMBER --json reviews | jq '.reviews | length')
  if [ "$NEW_REVIEWS" -gt "$PREVIOUS_COUNT" ]; then
    # New reviews detected, proceed to validation
    break
  fi
  sleep 120  # Wait 2 more minutes
done

# If no new reviews after 15 minutes total, assume complete
```

**Cycle Tracking:**
```
🔄 Review Cycle 1/3
   ✓ 4 issues fixed
   ✓ Pushed and triggered re-review
   ⏳ Waiting for new reviews...

🔄 Review Cycle 2/3
   ✓ 2 new issues fixed
   ✓ Pushed and triggered re-review
   ⏳ Waiting for new reviews...

🔄 Review Cycle 3/3
   ✓ 1 new issue fixed
   ✓ Pushed and triggered re-review
   ⏸️ Max cycles reached, stopping

✅ Review cycles complete after 3 iterations
```

**Exit Conditions:**
- Max cycles reached (`current_cycle > max_cycles`)
- No new reviews after 15-minute timeout
- No VALID reviews to address (all INVALID or already fixed)
- Build or tests fail (requires manual intervention)

### 9. Summary Report

Provide completion summary based on mode:

**Manual Mode Summary:**
```
✅ PR Review Response Complete (Manual Mode)

📋 PR #$PR_NUMBER: [Title]
🔗 $PR_URL

📊 Review Comments Processed:
- 2 CRITICAL → 2 Fixed (user confirmed)
- 3 HIGH → 3 Fixed (user confirmed)
- 2 MEDIUM → 1 Fixed (user selected), 1 Deferred
- 1 LOW → Deferred (not selected)

🔧 Changes Made:
- Fixed XSS vulnerability in src/auth/login.ts:42 (security-reviewer)
- Added 12 unit tests for src/utils/parser.ts (tdd-guide)
- Optimized database query in src/api/users.ts:89 (code-reviewer)
- Applied immutability patterns in src/hooks/useAuth.ts

⏭️ Deferred/Not Applicable:
- [@reviewer] Documentation improvement in src/api/client.ts (out of scope)
- [@reviewer] Optional refactoring suggestion in src/utils/helper.ts (will address separately)

✅ Build: Passing
✅ Tests: Passing (94% coverage)
✅ Changes: Committed and pushed
✅ Gemini Review: Triggered

PR ready for re-review!

Note: Copilot re-review requires manual UI action or automatic "Review new pushes" ruleset.
```

**Auto Mode Summary (Single Cycle):**
```
✅ PR Review Response Complete (Auto Mode)

📋 PR #$PR_NUMBER: [Title]
🔗 $PR_URL

🤖 AI Validation Results:
- 2 CRITICAL → 2 VALID → Fixed automatically
- 3 HIGH → 2 VALID → Fixed, 1 INVALID (incorrect review) → Skipped
- 2 MEDIUM → 2 VALID → Skipped (MEDIUM priority)
- 1 LOW → 1 OUT_OF_SCOPE → Issue #456 created

🔧 Changes Made (4 fixes):
- Fixed XSS vulnerability in src/auth/login.ts:42 (security-reviewer)
- Added 12 unit tests for src/utils/parser.ts (tdd-guide)
- Optimized database query in src/api/users.ts:89 (code-reviewer)
- Applied immutability patterns in src/hooks/useAuth.ts

⏭️ Skipped (3 items):
- [@reviewer] Performance issue in users.ts - INVALID: Code already optimized
- [@reviewer] Documentation in client.ts - MEDIUM priority, deferred
- [@reviewer] Refactoring in helper.ts - OUT_OF_SCOPE: Created issue #456

✅ Build: Passing
✅ Tests: Passing (94% coverage)
✅ Changes: Committed and pushed
✅ Gemini Review: Triggered

PR ready for re-review!
```

**Auto Mode Summary (Multiple Cycles):**
```
✅ PR Review Cycles Complete (Auto Mode - 3 cycles)

📋 PR #$PR_NUMBER: [Title]
🔗 $PR_URL

🔄 Cycle Summary:

Cycle 1/3:
  🤖 Validation: 7 reviews → 4 VALID, 3 INVALID/SKIPPED
  🔧 Fixed: 4 issues (XSS, tests, performance, style)
  ✅ Pushed and triggered re-review

Cycle 2/3:
  🤖 Validation: 3 new reviews → 2 VALID, 1 INVALID
  🔧 Fixed: 2 issues (error handling, edge case)
  ✅ Pushed and triggered re-review

Cycle 3/3:
  🤖 Validation: 1 new review → 1 VALID
  🔧 Fixed: 1 issue (naming convention)
  ✅ Pushed and triggered re-review
  ⏸️ Max cycles reached

📊 Total Across All Cycles:
- Reviews validated: 11
- Issues fixed: 7
- Issues skipped: 4 (2 INVALID, 2 OUT_OF_SCOPE)
- Commits created: 3
- Issues created: 2 (#456, #457)

✅ Build: Passing
✅ Tests: Passing (96% coverage)
✅ Final review cycle triggered

PR ready for final review!

💡 Tip: Run `/review-pr --auto --max-cycles 2` again if more reviews arrive.
```

## Error Handling

**No PR found:**
```
❌ No PR found for current branch
Run this command with PR number: /review-pr 123
```

**Build failures:**
- Launch `build-error-resolver` agent
- Fix incrementally
- Retry build

**Test failures:**
- Launch `tdd-guide` agent
- Analyze and fix failing tests
- Ensure no regressions

**Unclear review comments:**
- List ambiguous comments
- Ask user for clarification
- Skip unclear items for now

**Can't auto-fix:**
- Create TODO list for manual fixes
- Document what couldn't be automated
- Provide guidance for user

## Best Practices

**Auto Mode (Default):**
1. **Validate rigorously**: AI must read actual code to validate each review
2. **Trust validation**: If AI marks review as INVALID, skip it confidently
3. **Create issues for out-of-scope**: Don't skip good suggestions, track them
4. **Document validation reasons**: Clear explanations for skipped reviews
5. **Multiple cycles**: Use `--max-cycles 3` for iterative refinement
6. **Monitor progress**: Periodically check validation decisions are accurate
7. **Fully autonomous**: No user intervention needed, runs completely unattended

**Interactive Mode (--interactive):**
1. **Communicate first**: Before implementing, summarize the analysis and plan
2. **User confirms everything**: Present ALL review comments for user selection
3. **Recommend but don't force**: Pre-select CRITICAL/HIGH but allow deselection
4. **Respect user decisions**: If CRITICAL/HIGH deselected, trust user judgment
5. **Document clearly**: Reference both addressed and deferred items in commit messages
6. **Be transparent**: Explain why items were deferred (incorrect, out of scope, separate PR)

**Both Modes:**
1. **Batch intelligently**: Group related fixes in single commits
2. **Test thoroughly**: Always verify build and tests pass
3. **Collaborate smartly**: Use specialized agents for their expertise
4. **Run in parallel**: Launch independent agents concurrently

## Usage Examples

**Auto Mode (Default - Fully Autonomous):**
```bash
# AI validates and auto-fixes valid CRITICAL/HIGH issues
/review-pr                      # Current branch's PR, single cycle
/review-pr 123                  # Specific PR number, single cycle
/review-pr --max-cycles 3       # Up to 3 review cycles (recommended)
/review-pr 123 --max-cycles 2   # Specific PR, 2 cycles

# The command will:
# 1. Fetch all review comments
# 2. AI validates each review (read actual code)
#    - VALID: Review is correct, issue exists
#    - INVALID: Review is incorrect, code is fine
#    - OUT_OF_SCOPE: Valid but separate concern
# 3. Auto-fix VALID CRITICAL/HIGH issues (no user confirmation)
# 4. Skip INVALID reviews (document why)
# 5. Create GitHub issues for OUT_OF_SCOPE items
# 6. Verify build and tests
# 7. Commit and push
# 8. Trigger /gemini review
# 9. If --max-cycles > 1:
#    - Wait for new reviews (15 min timeout)
#    - Repeat steps 1-8 for new reviews
#    - Continue until max cycles or no new reviews
# ✅ FULLY AUTONOMOUS - No user intervention needed
```

**Interactive Mode (User Confirmation):**
```bash
# User confirms which reviews to address
/review-pr --interactive              # Current branch's PR
/review-pr 123 --interactive          # Specific PR number

# The command will:
# 1. Fetch all review comments
# 2. Categorize by severity (CRITICAL/HIGH/MEDIUM/LOW)
# 3. Present ALL comments to user for selection
#    - CRITICAL/HIGH pre-selected (recommended)
#    - MEDIUM/LOW not selected (optional)
#    - User can deselect any item if review is incorrect
# 4. Launch appropriate agents in parallel for selected fixes
# 5. Verify build and tests
# 6. Commit and push (document deferred items)
# 7. Trigger /gemini review
# 8. Report what was fixed and what was deferred
```

**Mixed Mode (Interactive for MEDIUM/LOW only):**
```bash
# Auto-fix CRITICAL/HIGH, ask about MEDIUM/LOW
/review-pr --interactive-medium

# The command will:
# 1. AI validates CRITICAL/HIGH → auto-fix if VALID
# 2. User confirms MEDIUM/LOW → fix if selected
# 3. Best of both: automation + control
```

## Example Workflows

### Auto Mode Workflow - Default (Single Cycle)

```
$ /review-pr

📋 Analyzing PR #456 reviews...

Found 7 review comments. Validating each...

🤖 AI Validation in Progress:

[CRITICAL] XSS vulnerability in auth.ts:42
  → Reading src/auth/login.ts:42...
  → Analysis: No input sanitization found
  → Verdict: ✅ VALID - Will fix

[HIGH] Missing tests in utils/parser.ts
  → Reading test files...
  → Analysis: Only 45% coverage, missing edge cases
  → Verdict: ✅ VALID - Will fix

[HIGH] Performance issue in api/users.ts:89
  → Reading src/api/users.ts:89...
  → Analysis: Already using SELECT with JOIN, no N+1 query
  → Verdict: ❌ INVALID - Review is incorrect, code is optimized

[MEDIUM] Code style in useAuth.ts:45
  → Reading src/hooks/useAuth.ts:45...
  → Analysis: Uses mutation, should use immutable pattern
  → Verdict: ✅ VALID - But MEDIUM priority, skipping

[MEDIUM] Documentation in api/client.ts:12
  → Analysis: API design discussion, not code issue
  → Verdict: 📋 OUT_OF_SCOPE - Creating issue #458

📊 Validation Complete:
- 2 CRITICAL/HIGH VALID → Will fix automatically
- 1 CRITICAL/HIGH INVALID → Skip (incorrect review)
- 2 MEDIUM VALID → Skip (lower priority)
- 1 OUT_OF_SCOPE → Issue created

🔧 Fixing 2 validated issues...

[1/2] Launching security-reviewer agent for XSS fix...
[2/2] Launching tdd-guide agent for test coverage...

✅ All fixes applied, tested, and pushed
✅ /gemini review triggered
✅ Issue #458 created for out-of-scope item

No user intervention required!
```

### Auto Mode Workflow (Multiple Cycles)

```
$ /review-pr --max-cycles 3

📋 Starting auto-review cycles (max 3)...

🔄 Cycle 1/3
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📋 Found 7 review comments

🤖 Validating...
  ✅ 4 VALID (CRITICAL/HIGH)
  ❌ 2 INVALID
  📋 1 OUT_OF_SCOPE

🔧 Fixing 4 issues...
  ✓ XSS vulnerability fixed
  ✓ Tests added (coverage 45% → 87%)
  ✓ Error handling improved
  ✓ Edge case handled

✅ Committed and pushed
✅ /gemini review triggered
⏳ Waiting for new reviews (timeout: 15 min)...

🔄 Cycle 2/3
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📋 Found 3 new review comments

🤖 Validating...
  ✅ 2 VALID (HIGH)
  ❌ 1 INVALID

🔧 Fixing 2 issues...
  ✓ Null check added
  ✓ Type narrowing improved

✅ Committed and pushed
✅ /gemini review triggered
⏳ Waiting for new reviews...

🔄 Cycle 3/3
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📋 Found 1 new review comment

🤖 Validating...
  ✅ 1 VALID (MEDIUM) → Skipping (MEDIUM priority)

⏸️ No CRITICAL/HIGH issues to fix
⏸️ Max cycles reached (3/3)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ Review Cycles Complete!
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📊 Summary:
- Total reviews: 11
- Fixed: 6 issues
- Skipped: 5 issues (3 INVALID, 1 OUT_OF_SCOPE, 1 MEDIUM)
- Commits: 2
- Cycles: 3

✅ Build: Passing
✅ Tests: Passing (95% coverage)
✅ PR ready for final review!
```

### Interactive Mode Workflow (User Confirmation)

```
$ /review-pr --interactive

📋 Analyzing PR #456 reviews...

Found 7 review comments across all priorities.

┌─ Review Comment Selection ─────────────────────────────┐
│ Which review comments should be addressed?              │
│                                                          │
│ CRITICAL (Recommended):                                  │
│ ☑ [CRITICAL] XSS vulnerability in auth.ts              │
│                                                          │
│ HIGH (Recommended):                                      │
│ ☑ Missing tests in utils/parser.ts                     │
│ ☐ Performance issue in api/users.ts (incorrect review) │
│                                                          │
│ MEDIUM:                                                  │
│ ☑ Code style in useAuth.ts                             │
│ ☐ Documentation in api/client.ts (out of scope)        │
│                                                          │
│ LOW:                                                     │
│ ☐ Alternative approach suggestion                       │
└──────────────────────────────────────────────────────────┘

User deselected HIGH priority item "Performance issue"
Reason: Review is incorrect, current implementation is optimal.

🔧 Fixing 3 selected issues...

[1/3] Launching security-reviewer agent for XSS fix...
[2/3] Launching tdd-guide agent for test coverage...
[3/3] Applying immutability patterns...

✅ All fixes applied, tested, and pushed
✅ /gemini review triggered

Deferred items documented in commit message:
- [@reviewer] Performance issue - INVALID (code already optimized)
- [@reviewer] Documentation - OUT_OF_SCOPE (not selected)
- [@reviewer] Alternative approach - LOW priority (not selected)
```

## Agent Collaboration Examples

**Example 1: Parallel Independent Fixes**
```
Reviews found:
- Security issue in auth.ts
- Missing tests in utils.ts
- Performance issue in cache.ts

Strategy: Launch 3 agents in parallel
→ Single message with 3 Task calls
```

**Example 2: Sequential Dependent Fixes**
```
Reviews found:
- Refactor component structure
- Add tests for new structure

Strategy: Sequential execution
→ First: refactor-cleaner agent
→ Then: tdd-guide agent (after refactor)
```

**Example 3: Mixed Strategy**
```
Reviews found:
- Security issue in auth.ts (independent)
- Refactor utils.ts (dependency step 1)
- Test utils.ts (dependency step 2)

Strategy: Hybrid
→ Parallel: security-reviewer for auth.ts
→ Sequential: refactor-cleaner then tdd-guide for utils.ts
```
