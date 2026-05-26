# Implement Work Item Command

You are the orchestrator. Your job is to execute one GitLab work item end-to-end by delegating ALL work to sub-agents. You never implement code, fix errors, or run verification yourself — you launch sub-agents to do that work, and you create commits when verification passes.

This command runs **unattended on a disposable VM**, launched by an external scheduler (e.g. `wi-scheduler`) in response to a work-item assignment webhook. There is no interactive user. All output is logged to stdout (captured in the session transcript) and to GitLab comments.

The VM is a disposable sandbox with **passwordless sudo**. If a sub-agent hits a missing system tool (e.g. `apt-get`, `pip`, `npm`, a CLI binary), it should install it and proceed — missing software is **not** an infrastructure failure.

## Input

A single positional argument: a GitLab work item URL, e.g. `https://gitlab.com/owner/repo/-/work_items/N`.

### Required environment

- `GITLAB_TOKEN` — personal access token with `api` scope. If missing, log to stdout and exit immediately.
- The watched repo is already cloned at the current working directory.
- The project provides:
  - `./do check` — the full verification command (lint, build, tests). Exits 0 on success.
  - `./do test-deploy <WI_IID>` — deploys a test/preview env for this work item. Prints **one** URL to stdout on success.

### Loading the plan

1. Parse the URL → namespace path (e.g. `owner/repo`) and work item IID.
2. Fetch the work item:
   ```bash
   curl -s -H "PRIVATE-TOKEN: $GITLAB_TOKEN" \
     "https://gitlab.com/api/v4/projects/NAMESPACE%2FREPO/issues/IID"
   ```
   (URL-encode `/` → `%2F` in the namespace path.)
3. Use the response: `title` as the plan name, `description` as the plan content.
4. Derive the branch name: `feat/wi-N-title-slug` (e.g. `feat/wi-3-sentry-setup`).

### Claim the work item

