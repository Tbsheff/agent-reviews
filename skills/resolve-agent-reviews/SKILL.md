---
name: resolve-agent-reviews
description: Investigate PR review comments on the current PR, verify each finding against the code, and present a human-readable triage packet with recommended actions. Do not make code changes, post GitHub replies, resolve threads, commit, push, or start/watch/process review loops until a human explicitly approves the specific action.
license: MIT
compatibility: Requires git, gh (GitHub CLI), and Node.js installed.
allowed-tools: Bash(npx agent-reviews --bots-only --unanswered --expanded) Bash(pnpm dlx agent-reviews --bots-only --unanswered --expanded) Bash(yarn dlx agent-reviews --bots-only --unanswered --expanded) Bash(bunx agent-reviews --bots-only --unanswered --expanded) Bash(git config --global --get user.email)
metadata:
  author: Tbsheff
  version: "1.0.2"
  homepage: https://github.com/Tbsheff/agent-reviews
---

Triage review comments on the current PR. Be proactive in investigation only: fetch comments, inspect the referenced code, verify whether each finding is valid, and recommend the smallest safe next action. Stop after presenting the triage packet. Do not edit files, post GitHub replies, resolve threads, stage, commit, push, or start/watch/process follow-up review loops until a human explicitly approves the specific action.

## Prerequisites

All commands below use `npx agent-reviews`. If the project uses a different package manager, substitute the appropriate runner (e.g., `pnpm dlx agent-reviews` for pnpm, `yarn dlx agent-reviews` for Yarn, `bunx agent-reviews` for Bun). Honor the user's package manager preference throughout.

**Cloud environments only** (e.g., Codespaces, remote agents): verify git author identity so CI checks can map commits to the user. Run `git config --global --get user.email` and if empty or a placeholder, set it manually. Skip this check in local environments.

## Phase 1: FETCH & TRIAGE (synchronous)

### Step 1: Fetch All Bot Comments (Expanded)

Run `npx agent-reviews --bots-only --unanswered --expanded`

The CLI auto-detects the current branch, finds the associated PR, and authenticates via `gh` CLI or environment variables. If anything fails (no token, no PR, CLI not installed), it exits with a clear error message.

This shows only unanswered bot comments with full detail: complete comment body (no truncation), diff hunk (code context), and all replies. Each comment shows its ID in brackets (e.g., `[12345678]`).

If zero comments are returned, print "No unanswered bot comments found" and stop. Do not start watch mode unless the user explicitly approves watching for new bot comments.

### Step 2: Evaluate Each Finding

For each comment from the expanded output, read the referenced code and determine:

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

**Likely UNCERTAIN:**
- The fix would require architectural changes
- The behavior may be intentional business logic
- Multiple valid interpretations exist
- The fix could have unintended side effects

### Step 3: Present Findings and Wait

After triage, present all findings to the user in one triage packet. Include clear true positives, likely false positives, already-addressed comments, and uncertain items. The agent may recommend a default path, but the user must select or approve the path before any PR-visible action.

Use this structure:

```text
## PR Review Bot Triage

### Recommendation
I recommend Option {A/B/C}: {short rationale}.

### Options
1. Approve recommended path
   - Fixes: {comment IDs}
   - Reply-only / resolve: {comment IDs}
   - Leave open / ask author: {comment IDs}
   - Tradeoff: {risk and benefit}

2. Approve only low-risk actions
   - Fixes: {comment IDs}
   - Reply-only / resolve: {comment IDs}
   - Leaves open: {uncertain / high-risk comment IDs}
   - Tradeoff: {risk and benefit}

3. Override per comment
   - Reply with comment IDs and action: fix, reply-only, skip, leave open, or ask author
   - Tradeoff: Most control, slower closeout

### Findings
#### {comment_id} - {bot} - {classification}
- Claim: {what the bot says}
- Evidence: {code facts you verified}
- Recommended action: {fix / reply-only / skip / leave open / ask author}
- Tradeoffs: {risk of fixing vs not fixing}
- Recommended: {specific recommendation}
```

Ask the user to choose an option or provide per-comment actions. Then stop and wait for the user response before editing code, replying, resolving, committing, pushing, or continuing to watch.

