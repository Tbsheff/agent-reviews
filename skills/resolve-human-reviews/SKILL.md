---
name: resolve-human-reviews
description: Investigate PR review comments on the current PR, verify each finding against the code, and present a human-readable triage packet with recommended closeout actions. Do not make code changes, post GitHub replies, resolve threads, commit, or push until a human explicitly approves the closeout plan.
license: MIT
compatibility: Requires git, gh (GitHub CLI), and Node.js installed.
allowed-tools: Bash(npx agent-reviews *) Bash(pnpm dlx agent-reviews *) Bash(yarn dlx agent-reviews *) Bash(bunx agent-reviews *) Bash(git config --global --get user.email) Bash(git add *) Bash(git commit *) Bash(git push *)
metadata:
  author: Tbsheff
  version: "1.0.3"
  homepage: https://github.com/Tbsheff/agent-reviews
---

Triage review comments on the current PR. Be proactive in investigation only until the user chooses a closeout path: fetch comments, inspect the referenced code, verify whether each finding is valid, and recommend the smallest safe next action. Stop after presenting the triage packet. Do not edit files, post GitHub replies, resolve threads, stage, commit, or push until a human explicitly approves the closeout plan.

## Prerequisites

All commands below use `npx agent-reviews`. If the project uses a different package manager, substitute the appropriate runner (e.g., `pnpm dlx agent-reviews` for pnpm, `yarn dlx agent-reviews` for Yarn, `bunx agent-reviews` for Bun). Honor the user's package manager preference throughout.

**Cloud environments only** (e.g., Codespaces, remote agents): verify git author identity so CI checks can map commits to the user. Run `git config --global --get user.email` and if empty or a placeholder, set it manually. Skip this check in local environments.

## Phase 1: FETCH & TRIAGE (synchronous)

### Step 1: Fetch All Human Comments (Expanded)

Run `npx agent-reviews --humans-only --unanswered --expanded`

The CLI auto-detects the current branch, finds the associated PR, and authenticates via `gh` CLI or environment variables. If anything fails (no token, no PR, CLI not installed), it exits with a clear error message.

This shows only unanswered human comments with full detail: complete comment body (no truncation), diff hunk (code context), and all replies. Each comment shows its ID in brackets (e.g., `[12345678]`).

If zero comments are returned, print "No unanswered human comments found" and start watch mode unless the user explicitly opted out.

### Step 2: Evaluate Each Comment

For each comment from the expanded output, read the referenced code and the reviewer's comment. Human reviewers are generally more accurate and context-aware than bots, but their feedback can still involve tradeoffs. Determine:

1. **ACTIONABLE** - The reviewer identified a real issue or requested a concrete change
2. **DISCUSSION** - The comment raises a valid point but the right approach is unclear
3. **ALREADY ADDRESSED** - The concern has already been fixed or is no longer relevant

**Likely ACTIONABLE:**
- Reviewer points out a bug or logic error
- Reviewer requests a specific code change
- Reviewer identifies missing edge cases or error handling
- Reviewer flags a naming, API, or architectural concern with a clear fix
- Reviewer suggests a better approach with justification

**Likely DISCUSSION:**
- Reviewer suggests an architectural change with meaningful tradeoffs
- Comment involves performance vs readability, API stability vs cleanup, or similar tension
- Reviewer's suggestion conflicts with patterns used elsewhere in the codebase
- The feedback is subjective without team consensus
- You disagree with the feedback and want the author to weigh in

**Likely ALREADY ADDRESSED:**
- The code has been changed since the review was posted
- Another commit already fixed the issue
- The comment refers to code that no longer exists

### Step 3: Present Findings and Wait

After triage, present all findings to the user in one triage packet. Include clear actionable feedback, discussion items, already-addressed comments, and uncertain items. The agent may recommend a default path, but the user must select or approve the path before any PR-visible action.

Use this structure:

```text
## Human Review Triage

### Recommendation
I recommend Option {A/B/C}: {short rationale}.

### Options
1. Approve recommended closeout
   - Fixes: {comment IDs}
   - Commit/push: yes/no
   - Post replies: {comment IDs}
   - Resolve threads: {comment IDs}
   - Leave open / ask author: {comment IDs}
   - Watch after closeout: yes, unless the user opts out
   - Tradeoff: {risk and benefit}

2. Approve only low-risk actions
   - Fixes: {comment IDs}
   - Commit/push: yes/no
   - Post replies: {comment IDs}
   - Resolve threads: {comment IDs}
   - Leaves open: {uncertain / high-risk comment IDs}
   - Watch after closeout: yes, unless the user opts out
   - Tradeoff: {risk and benefit}

3. Override per comment
   - Reply with comment IDs and action: fix, reply-only, skip, or ask author
   - Tradeoff: Most control, slower closeout

### Findings
#### {comment_id} - @{reviewer} - {classification}
- Feedback: {what the reviewer asked for}
- Evidence: {code facts you verified}
- Recommended action: {fix / reply-only / skip / leave open / ask author}
- Tradeoffs: {risk of accepting vs declining}
- Recommended: {specific recommendation}
```