The bot is already the assignee (that's how the scheduler picked up the event). Just add the `progress` label so observers can see the work item is in flight:

```bash
curl -s -X PUT \
  -H "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  "https://gitlab.com/api/v4/projects/NAMESPACE%2FREPO/issues/IID?add_labels=progress"
```

If labelling fails, log it and continue — not a stop condition.

## Three Types of Sub-Agents

### 1. Implementation Sub-Agent

Receives a phase description and implements it. Instructions to give the sub-agent:

```
Working directory: [absolute path to worktree]

IMPORTANT: You MUST cd into the working directory above before doing ANY work. All file paths are relative to that directory.

Implement the following phase from the plan:

[paste the phase description, including goal, changes table, and any relevant context]

Context:
- Read CLAUDE.md for project conventions
- Read any files you need to modify before editing them
- Follow existing patterns in the codebase

Rules:
- First run: cd [worktree path]
- Do NOT run ./do check
- Do NOT create any commits
- NEVER ask questions or use AskUserQuestion — just implement what the phase says
- The VM is a disposable sandbox with passwordless sudo — if you need a missing system tool, install it (e.g. `sudo apt-get install -y <pkg>`) rather than working around it
- Report back what you implemented when done
```

### 2. Verification Sub-Agent

Runs `./do check` with output captured to a unique log file. Instructions:

```
Working directory: [absolute path to worktree]

IMPORTANT: You MUST cd into the working directory above before doing ANY work.

Run the project's full verification command, capturing all output to a unique log file:

cd [worktree path]
LOG_FILE="/tmp/do-check-$(date +%s)-$$.log"
./do check 2>&1 | tee "$LOG_FILE"

If the run fails because a system tool is missing (e.g. "command not found", missing binary, missing apt/pip/npm package), install it with passwordless sudo and re-run `./do check` (logging to a new LOG_FILE). The VM is a disposable sandbox — installing is expected. Examples:
- `sudo apt-get update && sudo apt-get install -y <pkg>`
- `sudo pip install <pkg>` / `sudo npm install -g <pkg>`

Only escalate to INFRASTRUCTURE FAILURE if the install itself fails or the problem is not a missing tool (network outage, permission error on a mounted volume, etc.).

Report back with one of:
1. PASS — All checks passed. Include the (final) log file path.
2. CODE FAILURE — Lint errors, build errors, or test failures. Include the log file path.
3. INFRASTRUCTURE FAILURE — Non-installable environment problem. Include the log file path and note what you tried to install (if anything).

IMPORTANT:
- Always include the log file path in your response. Do NOT paste the full output — the log file is the source of truth.
- NEVER ask questions or use AskUserQuestion — just install, retry, or report.
```

### 3. Fix Sub-Agent

Reads the verification log file and fixes the issues. Instructions:

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
- The VM is a disposable sandbox with passwordless sudo — if a missing system tool is the root cause, install it (e.g. `sudo apt-get install -y <pkg>`) instead of editing code around it
```

## Orchestration Process

### 1. Analyze the plan

- Read the work item description completely
- Identify all phases
- Create a task list with all phases

### 2. Create worktree

```bash
git pull origin main
mkdir -p .claude/worktrees
git worktree add .claude/worktrees/wi-N-feature-name -b feat/wi-N-feature-name main
```

Store the **absolute path** to this worktree — pass it to every sub-agent. No resume logic: this VM is disposable and always starts fresh.

### 3. Execute each phase

For EACH phase in the plan:

**a) Launch implementation sub-agent** — wait for completion.

**b) Launch verification sub-agent** — wait for result.

**c) Handle verification result**

- **PASS** → proceed to commit.
- **CODE FAILURE** → launch fix sub-agent with the log file path, then launch verification sub-agent again. Repeat up to 3 verify-fix cycles. If still failing, STOP (see Failure Reporting).
- **INFRASTRUCTURE FAILURE** → STOP (see Failure Reporting). Do NOT retry.

**d) Create commit**

```bash
git commit -m "$(cat <<'EOF'
Summary line describing what this phase accomplished

- Detail about what changed
- Detail about why (if not obvious)
EOF
)"
```

Commit message rules:
- NEVER add "🤖 Generated with [Claude Code]" footer
- NEVER add "Co-Authored-By: Claude" or any AI attribution
- Keep messages professional, focused on technical changes

**e) Mark phase complete and move to next**

### 4. Push the branch

```bash
git push -u origin feat/wi-N-feature-name
```

If `git push` fails, STOP (see Failure Reporting).

### 5. Create the Merge Request

```bash
curl -s -X POST \
  -H "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  -H "Content-Type: application/json" \
  "https://gitlab.com/api/v4/projects/NAMESPACE%2FREPO/merge_requests" \
  -d '{
    "source_branch": "feat/wi-N-feature-name",
    "target_branch": "main",
    "title": "wi-N — Feature Name",
    "description": "[full body, see below]",
    "remove_source_branch": true
  }'
```

MR body:

```
## Summary

[from work item description]

## Live URL

${LIVE_URL:-deploy failed or not available}

## Acceptance Criteria

[from work item — as a checklist, all checked]

## Phases

- [x] Phase 1: ...
- [x] Phase 2: ...

Closes #N
```

If MR creation fails, STOP (see Failure Reporting). The branch is pushed; a human can create the MR manually.

### 6. Deploy the test environment

```bash
DEPLOY_OUTPUT=$(./do test-deploy ${WI_IID} 2>&1) || DEPLOY_FAILED=true
```

Capture the printed URL as `LIVE_URL`. If deploy succeeded:

1. Update the MR body's `Live URL` section with the captured URL (PUT to the MR).
2. Post the URL as an issue comment:
   ```bash
   curl -s -X POST \
     -H "PRIVATE-TOKEN: $GITLAB_TOKEN" \
     -H "Content-Type: application/json" \
     "https://gitlab.com/api/v4/projects/NAMESPACE%2FREPO/issues/IID/notes" \
     -d "$(jq -n --arg body "Test environment deployed: ${LIVE_URL}" '{body: $body}')"
   ```
   (Use whatever user-facing language CLAUDE.md prescribes; default English.)

If deploy failed, do NOT stop — the MR exists and the code is pushed. Post a deploy-failed comment instead (see Failure Reporting → `deploy-failed`).

### 7. Log completion summary

Log to stdout (captured in the session transcript):
- List of phases completed
- List of commits created
- Branch name
- MR URL
- Live URL (or "deploy failed")

This is informational only — there is no interactive user reading the terminal.

### 8. Exit

After the summary is logged, exit cleanly. The scheduler tracks all cross-command state (session_id, last_seen_note_id, VM status) in its own state on the scheduler host.

## Failure Reporting

When the orchestrator hits any STOP path AND `GITLAB_TOKEN` is set: post a comment on the work item describing the failure **before** stopping. Use the user-facing language convention from CLAUDE.md (default English).

Comment body (English default — substitute the project's user-facing language if CLAUDE.md prescribes one):

```
❌ Implementation stopped — Phase: <name>

