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

### 3. Identify Packages

Every project is a monorepo. Ask: "What packages will this project have? For example: a backend API, a frontend app, a shared library?"

Help the user identify the initial set of packages. Each package is a top-level directory in the monorepo. Examples:
- `server/` — Backend API
- `client/` — Frontend app
- `shared/` — Shared types/utilities
- `infra/` — Infrastructure/deployment

It's fine to start with one package. More can be added later via `/dev:plan`.

### 4. Create Directory Structure

Create the following files and directories:

```
{{package_a}}/              ← empty package directories from step 3
{{package_b}}/
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

For the glossary and context map, use the templates as a starting point. The glossary template includes starter methodology terms (Monorepo, `./do` Script, Package) — keep these and add project-specific domain terms as the project develops.

Create a `.gitkeep` in each empty package directory so they are tracked in git. Empty directories get a `.gitkeep` file.

### 5. Generate CLAUDE.md

Create `CLAUDE.md` in the project root using the template at `templates/CLAUDE.md.template`. Fill in:

- **{{PROJECT_NAME}}** — From the conversation
- **{{PROJECT_DESCRIPTION}}** — 2-3 paragraphs based on what the user described. This should be substantial — it's the main orientation for any Claude instance. Include the vision, what it aims to achieve, who it's for.
- **{{TECH_STACK}}** — From step 2
- **{{PACKAGE_NAME}}, {{PACKAGE_PATH}}, {{PACKAGE_PURPOSE}}** — Fill in the Packages table from step 3
- **{{SOURCE_TREE}}** — Leave as a placeholder comment: `# TODO: Update as source files are created`

All other sections (Documentation Structure, Before Starting Work, The `./do` Script, Planning and Implementation, Pre-change Questions) come from the template as-is.

### 6. Create Starter `./do` Script

Create `./do` in the project root from the template at `templates/do.template`. All command bodies are placeholders — `/dev:plan` will fill these in as part of the first feature plan.

Make it executable: `chmod +x ./do`

### 7. Initialize Git Repository

Initialize a git repository and create the initial commit:

1. Run `git init`
2. Stage all created files with `git add`
3. Create an initial commit with a message like: `Bootstrap project with dev-workflow methodology`

### 8. Summary

Report what was created:

```
Created:
- CLAUDE.md — Project guide for Claude instances
- ./do — Development task runner
- {{package_a}}/ — Package directory
- {{package_b}}/ — Package directory
- product/DDD/glossary.md — Domain glossary (with methodology terms)
- product/DDD/context-map.md — Bounded context map (starter)
- product/DDD/contexts/.gitkeep
- product/ADR/.gitkeep
- product/plans/todo/.gitkeep
- product/plans/done/.gitkeep
- Initialized git repository with initial commit
```

Then explain the workflow:

"Your project is set up as a monorepo. Here's how to use it:

1. **`/dev:plan`** — Create a feature plan. Describe what you want to build, and I'll create a phased plan with acceptance tests, DDD docs, ADRs, and implementation steps.
2. **`/dev:implement`** — Execute a plan. I'll work through each phase, verify with `./do check`, and commit.

Start by describing your first feature and I'll help you plan it."

## Rules

1. **This command is for new, empty projects.** Don't run it on an existing project with established structure.
2. **Every project is a monorepo.** All code lives in one repository. Identify packages early (step 3) and create their directories.
3. **Be conversational.** Ask questions, understand the project, help the user think through their idea.
4. **Don't over-scaffold.** Create the structure and starters, not the implementation. The project grows through `/dev:plan` and `/dev:implement`.
5. **Empty directories get `.gitkeep`.** So they can be tracked in git.
6. **Templates are the source of truth.** Reference the template files in `templates/` for the expected format of each generated file.
7. **Always create a git repo and initial commit.** The project should be version-controlled from the start.
