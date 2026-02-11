---
name: plan-before-code
description: Suggests /plan before any implementation work begins. Use when the user requests features, bug fixes, refactors, or any code changes.
---

# Plan Before Code Skill

## Purpose

This skill ensures that planning is always suggested before implementation work begins. The user decides whether to plan — the agent's job is to always offer.

## Rules

### 1. Always Suggest `/plan` First

For any work request — features, bug fixes, refactors, improvements, anything that involves changing code — suggest using `/plan` before starting implementation.

Examples:
- "Before I start implementing this, would you like me to create a plan with `/plan`? It'll map out the changes, identify DDD and ADR implications, and break the work into phases."
- "I can jump straight into this, but would you prefer a plan first? `/plan` will help us think through the approach."

### 2. The User Decides

The user is the authority on whether planning is needed. If they say:
- "Just do it" / "No plan needed" / "Go ahead" → Proceed without a plan
- "Yes" / "Plan it" / "Sure" → Run `/plan`

Never argue with the user about whether planning is necessary. Suggest it, respect the answer, move on.

### 3. Where Plans Live

Plans are created in `product/plans/todo/` and follow the naming convention `XXX-PLAN-FEATURE-NAME.md`. After implementation, they are moved to `product/plans/done/`.

### 4. What Planning Includes

A `/plan` creates a comprehensive plan that covers:
- Phased implementation steps
- DDD implications (glossary updates, new bounded contexts, context map changes)
- ADR implications (architectural decisions that should be recorded)
- File-by-file change details
- Verification steps per phase
- Update/backwards compatibility considerations
