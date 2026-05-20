# Fix-Comment Command

You are the fix-comment orchestrator for a GitLab work item. Your job is to run **ONE pass** of addressing GitLab review comments, then **exit**.

This command runs **unattended on a disposable VM**, invoked by an external scheduler (e.g. `wi-scheduler`) once per `Note Hook` event (with kill-and-restart semantics when comments arrive close together). You are stateless: read everything from the positional args, do the work, post the reply, exit. The scheduler tracks `last_seen_note_id` and feeds it back in on the next invocation.

## Input

Four positional args (the scheduler passes these every tick):

1. `WI_IID` — the GitLab work item IID
2. `WORKTREE_PATH` — absolute path to the worktree on the VM (e.g. `/workspace/.claude/worktrees/wi-7-some-feature`)
3. `BRANCH` — the source branch name (e.g. `feat/wi-7-some-feature`)
4. `LAST_SEEN_NOTE_ID` — the highest note ID the bot has already processed for this issue. May be `0` on the first invocation.

### Required environment

- `GITLAB_TOKEN` — personal access token with `api` scope. If missing, log to stdout and exit immediately; the scheduler will retry on the next event.
- The project provides:
  - `./do check` — the full verification command. Exits 0 on success.
  - `./do test-deploy <WI_IID>` — redeploys the test/preview env for this work item. Prints one URL to stdout on success.

The project path (URL-encoded `namespace/repo`) and bot user ID are derived at the start of the tick — see Setup below.

## Setup (every tick — no caching across ticks)

```bash
cd "$WORKTREE_PATH"
PROJECT_URL=$(git config --get remote.origin.url)
# Parse PROJECT_URL → namespace/repo, URL-encode `/` → `%2F` → PROJECT
BOT_USER_ID=$(curl -s -H "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  "https://gitlab.com/api/v4/user" | jq -r '.id')
```

Also read CLAUDE.md to pick up project conventions, including the user-facing language for GitLab comments (default English if not specified).

## Three Types of Sub-Agents

### 1. Implementation Sub-Agent

Receives a phase description (here: the review-comment batch) and implements it. Instructions:

```
Working directory: [absolute path to worktree]

IMPORTANT: You MUST cd into the working directory above before doing ANY work. All file paths are relative to that directory.

Implement the following change:

[paste the phase description — see Step 3 below]

Context:
- Read CLAUDE.md for project conventions
- Read any files you need to modify before editing them
- Follow existing patterns in the codebase

Rules:
- First run: cd [worktree path]
- Do NOT run ./do check
- Do NOT create any commits
- NEVER ask questions or use AskUserQuestion
- Report back what you implemented when done
```

### 2. Verification Sub-Agent

```
Working directory: [absolute path to worktree]

IMPORTANT: You MUST cd into the working directory above before doing ANY work.

Run the project's full verification command, capturing all output to a unique log file:

cd [worktree path]
LOG_FILE="/tmp/do-check-$(date +%s)-$$.log"
./do check 2>&1 | tee "$LOG_FILE"

Report back with one of:
1. PASS — All checks passed. Include the log file path.
2. CODE FAILURE — Lint errors, build errors, or test failures. Include the log file path.
3. INFRASTRUCTURE FAILURE — Network issues, missing tools, permission errors, environment problems. Include the log file path.

IMPORTANT:
- Always include the log file path in your response. Do NOT paste the full output — the log file is the source of truth.
- NEVER ask questions or use AskUserQuestion — just report the result.
```

### 3. Fix Sub-Agent

```
Working directory: [absolute path to worktree]

IMPORTANT: You MUST cd into the working directory above before doing ANY work. All file paths are relative to that directory.

The code has verification errors. Your task is to fix ALL errors.

The verification log is at: [log file path]

Instructions:
1. First run: cd [worktree path]
2. Read the log file to understand what failed
3. Identify which files have errors
4. Read those files to understand the issues
5. Fix ALL errors in the code
6. Report back what you fixed

Rules:
- Do NOT run ./do check
- Do NOT create any commits
- NEVER ask questions or use AskUserQuestion — just fix and report
```

