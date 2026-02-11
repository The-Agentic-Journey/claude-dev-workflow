# Bootstrap Command

You are scaffolding the development methodology for a new project. This is a conversational process — you're helping someone set up their project with structured planning, DDD documentation, and a verification workflow.

## Process

### 1. Conversational Start

Ask the user about their project:

"What do you want to build? Just explain the idea to me."

Listen and ask follow-up questions to understand:
- What the project does
- Who it's for
- What problem it solves

Get enough to write a meaningful project overview (2-3 paragraphs). This will go in CLAUDE.md and is the primary orientation for any Claude instance working on the project.

### 2. Tech Stack

Ask: "Do you already have a tech stack in mind, or would you like me to suggest one?"

- **User has a stack** → Use it
- **User wants suggestions** → Recommend a stack based on the project description. Explain your reasoning. Let the user approve or modify.

### 3. Create Directory Structure

Create the following files and directories:

```
product/
├── DDD/
│   ├── glossary.md            ← from templates/glossary.md.template
│   ├── context-map.md         ← from templates/context-map.md.template
│   └── contexts/
│       └── .gitkeep
├── ADR/
│   └── .gitkeep
└── plans/
    ├── todo/
    │   └── .gitkeep
    └── done/
        └── .gitkeep
```

For the glossary and context map, use the templates as a starting point. Keep the placeholder structure — the user will fill these in as the project develops, or `/plan` will include DDD updates as phases in feature plans.

Empty directories get a `.gitkeep` file so they are tracked in git.

### 4. Generate CLAUDE.md

Create `CLAUDE.md` in the project root using the template at `templates/CLAUDE.md.template`. Fill in:

- **{{PROJECT_NAME}}** — From the conversation
- **{{PROJECT_DESCRIPTION}}** — 2-3 paragraphs based on what the user described. This should be substantial — it's the main orientation for any Claude instance. Include the vision, what it aims to achieve, who it's for.
- **{{TECH_STACK}}** — From step 2
- **{{SOURCE_TREE}}** — Leave as a placeholder comment: `# TODO: Update as source files are created`

All other sections (Documentation Structure, Before Starting Work, Development Commands, Planning and Implementation, Pre-change Questions) come from the template as-is.

### 5. Create Starter `./do` Script

Create `./do` in the project root from the template at `templates/do.template`. All command bodies are placeholders — the user fills them in as the project develops.

Make it executable: `chmod +x ./do`

### 6. Summary

Report what was created:

```
Created:
- CLAUDE.md — Project guide for Claude instances
- ./do — Development task runner (commands need implementation)
- product/DDD/glossary.md — Domain glossary (starter)
- product/DDD/context-map.md — Bounded context map (starter)
- product/DDD/contexts/.gitkeep
- product/ADR/.gitkeep
- product/plans/todo/.gitkeep
- product/plans/done/.gitkeep
```

Then explain the workflow:

"Your project is set up. Here's how to use it:

1. **`/plan`** — Create a feature plan. Describe what you want to build, and I'll create a phased plan with DDD docs, ADRs, and implementation steps.
2. **`/implement`** — Execute a plan. I'll work through each phase, verify with `./do check`, and commit.
3. **`./do`** — Your task runner. Fill in the command bodies (build, lint, test, run, check) as your project develops. `./do check` is the mandatory pre-commit verification.

Start by describing your first feature and I'll help you plan it."

## Rules

1. **This command is for new, empty projects.** Don't run it on an existing project with established structure.
2. **Be conversational.** Ask questions, understand the project, help the user think through their idea.
3. **Don't over-scaffold.** Create the structure and starters, not the implementation. The project grows through `/plan` and `/implement`.
4. **Empty directories get `.gitkeep`.** So they can be tracked in git.
5. **Templates are the source of truth.** Reference the template files in `templates/` for the expected format of each generated file.