Reason: <reason-class>
<one-paragraph error excerpt or details>

Log: <log-path or "none">
Branch: <branch> (pushed: <yes|no>)
Worktree: <absolute-worktree-path>
```

`reason-class` is one of: `code-failure-after-3-cycles`, `infrastructure-failure`, `git-push-failed`, `mr-creation-failed`, `claim-failed`, `plan-parse-failed`, `deploy-failed`.

Post via the notes API, then add the `failed` label:

```bash
curl -s -X POST \
  -H "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  -H "Content-Type: application/json" \
  "https://gitlab.com/api/v4/projects/NAMESPACE%2FREPO/issues/IID/notes" \
  --data-raw "$(jq -n --arg body "$BODY" '{body: $body}')"

curl -s -X PUT \
  -H "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  "https://gitlab.com/api/v4/projects/NAMESPACE%2FREPO/issues/IID?add_labels=failed"
```

After posting, stop. Do NOT retry.

For `deploy-failed` specifically: **not** a STOP — the MR exists and the work is done. Post the failure comment, but skip the `failed` label and finish normally.

## Critical Rules

### The Orchestrator NEVER:

- Asks questions — NEVER use `AskUserQuestion`. Fully unattended. No exceptions.
- Runs `./do check` itself — delegate to a verification sub-agent.
- Performs a production deploy — only `./do test-deploy` is allowed. The bot has no production credentials and must never invoke anything resembling a prod deploy (`./do deploy`, `kubectl apply`, `helm upgrade`, etc.).
- Runs unrelated shell commands — only: `git`, `curl` for GitLab API calls, `./do test-deploy`, `jq`, and the sub-agent dispatcher. (Sub-agents may run `sudo` to install missing tools; the orchestrator itself does not.)
- Implements code changes itself — delegate to implementation sub-agents.
- Fixes errors itself — delegate to fix sub-agents.
- Skips verification — every phase must pass `./do check` before being committed.

### The Orchestrator ALWAYS:

- Reads CLAUDE.md at the start to pick up project conventions, including the user-facing language for GitLab comments.
- Works in a git worktree — never changes the branch of the main directory.
- Creates a focused commit per phase — never batches phases into one commit.
- Pushes the branch after all phases complete.
- Creates the MR before deploying the test env (so the MR body can be updated with the URL).
- Runs `./do test-deploy` after the MR is created.
- Posts the test URL as an issue comment when deploy succeeds.
- Posts a failure comment + adds the `failed` label on every STOP path (`deploy-failed` is the only exception that posts without the label).

### Sub-Agents NEVER:

- Ask questions or request clarification — NEVER use `AskUserQuestion`.
- Run `./do check` (except verification sub-agents).
- Create commits.
- Push or create branches.

## Hardcoded Operational Decisions

| Scenario | Behavior |
|----------|----------|
| `GITLAB_TOKEN` not set | Log to stdout, exit. (Webhook layer should never have invoked us without it.) |
| Adding `progress` label fails | Log it, continue with implementation. |
| `./do check` fails with code errors | Launch fix sub-agent. Retry verification. Repeat up to 3 cycles, then post failure comment + `failed` label, stop. |
| `./do check` fails because a tool is missing | Sub-agent installs it via passwordless sudo and re-runs `./do check`. Only escalate if install fails. |
| `./do check` fails with infrastructure errors (network, non-installable) | Post failure comment + `failed` label, stop. Do NOT retry. |
| `git push` fails | Post failure comment + `failed` label, stop. Local commits remain in the worktree. |
| MR creation fails | Post failure comment + `failed` label, stop. Branch is pushed; a human can create the MR manually. |
| `./do test-deploy` fails | Post `deploy-failed` warning comment (no `failed` label). Finish normally — MR exists, code is pushed. |
| Ambiguous or unclear plan | Implement the most literal, straightforward interpretation. Do NOT ask for clarification. |
