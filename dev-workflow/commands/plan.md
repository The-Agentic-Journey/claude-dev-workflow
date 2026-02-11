# Plan Command

You are creating a feature plan. Plans document what will be built, how it will be implemented phase by phase, and what DDD/ADR work is needed. The plan is a document — you do NOT execute any changes.

## Input

The user describes what they want to build. This may be a feature request, a bug fix, a refactor, or any other change.

If the user provides a file path, read it for context but still go through the full planning process.

## Process

### 1. Understand the Feature

Ask clarifying questions about scope, requirements, and constraints. Dig into what the user really needs. Don't assume — ask.

- What problem does this solve?
- Who is affected?
- What are the boundaries of this change?
- Are there constraints (performance, compatibility, etc.)?

Keep asking until you have a clear picture. It's fine to ask multiple rounds of questions.

### 2. Explore the Codebase

Read and understand the existing code before planning changes:

1. Read `CLAUDE.md` for project orientation
2. Read `product/DDD/glossary.md` for domain vocabulary
3. Read `product/DDD/context-map.md` for bounded context overview
4. Read relevant bounded context docs in `product/DDD/contexts/`
5. Read existing ADRs in `product/ADR/` that relate to the feature
6. Read the source files that will be affected
7. Check `product/plans/done/` for similar past work

### 3. Make Technical Decisions with the User

For every significant decision the plan involves, present it to the user for approval. Don't decide alone — propose options and explain trade-offs:

- "For X, I'd suggest approach A because [reason]. We could also do B [reason] or C [reason]. What do you think?"
- Architecture choices, library choices, data model decisions, API design — all should be presented
- The user is the overall authority. Every decision should be explicitly approved before it goes into the plan.

Continue until all significant decisions are resolved.

### 4. Identify DDD Implications

If the feature introduces new domain concepts, the plan should include phases for:

- **Glossary updates** — New terms, updated definitions
- **New bounded context docs** — If a new context is needed, create `product/DDD/contexts/{name}.md`
- **Context map updates** — New contexts, new relationships, updated diagrams

These are documented as phases in the plan. They are NOT executed now.

### 5. Identify ADR Implications

If the feature involves an architectural decision (one that was discussed and decided with the user in step 3), the plan should include a phase for creating an ADR in `product/ADR/`.

The ADR captures the decision, rationale, and consequences. It is documented as a phase in the plan. It is NOT created now.

### 6. Determine Next Plan Number

Scan both directories for the highest existing plan number:
- `product/plans/todo/`
- `product/plans/done/`

The new plan gets the next number (e.g., if highest is 004, new plan is 005).

### 7. Write the Plan

Create the plan file at `product/plans/todo/XXX-PLAN-FEATURE-NAME.md`.

Reference the plan template at `templates/plan.md.template` for the expected format. The plan must include:

- **Overview** — What and why
- **Phases** — Each with: goal, changes table (file, action, details), verification steps
- **DDD phases** — Glossary updates, context docs, context map updates (as applicable)
- **ADR phases** — Architecture decision records (as applicable)
- **Code phases** — Actual implementation work
- **Files Summary** — Complete table of all files touched
- **End-to-End Verification** — Including `./do check`
- **Update Considerations** — Config, storage, dependencies, migration, backwards compatibility

Order phases logically. DDD/ADR phases typically come before or alongside the code phases they document.

The plan should reflect all the decisions made with the user in step 3.

### 8. Final Review with User

Present the complete plan to the user. Iterate until they approve:

- Walk through each phase briefly
- Highlight key decisions and their rationale
- Ask if anything needs to change

The plan is ready when the user approves it.

## Critical Rules

1. **The `/plan` command writes the plan document ONLY.** It does NOT execute any changes — no code, no DDD edits, no ADR creation. All of that happens when `/implement` executes the plan.

2. **Every significant decision goes through the user.** Don't make architectural choices unilaterally. Present options, explain trade-offs, get approval.

3. **DDD and ADR work are plan phases, not separate commands.** If a feature needs glossary updates or a new ADR, those appear as phases within the plan alongside the code changes.

4. **Plans reference the template format.** See `templates/plan.md.template` for the expected structure.

5. **Plans go in `product/plans/todo/`.** After implementation via `/implement`, they are moved to `product/plans/done/`.
