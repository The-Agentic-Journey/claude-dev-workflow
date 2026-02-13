---
name: no-raw-commands
description: Enforces that all project commands go through ./do. Activates when about to run npm, yarn, pnpm, npx, docker, docker compose, pip, cargo, make, mvn, gradle, poetry, pipenv, bun, deno, or any build/test/run/lint/install command directly. These must always go through ./do run, ./do build, ./do test, ./do lint, ./do check instead.
---

# No Raw Commands

## Purpose

The `./do` script is the **only** interface for running project commands. Never bypass it by running package managers, container tools, or build systems directly.

## Rule

When you need to run a project-related command, ALWAYS use `./do`:

| Instead of | Use |
|------------|-----|
| `npm start`, `yarn start`, `pnpm start` | `./do run` |
| `npm run build`, `yarn build` | `./do build` |
| `npm test`, `yarn test`, `pytest` | `./do test` |
| `npm run lint`, `eslint .` | `./do lint` |
| `npm run check`, any verification | `./do check` |
| `docker compose up`, `docker run` | `./do run` |
| `npm install`, `yarn add`, `pip install` | Not your job — `./do` handles dependency installation automatically |
| `npx`, `bunx`, `pnpm dlx` | Not directly — if a tool is needed, it should be wired into `./do` |
| `make`, `cargo build`, `mvn package`, `gradle build` | `./do build` |

## Why

1. **`./do` is self-contained.** It handles dependency installation, environment setup, and configuration. Running `npm install` directly may miss setup steps that `./do` performs.
2. **`./do` knows the runtime environment.** It configures PORT/HOST variables, allowed hosts, CORS, and other environment-specific settings. Running `docker compose up` directly may start services with wrong configuration.
3. **`./do` is the source of truth.** If `./do run` doesn't work, that's a bug in `./do` that needs fixing — not a reason to bypass it.
4. **Consistency across agents.** Every sub-agent (implementation, verification, fix) uses the same interface. No agent should have knowledge of the underlying toolchain beyond what `./do` exposes.

## When This Does NOT Apply

This rule applies to **project commands** only. You may still use:
- `git` commands directly (commits, branches, push)
- `curl` for API calls (e.g., creating PRs)
- `ls`, `cat`, `mkdir`, and other filesystem operations
- `chmod` for file permissions
- Any tool that is not a project build/test/run/lint/install command

## What To Do If `./do` Doesn't Support What You Need

If you need to run something that `./do` doesn't have a command for, **do not run it directly**. Instead:
1. Note that `./do` needs a new subcommand or update
2. This should be addressed as a `./do` script change in the current plan, or flagged for the user
