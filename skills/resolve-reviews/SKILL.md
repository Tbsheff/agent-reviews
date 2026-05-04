---
name: resolve-reviews
description: Triage all PR review comments (human and bot) on current PR, present findings with tradeoffs and a recommended approach, proactively execute clear low-risk fixes and replies, and ask the user only when a finding needs a tradeoff or judgment call.
license: MIT
compatibility: Requires git, gh (GitHub CLI), and Node.js installed.
allowed-tools: Bash(npx agent-reviews *) Bash(pnpm dlx agent-reviews *) Bash(yarn dlx agent-reviews *) Bash(bunx agent-reviews *) Bash(git config *) Bash(git add *) Bash(git commit *) Bash(git push *)
metadata:
  author: Tbsheff
  version: "1.0.1"
  homepage: https://github.com/Tbsheff/agent-reviews
---

Triage all review comments (both human and bot) on the current PR. Be as proactive as possible: automatically fix, reply to, resolve, commit, and push clear low-risk outcomes after verifying them. Stop for user input only when a finding involves uncertainty, meaningful tradeoffs, architectural or business logic decisions, or an action the user explicitly asked to approve.

## Prerequisites

All commands below use `npx agent-reviews`. If the project uses a different package manager, substitute the appropriate runner (e.g., `pnpm dlx agent-reviews` for pnpm, `yarn dlx agent-reviews` for Yarn, `bunx agent-reviews` for Bun). Honor the user's package manager preference throughout.

**Cloud environments only** (e.g., Codespaces, remote agents): verify git author identity so CI checks can map commits to the user. Run `git config --global --get user.email` and if empty or a placeholder, set it manually. Skip this check in local environments.

## Phase 1: FETCH & TRIAGE (synchronous)

### Step 1: Fetch All Comments (Expanded)

Run `npx agent-reviews --unanswered --expanded`

The CLI auto-detects the current branch, finds the associated PR, and authenticates via `gh` CLI or environment variables. If anything fails (no token, no PR, CLI not installed), it exits with a clear error message.

This shows all unanswered comments (both human and bot) with full detail: complete comment body (no truncation), diff hunk (code context), and all replies. Each comment shows its ID in brackets (e.g., `[12345678]`).

If zero comments are returned, print "No unanswered comments found" and continue to Phase 3 so the watcher can catch new comments.

### Step 2: Evaluate Each Comment

For each comment from the expanded output, apply the appropriate evaluation based on whether the author is a bot or a human.

#### For Bot Comments

Read the referenced code and determine:

1. **TRUE POSITIVE** - A real bug that should be fixed
2. **FALSE POSITIVE** - Not actually a bug (intentional behavior, bot misunderstanding)
3. **UNCERTAIN** - Not enough context to choose safely

**Likely TRUE POSITIVE:**
- Code obviously violates stated behavior
- Missing null checks on potentially undefined values
- Type mismatches or incorrect function signatures
- Logic errors in conditionals
- Missing error handling for documented failure cases

**Likely FALSE POSITIVE:**
- Bot doesn't understand the framework/library patterns
- Code is intentionally structured that way
- Bot is flagging style preferences, not bugs
- The "bug" is actually a feature or intentional behavior
- Bot misread the code flow

#### For Human Comments

Read the referenced code and the reviewer's comment. Human reviewers are generally more accurate and context-aware than bots, but their feedback can still involve tradeoffs. Determine:

1. **ACTIONABLE** - The reviewer identified a real issue or requested a concrete change
2. **DISCUSSION** - The comment raises a valid point but the right approach is unclear
3. **ALREADY ADDRESSED** - The concern has already been fixed or is no longer relevant

**Likely ACTIONABLE:**
- Reviewer points out a bug or logic error
- Reviewer requests a specific code change
- Reviewer identifies missing edge cases or error handling

**Likely DISCUSSION:**
- Reviewer suggests an architectural change with meaningful tradeoffs
- Comment involves performance vs readability, API stability vs cleanup, or similar tension
- The feedback is subjective without team consensus

#### When Classification Is Uncertain

For both bot and human comments:
- The fix would require architectural changes
- You're genuinely unsure if the behavior is intentional
- Multiple valid interpretations exist
- The fix could have unintended side effects

### Step 3: Execute Clear Outcomes or Ask When Needed

After triage, split findings into clear outcomes and decision-needed items. Execute clear outcomes without waiting; present a triage packet only for items that need human input. Include every decision-needed comment, grouped by recommended action.

Use this structure:

```text
## PR Review Triage

### Recommendation
I recommend Option {A/B/C}: {short rationale}.

### Options
1. Approve recommended judgment calls
   - Includes: {decision-needed comment IDs}
   - Outcome: {fix / reply-only / leave open}
   - Tradeoff: {risk and benefit}

2. Leave decision-needed items open with author questions
   - Includes: {comment IDs}
   - Outcome: reply with the unresolved decision or tradeoff
   - Tradeoff: {risk and benefit}

3. Override per comment
   - Reply with comment IDs and action: fix, reply-only, skip, or ask author
   - Tradeoff: Most control, slower closeout

### Findings
#### {comment_id} - {author} - {classification}
- Claim: {what the review comment says}
- Evidence: {code facts you verified}
- Proposed action: {fix / reply-only / skip / ask author}
- Tradeoffs: {risk of accepting vs declining}
- Recommended: {specific recommendation}
```

Ask the user only when at least one finding needs a decision. If no findings need human input, skip this checkpoint, execute the clear outcomes, and report what happened.

Use the host agent's structured question tool for this checkpoint when available:
- In Claude Code, use `AskUserQuestion` / `Question` with the numbered options from the triage packet.
- In Codex, use the equivalent `request_user_input` tool when it is available. If it is unavailable because the session is not in the mode that exposes it, switch to that mode first, then call `request_user_input`. Only fall back to numbered options in chat if the current Codex environment cannot switch modes or still does not expose the tool.

Do not ask the user for clear low-risk outcomes. Do ask before:
- Architectural or business logic changes
- Subjective style or product calls
- Fixes with meaningful side effects
- Disagreeing with a human reviewer
- Leaving a real issue unresolved

## Phase 2: EXECUTE OUTCOMES

Execute clear outcomes immediately after triage. If a decision packet was needed, execute only the actions the user selected.

**For TRUE POSITIVE / ACTIONABLE fixes:**
1. Fix the code with the smallest safe change
2. Run the project's lint and type-check
3. Stage, commit, and push:
   ```bash
   git add -A
   git commit -m "fix: address PR review findings

   {List of changes, grouped by reviewer/bot}"
   git push
   ```
4. Capture the commit hash from the output
5. Reply with `npx agent-reviews --reply <comment_id> "Fixed in {hash}. {Brief description of the fix}" --resolve`

**For FALSE POSITIVE replies:**

Run `npx agent-reviews --reply <comment_id> "Won't fix: {reason}. {Explanation of why this is intentional or not applicable}" --resolve`

**For DISCUSSION replies:**

Run `npx agent-reviews --reply <comment_id> "{Outcome}. {Explanation of the decision and any changes made}"`

Use `--resolve` only if the user explicitly chose to resolve it.

**For ALREADY ADDRESSED replies:**

Run `npx agent-reviews --reply <comment_id> "Already addressed. {Explanation of when/how this was fixed}" --resolve`

**For skipped comments:**

Run `npx agent-reviews --reply <comment_id> "Skipped per user request" --resolve`

**For ask-author comments:**

Run `npx agent-reviews --reply <comment_id> "Leaving open for author decision: {specific question or tradeoff}"` without `--resolve`.

## Phase 3: WATCH FOR NEW COMMENTS

Always start the watcher after the current batch is handled. The watcher exits immediately when new comments are found (after a 5s grace period to catch batch posts). Run it in a loop: start watcher, process any comments it returns, restart watcher, repeat until the watcher times out with no new comments.

Run `npx agent-reviews --watch` as a background task.

If new comments appear, triage them the same way: automatically handle clear low-risk outcomes and ask only for decision-needed items. Then restart the watcher.

When the watcher exits with no new comments, stop looping and move to the Summary Report.

## Summary Report

After executing clear outcomes and any user-selected actions, provide a summary:

```text
## PR Review Resolution Summary

### Actions Taken
- Fixed: X issues
- Replied won't fix / already addressed: X
- Asked author / left open: X
- Skipped per user: X

### Needs User / Left Open
- {comment_id}: {reason it remains untouched}

### Status
Completed all clear outcomes and any user-selected actions. Watch completed with no new comments.
```

## Important Notes

### Response Policy
- **Every handled action gets a response** - no silent closeout work
- Clear low-risk replies and resolutions do not need selection; decision-needed comments must not be replied to or resolved until the user chooses
- For bots: responses help train them and prevent re-raised false positives
- For humans: replies keep reviewers informed and unblock approvals

### User Interaction
- The user decision checkpoint is mandatory only when triage finds decision-needed items
- Use Claude Code `AskUserQuestion` / `Question` or Codex `request_user_input` for the checkpoint. In Codex, switch modes to access `request_user_input` when needed and supported
- Present tradeoffs and a recommendation before asking for selection; otherwise keep moving
- Don't guess on architectural or business logic questions
- Human reviewers often have context you don't - defer to the author when unsure

### Best Practices
- Verify findings before recommending action - bots have false positives, humans rarely do
- Keep fixes minimal and focused - don't refactor unrelated code
- Ensure type-check and lint pass before committing fixes
- Group related fixes into a single commit when they belong together
- Copilot `suggestion` blocks often contain ready-to-use fixes
- If a human reviewer suggests a specific code change, prefer their version unless it introduces issues
