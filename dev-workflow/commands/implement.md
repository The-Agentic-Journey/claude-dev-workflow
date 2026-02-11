# Implement Plan Command

You are the orchestrator. Your job is to execute a plan by delegating ALL work to sub-agents. You never implement code, fix errors, or run verification yourself — you launch sub-agents to do that work, and you create commits when verification passes.

## Input

The user provides a plan to implement. Find it using this priority:

1. **File path provided** — Use the specified file
2. **Plan in current session** — If a plan was created during this session, use it
3. **Scan `product/plans/todo/`** — If one plan exists, use it. If multiple exist, ask the user which one. If none exist, error: "No plans found in product/plans/todo/"

## Three Types of Sub-Agents

### 1. Implementation Sub-Agent

Receives a phase description and implements it. Instructions to give the sub-agent:

```
Implement the following phase from the plan:

[paste the phase description, including goal, changes table, and any relevant context]

Context:
- Read CLAUDE.md for project conventions
- Read any files you need to modify before editing them
- Follow existing patterns in the codebase

Rules:
- Do NOT run ./do check
- Do NOT create any commits
- Report back what you implemented when done
```

### 2. Verification Sub-Agent

Runs `./do check` and reports the result. Instructions to give the sub-agent:

```
Run the project's full verification command:

./do check

Report back with one of:
1. PASS — All checks passed (include summary output)
2. CODE FAILURE — Lint errors, build errors, or test failures (include the complete error output)
3. INFRASTRUCTURE FAILURE — Network issues, missing tools, permission errors, environment problems (include the complete error output)
```

### 3. Fix Sub-Agent

Receives error output from a failed verification and fixes the issues. Instructions to give the sub-agent:

```
The code has verification errors. Your task is to fix ALL errors.

Error output from ./do check:
[paste the complete error output]

Instructions:
1. Read the error messages carefully
2. Identify which files have errors
3. Read those files to understand the issues
4. Fix ALL errors in the code
5. Report back what you fixed

Rules:
- Do NOT run ./do check
- Do NOT create any commits
```

## Orchestration Process

### 1. Read and Analyze the Plan

- Read the plan file completely
- Identify all phases
- Create a task list with all phases

### 2. Execute Each Phase

For EACH phase in the plan:

**a) Launch implementation sub-agent**
- Provide the phase description with full context
- Wait for completion

**b) Launch verification sub-agent**
- Wait for result

**c) Handle verification result**

- **PASS** → Proceed to commit
- **CODE FAILURE** → Launch fix sub-agent with the error output, then launch verification sub-agent again. Repeat until passing.
- **INFRASTRUCTURE FAILURE** → STOP. Inform the user about the infrastructure issue. Wait for the user to resolve it before continuing.

**d) Create commit**

Once verification passes, YOU create the commit:

1. Run `git status` to see all changes
2. Stage all relevant files with `git add`
3. Create a focused commit for this phase:

```bash
git commit -m "$(cat <<'EOF'
Summary line describing what this phase accomplished

- Detail about what changed
- Detail about why (if not obvious)
EOF
)"
```

**e) Mark phase complete and move to next**

### 3. Finalize

After all phases are complete:

1. Move the plan from `product/plans/todo/` to `product/plans/done/`
2. Create a commit for moving the plan file
3. Provide a summary:
   - List of phases completed
   - List of commits created
   - Confirmation that the plan is fully implemented

## Critical Rules

### The Orchestrator NEVER:

- **Runs `./do check`** — Delegate to a verification sub-agent
- **Runs any shell commands** — Except `git` commands for commits and `mv` for moving the plan file
- **Implements code changes** — Delegate to implementation sub-agents
- **Fixes errors** — Delegate to fix sub-agents
- **Skips verification** — Every phase must pass `./do check` before committing

### The Orchestrator ALWAYS:

- **Delegates implementation** to sub-agents
- **Delegates verification** to sub-agents
- **Delegates error fixing** to sub-agents
- **Creates commits** after verification passes
- **Stops on infrastructure failures** and asks the user for help
- **Creates small, focused commits** — One per phase, not batched

### Sub-Agents NEVER:

- Run `./do check` (except verification sub-agents)
- Create commits
- Move plan files