## Tick Logic

### Step 1: Find unprocessed comments

```bash
NOTES=$(curl -s -H "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  "https://gitlab.com/api/v4/projects/${PROJECT}/issues/${WI_IID}/notes?sort=asc&per_page=100")
```

Filter for notes where ALL of:

- `.system == false` (skip GitLab automation events)
- `.author.id != BOT_USER_ID` (skip our own notes)
- `.id > LAST_SEEN_NOTE_ID`

Sort the filtered list by `.id` ascending (oldest first). Capture as `UNPROCESSED`.

- **If `UNPROCESSED` is empty:** no work this tick. Exit. The scheduler will not advance `last_seen_note_id` and will react to the next event.
- **Else:** continue.

### Step 2: Acknowledge

Post one ack covering all unprocessed comments (regardless of count). Use the user-facing language from CLAUDE.md. English default:

```bash
COUNT=$(echo "$UNPROCESSED" | jq 'length')
if [ "$COUNT" -eq 1 ]; then
  BODY="🛠️ Working on the review comment…"
else
  BODY="🛠️ Working on ${COUNT} review comments…"
fi

curl -s -X POST -H "PRIVATE-TOKEN: $GITLAB_TOKEN" -H "Content-Type: application/json" \
  "https://gitlab.com/api/v4/projects/${PROJECT}/issues/${WI_IID}/notes" \
  --data-raw "$(jq -n --arg body "$BODY" '{body: $body}')"
```

If the project's CLAUDE.md prescribes German, post `"🛠️ Arbeite an der Anmerkung…"` / `"🛠️ Arbeite an ${COUNT} Anmerkungen…"`. Same pattern for any other language.

### Step 3: Implement the fix

Implementation sub-agent phase description — combine all unprocessed comments into one task:

```
The user left these review comment(s) on the GitLab work item. Address all of them in one minimal change.

<for each comment in UNPROCESSED, in order:>
Comment by @<author.username> (note id <id>):
> <body>

</for each>

Rules:
- First run: cd <WORKTREE_PATH>
- Make the minimal change addressing every comment above.
- Do NOT refactor surrounding code.
- Do NOT run ./do check.
- Do NOT create commits.
- NEVER ask questions or use AskUserQuestion.
```

Run Implementation → Verification → (Fix×3) cycle:

- **PASS** → proceed to Step 4a (commit + push + redeploy + reply).
- **CODE FAILURE** → launch fix sub-agent with the log file path, then re-verify. Repeat up to 3 cycles. If still failing → Step 4b.
- **INFRASTRUCTURE FAILURE** → Step 4c.

### Step 4a: Verification PASSED

1. Commit:
   ```bash
   cd "$WORKTREE_PATH" && git add -A && git commit -m "$(cat <<'EOF'
   Address review comments

   <one-line summary referencing the highest comment id in the batch>
   EOF
   )"
   ```

2. Push: `git push`

3. Redeploy:
   ```bash
   DEPLOY_OUTPUT=$(./do test-deploy ${WI_IID} 2>&1) || DEPLOY_FAILED=true
   ```
   Extract the new live URL from `$DEPLOY_OUTPUT`.

4. Post final reply (use CLAUDE.md language; English default):
   ```
   ✅ Fixed and redeployed: <LIVE_URL>

   Commit: <SHA>
   ```
   If deploy failed:
   ```
   ⚠️ Code change pushed, but deploy failed.

   Commit: <SHA>
   Deploy log: <log excerpt>
   ```

5. Exit. The scheduler will update its `last_seen_note_id` to the newest id in `UNPROCESSED` after the tmux session ends.

### Step 4b: Verification FAILED after 3 fix cycles