Use the host agent's structured question tool for this checkpoint when available:
- In Claude Code, use `AskUserQuestion` / `Question` with the numbered options from the triage packet.
- In Codex, use the equivalent `request_user_input` tool when it is available. If it is unavailable because the session is not in the mode that exposes it, switch to that mode first, then call `request_user_input`. Only fall back to numbered options in chat if the current Codex environment cannot switch modes or still does not expose the tool.

Do not proceed without user selection. Before user selection, do not:
- Edit code
- Reply to comments
- Resolve review threads
- Commit
- Push
- Start or restart the watcher

## Phase 2: EXECUTE HUMAN-APPROVED OUTCOMES ONLY

Execute only the specific actions the human approved. If the user approved fixes, make the smallest safe code changes and run verification. Stop again before posting GitHub replies, resolving threads, committing, pushing, or watching unless those actions were also explicitly approved.

Do not treat approval to fix as approval to reply, resolve, commit, push, or watch. Each category needs explicit approval.

**For TRUE POSITIVE fixes:**
1. Fix the code with the smallest safe change
2. Run the project's lint and type-check
3. Stop and summarize the local changes and verification result if commit/push approval was not already explicit
4. Only if the human explicitly approved committing and pushing, request any required tool permission, then stage, commit, and push:
   ```bash
   git add -A
   git commit -m "fix: address PR review bot findings

   {List of fixes, grouped by bot}"
   git push
   ```
5. Capture the commit hash from the output
6. Only if the human explicitly approved posting replies and resolving threads, request any required tool permission, then reply with `npx agent-reviews --reply <comment_id> "Fixed in {hash}. {Brief description of the fix}" --resolve`; otherwise provide the suggested reply text in chat only

**For FALSE POSITIVE replies:**

Only if the human explicitly approved posting this reply, request any required tool permission, then run `npx agent-reviews --reply <comment_id> "Won't fix: {reason}. {Explanation of why this is intentional or not applicable}"`. Include `--resolve` only when thread resolution was separately approved; otherwise provide the reply text in chat only.

**For skipped comments:**

Only if the human explicitly approved posting this reply, request any required tool permission, then run `npx agent-reviews --reply <comment_id> "Skipped per user request"`. Include `--resolve` only when thread resolution was separately approved; otherwise provide the reply text in chat only.

**For ask-author comments:**

Only if the human explicitly approved posting this reply, request any required tool permission, then run `npx agent-reviews --reply <comment_id> "Leaving open for author decision: {specific question or tradeoff}"` without `--resolve`; otherwise provide the reply text in chat only.

## Phase 3: WATCH ONLY WHEN EXPLICITLY APPROVED

Do not start watch mode by default.

Only run watch mode if the human explicitly approves watching for new comments. When approved, request any required tool permission, then run `npx agent-reviews --watch --bots-only` as a background task.

When watch mode returns new comments, treat them as a new triage batch: investigate and present findings, then stop for human review. Do not auto-fix, auto-reply, auto-resolve, commit, push, or continue the watch loop without explicit approval for that batch.

When the watcher exits with no new comments, move to the Summary Report.

## Summary Report

After executing the human-approved actions and any approved watch pass, provide a summary:

```text
## PR Review Bot Resolution Summary

### Actions Taken
- Fixed: X bugs
- Replied won't fix: X
- Asked author / left open: X
- Skipped per user: X

### Needs User / Left Open
- {comment_id}: {reason it remains untouched}

### Status
Completed the human-approved actions. Watch only ran if explicitly approved.
```

## Important Notes

### Response Policy
- Every recommended action must be shown to the human before execution.
- No code changes, GitHub replies, thread resolutions, commits, pushes, or watch-loop processing may happen without explicit human approval.
- A "clear" or "low-risk" finding means the recommendation can be confident. It does not grant permission to execute.
- Replies should document evidence, but only after the human approves posting them.
- "Won't fix" responses should document evidence, not just opinion

### User Interaction
- The user decision checkpoint is mandatory after every triage packet, including comments returned by watch mode
- Use Claude Code `AskUserQuestion` / `Question` or Codex `request_user_input` for the checkpoint. In Codex, switch modes to access `request_user_input` when needed and supported
- Present tradeoffs and a recommendation before asking for selection
- Do not guess on architectural or business logic questions

### Best Practices
- Verify findings before recommending action - bots have false positives
- Keep fixes minimal and focused - don't refactor unrelated code
- Ensure type-check and lint pass before committing fixes
- Group related fixes into a single commit when they belong together
- Copilot `suggestion` blocks often contain ready-to-use fixes
