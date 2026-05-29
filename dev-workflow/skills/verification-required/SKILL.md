---
name: verification-required
description: Enforces that ./do check must pass before any commit. Use whenever creating commits or finishing implementation work.
---

# Verification Required Skill

## Purpose

This skill enforces that `./do check` MUST pass before any commits are made. There are NO exceptions.

## Rules

### 1. `./do check` is MANDATORY

Before creating ANY commit, you MUST run `./do check` and it MUST pass completely. This is non-negotiable.

Always capture output to a unique log file using `tee`:

```bash
LOG_FILE="/tmp/do-check-$(date +%s)-$$.log"
./do check 2>&1 | tee "$LOG_FILE"
```

The log file is the source of truth. If verification fails, read the log file to diagnose errors — never re-run `./do check` just to see the output again.

### 2. Missing Tools and Native-Build Failures are NOT a Stop — Provision and Retry

A missing tool, a missing compiler/`make`, a node-gyp / native-build failure, or a missing apt/pip/npm package is **not** an infrastructure failure and **not** a code failure — it is a provisioning gap you are expected to close. If you have sudo (e.g. a disposable sandbox), install it and re-run:

```bash
sudo apt-get update && sudo apt-get install -y build-essential <pkg>
./do check 2>&1 | tee "/tmp/do-check-$(date +%s)-$$.log"
```

Only after the toolchain is present do you classify what remains:

- If a dependency *still* won't build or run, that is a **code failure** — fix it (e.g. bump the dependency to a version compatible with the current runtime). See Rule 4.
- Treat it as infrastructure (Rule 3) ONLY if a provisioning command you actually ran fails, or the tool genuinely cannot be installed.

In an interactive session without sudo, ask the user to install the missing tool rather than declaring a full stop.

### 3. Genuine Infrastructure Failures are a FULL STOP

If `./do check` fails due to a genuine infrastructure issue — a network/DNS outage, an SSH/permission error, or a permission error on a mounted volume — this is a **FULL STOP**. (A missing tool is NOT this — see Rule 2.) You CANNOT:

- Proceed with commits
- Skip `./do check` and use `./do lint` or `./do build` as a substitute
- Assume the code is correct because lint/build passed
- Work around the infrastructure issue

Instead, you MUST:

1. Stop all implementation work immediately
2. Inform the user that `./do check` failed due to infrastructure
3. Explain specifically what infrastructure issue occurred
4. Ask the user to fix the infrastructure issue or provide guidance
5. Wait for the user to confirm the infrastructure is fixed before continuing

### 4. Code Failures Must Be Fixed

If `./do check` fails due to code issues (lint errors, build errors, test failures), you MUST:

1. Fix all errors (or delegate to sub-agents to fix them)
2. Run `./do check` again
3. Repeat until `./do check` passes completely
4. Only then create commits

### 5. No Partial Verification

The following are NOT acceptable substitutes for `./do check`:

- `./do lint` alone
- `./do build` alone
- `./do test` alone
- Any combination that doesn't include the full `./do check` pipeline
- Any other project-specific commands that only cover part of the verification

`./do check` is the single source of truth for whether code is ready to commit.

## Rationale

The `./do check` command runs the full verification pipeline for the project. Partial checks are insufficient because:

1. Integration tests catch issues that unit tests miss
2. The full pipeline validates the complete build and deployment flow
3. Skipping `./do check` means shipping unverified code