Ask the user to choose an option or provide per-comment actions. Then stop and wait for the user response before editing code, replying, resolving, committing, or pushing. The selected option is the approval source of truth: if it includes commit/push, replies, or thread resolution, execute those steps after the fix and verification without asking again.

Use the host agent's structured question tool for this checkpoint when available:
- In Claude Code, use `AskUserQuestion` / `Question` with the numbered options from the triage packet.
- In Codex, use the equivalent `request_user_input` tool when it is available. If it is unavailable because the session is not in the mode that exposes it, use the host's supported mode-switch mechanism when one exists, then call `request_user_input`. Only fall back to numbered options in chat if the current Codex environment cannot switch modes or still does not expose the tool.

Do not proceed without user selection. Before user selection, do not:
- Edit code
- Reply to comments
- Resolve review threads
- Commit
- Push
- Start or restart the watcher

## Phase 2: EXECUTE HUMAN-APPROVED OUTCOMES ONLY

Execute only the actions included in the human-approved closeout plan. If the selected option includes fixes, make the smallest safe code changes and run verification. If the same selected option includes commit/push, replies, or thread resolution, continue through those steps without another checkpoint.

Do not infer unlisted actions. But when the approved option explicitly lists fix, commit/push, reply, resolve, and watch behavior, treat that option as approval for the whole listed bundle.

**For ACTIONABLE fixes:**
1. Fix the code with the smallest safe change
2. Run the project's lint and type-check
3. If the approved closeout plan did not include commit/push, stop and summarize the local changes and verification result
4. If the approved closeout plan included commit/push, stage, commit, and push:
   ```bash
   git add -A
   git commit -m "fix: address PR review feedback

   {List of changes, grouped by reviewer}"
   git push
   ```
5. Capture the commit hash from the output
6. If the approved closeout plan included posting replies and resolving this thread, reply with `npx agent-reviews --reply <comment_id> "Fixed in {hash}. {Brief description of the fix}" --resolve`; otherwise provide the suggested reply text in chat only

**For DISCUSSION replies:**

If the approved closeout plan included posting this reply, run `npx agent-reviews --reply <comment_id> "{Outcome}. {Explanation of the decision and any changes made}"`; otherwise provide the reply text in chat only.

Use `--resolve` only if the user explicitly chose to resolve it.

**For ALREADY ADDRESSED replies:**

If the approved closeout plan included posting this reply, run `npx agent-reviews --reply <comment_id> "Already addressed. {Explanation of when/how this was fixed}"`. Include `--resolve` when the approved plan resolves this thread; otherwise provide the reply text in chat only.

**For skipped comments:**

If the approved closeout plan included posting this reply, run `npx agent-reviews --reply <comment_id> "Skipped per user request"`. Include `--resolve` when the approved plan resolves this thread; otherwise provide the reply text in chat only.

**For ask-author comments:**

If the approved closeout plan included posting this reply, run `npx agent-reviews --reply <comment_id> "Leaving open for author decision: {specific question or tradeoff}"` without `--resolve`; otherwise provide the reply text in chat only.

## Phase 3: WATCH AFTER CLOSEOUT

After completing the approved closeout plan, start watch mode unless the user explicitly opted out.

Run `npx agent-reviews --watch --humans-only` as a background task.

When watch mode returns new comments, treat them as a new triage batch: investigate and present findings, then stop for human review. Do not auto-fix, auto-reply, auto-resolve, commit, push, or continue the watch loop without a new approved closeout plan for that batch.

When the watcher exits with no new comments, move to the Summary Report.

## Summary Report

After executing the human-approved actions and any approved watch pass, provide a summary:

```text
## Human Review Resolution Summary

### Actions Taken
- Fixed: X issues
- Replied already addressed: X
- Asked author / left open: X
- Skipped per user: X

### Needs User / Left Open
- {comment_id}: {reason it remains untouched}

### Status
Completed the human-approved closeout plan. Watch ran unless the user opted out.
```

## Important Notes

### Response Policy
- Every recommended action must be shown to the human before execution.
- No code changes, GitHub replies, thread resolutions, commits, or pushes may happen without an approved closeout plan that lists those actions.
- A "clear" or "low-risk" finding means the recommendation can be confident. It does not grant permission to execute.
- Replies should document evidence, but only after the human approves posting them.
- Even "already addressed" comments deserve evidence

### User Interaction
- The user decision checkpoint is mandatory after every triage packet, including comments returned by watch mode
- Use Claude Code `AskUserQuestion` / `Question` or Codex `request_user_input` for the checkpoint. In Codex, switch modes to access `request_user_input` when needed and supported
- Present tradeoffs and a recommendation before asking for selection
- Human reviewers often have context you don't - defer to the author when unsure

### Best Practices
- Human reviewers are generally more accurate than bots, but still verify against the current diff
- Keep fixes minimal and focused - don't refactor unrelated code
- Ensure type-check and lint pass before committing fixes
- Group related fixes into a single commit when they belong together
- If a reviewer suggests a specific code change, prefer their version unless it introduces issues