1. Post failure comment (CLAUDE.md language; English default):
   ```
   ❌ Could not address the review comment(s).

   Last verification: code-failure-after-3-cycles
   Log: <log-path>
   Branch: <BRANCH> (changes NOT pushed — verification failed)
   Worktree: <WORKTREE_PATH>

   Please review manually or leave a follow-up comment with more context.
   ```

2. Add the `failed` label:
   ```bash
   curl -s -X PUT -H "PRIVATE-TOKEN: $GITLAB_TOKEN" \
     "https://gitlab.com/api/v4/projects/${PROJECT}/issues/${WI_IID}?add_labels=failed"
   ```

3. Exit. The scheduler will see the `failed` label and stop dispatching further ticks for this work item.

### Step 4c: Verification FAILED with infrastructure error

1. Post failure comment (CLAUDE.md language; English default):
   ```
   ❌ Infrastructure error while verifying the review comment(s).

   Log: <log-path>
   Branch: <BRANCH> (changes NOT pushed)
   Worktree: <WORKTREE_PATH>
   ```

2. Add `failed` label (same curl as 4b).

3. Exit.

## Failure Reporting

Any path that exits without a successful fix:

1. Posts a comment via `POST /issues/<IID>/notes` in the project's user-facing language. Reason class is one of: `code-failure-after-3-cycles`, `infrastructure-failure`, `git-push-failed`, `deploy-failed` (note: deploy-failed does NOT add the `failed` label — push happened, future events can still trigger merge).
2. Adds the `failed` label via `PUT /issues/<IID>?add_labels=failed` (except for `deploy-failed`).

## Critical Rules

### The /fix-comment Orchestrator NEVER:

- Asks questions — NEVER use `AskUserQuestion`. Fully unattended.
- Checks MR approval state — that's an MR Hook concern handled by the scheduler, not by this command.
- Invokes `/merge` — the user merges manually, the bot does not.
- Performs a production deploy — only `./do test-deploy` is allowed.
- Runs `./do check` directly — delegate to verification sub-agents.
- Implements code changes — delegate to implementation sub-agents.
- Updates any state file — there is none; the scheduler owns all cross-tick state.
- Re-processes notes with `id <= LAST_SEEN_NOTE_ID`.
- Acts on system notes (`.system == true`) or its own bot comments.
- Pushes commits without verification passing first.

### The /fix-comment Orchestrator ALWAYS:

- Reads `LAST_SEEN_NOTE_ID` from positional args (not from any file).
- Reads CLAUDE.md at the start to pick up project conventions, including the user-facing language for GitLab comments.
- Acknowledges the batch with one "🛠️ Working on…" reply (in the project's language) BEFORE starting the fix.
- Replies with the new live URL after a successful fix-and-redeploy.
- Posts a failure comment in the project's language before exiting on errors.
- Operates inside `WORKTREE_PATH` for all git and verification work — never in the main working directory.
- Returns cleanly after the tick — does not block, does not sleep, does not loop.

## Hardcoded Operational Decisions

| Scenario | Behavior |
|----------|----------|
| `GITLAB_TOKEN` not set | Log to stdout, exit. Scheduler will retry on next event. |
| No unprocessed comments | Exit silently. `last_seen_note_id` unchanged. |
| All comments are bot-authored or system | Treated as "no unprocessed." Exit. |
| Comment author is the bot itself | Skip individual note. Continue with the rest. |
| One unprocessed comment | Single-comment ack. One commit. One redeploy. One reply. |
| Multiple unprocessed comments in one tick | "Working on N…" ack. Combined into one fix, one commit, one redeploy, one reply. |
| Fix fails after 3 cycles | Post failure comment (`code-failure-after-3-cycles`), add `failed` label, exit. |
| Verification infrastructure error | Post failure (`infrastructure-failure`), add `failed` label, exit. |
| `git push` fails after a fix | Post failure (`git-push-failed`), add `failed` label, exit. Local commits remain in worktree. |
| `./do test-deploy` fails after a fix | Post `deploy-failed` warning reply. Do NOT add `failed` label — push happened. |
