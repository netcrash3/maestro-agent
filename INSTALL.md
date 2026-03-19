# Maestro Agent — Installation Guide

Maestro is a multi-agent feature development pipeline for [Claude Code](https://docs.anthropic.com/en/docs/claude-code). It orchestrates 8 specialized agents that take a feature request from architecture through implementation, review, testing, and documentation — all within your existing project.

## Prerequisites

- [Claude Code CLI](https://docs.anthropic.com/en/docs/claude-code) installed and authenticated
- An existing project directory where you want to add the pipeline

## Installation

### 1. Copy the agent files

From the root of this repository, copy the `.claude/agents/` directory and the `maestro-rules` subdirectory into your target project:

```bash
# From your target project root:
mkdir -p .claude/agents/maestro-rules

# Copy all agent definitions
cp /path/to/maestro_agent/.claude/agents/*.md .claude/agents/

# Copy the coding rules file
cp /path/to/maestro_agent/.claude/agents/maestro-rules/rules.md .claude/agents/maestro-rules/rules.md
```

This installs the following agents:

| Agent | Role |
|-------|------|
| `maestro` | Pipeline conductor — orchestrates all 8 stages |
| `solution-architect` | Analyzes the feature and produces architectural decisions |
| `execution-planner` | Produces a step-by-step implementation plan |
| `backend-engineer` | Implements backend code, APIs, and data models |
| `frontend-engineer` | Implements frontend components and UI |
| `code-reviewer` | Reviews all code (read-only — no source changes) |
| `code-review-cleanup` | Applies review feedback and fixes |
| `feature-test-engineer` | Writes and runs tests for the new feature |
| `doc-cleanup-organizer` | Archives all outputs and cleans up agent directories |
| `dev-build-launcher` | (Optional) Builds and launches the project locally for manual verification |
| `deployment-config-agent` | (Optional) Configures CI/CD pipelines for GitHub Actions, Vercel, or CircleCI |

### 2. Configure permissions

Create or merge the following into `.claude/settings.local.json` in your target project:

```json
{
  "permissions": {
    "allow": [
      "Bash(mkdir:*)",
      "Bash(cp:*)",
      "Bash(rm:*)",
      "Bash(node:*)",
      "Bash(ls:*)",
      "Bash(pwd:*)",
      "Bash(find:*)",
      "Bash(date:*)",
      "Bash(cat:*)",
      "Bash(cd:*)",
      "Bash(echo:*)",
      "Bash(grep:*)",
      "Bash(curl:*)",
      "Bash(lsof:*)",
      "Bash(npm run:*)",
      "Bash(npm ls:*)",
      "Bash(npm install:*)",
      "Bash(npm test:*)",
      "Bash(npx tsc:*)",
      "Bash(npx next:*)",
      "Bash(npx jest:*)",
      "Bash(npx create-next-app@latest:*)",
      "Bash(docker:*)",
      "mcp__ide__getDiagnostics"
    ]
  }
}
```

> **Note:** If you already have a `settings.local.json`, merge the `allow` entries into your existing permissions array. Do not overwrite your existing file.
>
> **Customize for your stack:** The permissions above are tuned for a Node.js / Next.js project. Adjust for your toolchain — for example, replace `npm` entries with `yarn`, `pnpm`, or `bun` equivalents, and add any build/test commands specific to your project (e.g., `Bash(python:*)`, `Bash(cargo:*)`, `Bash(go:*)`).

### 3. Customize the coding rules

Edit `.claude/agents/maestro-rules/rules.md` to match your project's conventions. The default rules cover JavaScript/TypeScript frontend and backend standards. Replace or extend them for your language, framework, and team style.

The rules file is read by `backend-engineer`, `frontend-engineer`, `code-reviewer`, `code-review-cleanup`, `feature-test-engineer`, and `dev-build-launcher` — so any conventions you define here are enforced across the entire pipeline.

### 4. (Optional) Copy the workflow reference

```bash
cp /path/to/maestro_agent/AGENT_WORKFLOW.md ./AGENT_WORKFLOW.md
```

This documents the full pipeline flow, context chain, and output directory structure. Useful as a team reference but not required for the agents to function.

## Directory structure after installation

```
your-project/
├── .claude/
│   ├── settings.local.json          ← permissions
│   └── agents/
│       ├── maestro.md               ← pipeline conductor
│       ├── solution-architect.md
│       ├── execution-planner.md
│       ├── backend-engineer.md
│       ├── frontend-engineer.md
│       ├── code-reviewer.md
│       ├── code-review-cleanup.md
│       ├── feature-test-engineer.md
│       ├── doc-cleanup-organizer.md
│       ├── dev-build-launcher.md
│       ├── deployment-config-agent.md
│       └── maestro-rules/
│           └── rules.md             ← your coding standards
├── ... (your existing project files)
```

## Usage

Start Claude Code in your project directory, then invoke the maestro agent with a feature request:

```
> @maestro Add a user profile page with avatar upload and bio editing
```

For new or blank projects, maestro will ask clarifying questions about your preferred frameworks and languages before starting. For existing projects without architecture documentation, it will suggest running the solution-architect first. For ambiguous requests, it will ask targeted questions to define the feature scope.

To automatically build and launch the project locally after the pipeline completes, append `--run local`:

```
> @maestro Add a user profile page with avatar upload and bio editing --run local
```

Maestro will run the full 8-stage pipeline (plus an optional 9th stage with `--run local`):

1. **solution-architect** — architectural analysis
2. **execution-planner** — detailed implementation plan
3. **backend-engineer** — backend implementation
4. **frontend-engineer** — frontend implementation
5. **code-reviewer** — code review (read-only)
6. **code-review-cleanup** — applies review fixes
7. **feature-test-engineer** — writes and runs tests
8. **doc-cleanup-organizer** — archives outputs, generates commit message, cleans up
9. **dev-build-launcher** — *(optional, with `--run local`)* builds and launches locally

When complete, all implementation documentation is archived under `maestro-features/<timestamp>/` and agent output directories are cleared.

### Running individual agents

You can also invoke agents individually outside the pipeline:

```
> @dev-build-launcher           # Build and launch locally for manual testing
> @deployment-config-agent      # Configure CI/CD deployment pipelines
> @solution-architect           # Run just the architecture analysis
```

## Pipeline outputs

Each pipeline run produces:

```
maestro-features/<unix_timestamp>/
├── feature_requirements.md      ← execution plan
├── backend_implementation.md    ← backend summary
├── frontend_implementation.md   ← frontend summary
├── api_docs.md                  ← API documentation snapshot
├── test_results.md              ← test report
├── code_review.md               ← review findings
└── commit-message.md            ← generated commit message for engineer review

maestro-api-documentation/
└── api-docs.md                  ← permanent, always-current API reference
```

## Persistent files

These files persist across pipeline runs and accumulate knowledge over time:

- `.claude/agents/solution-architect/outputs/architecture.md` — grows with each feature
- `.claude/agents/skills/skills.md` — living reference of project conventions
- `.claude/agents/maestro-rules/rules.md` — your static coding rules (manually maintained)
- `maestro-api-documentation/api-docs.md` — always-current API documentation

## Troubleshooting

**Agent can't find files / Write tool fails silently**
The Write and Read tools require absolute paths. All agents are configured to run `pwd` first and use the result as a path prefix. If an agent skips this step, it may write to the wrong location.

**Permission denied on Bash commands**
Add the missing command pattern to `.claude/settings.local.json` under `permissions.allow`. The format is `Bash(<command>:*)`.

**Pipeline stops mid-run**
Maestro retries failed stages up to 2 times. If a stage fails after retries, it halts and reports which stage failed and why. Fix the underlying issue and re-run `@maestro` with the same prompt.

**Stray .md files in project root**
The `doc-cleanup-organizer` agent scans for `.md` files that other agents may have accidentally written to the project root and archives them. It only cleans up files created during the current run — permanent documentation files like `README.md`, `AGENT_WORKFLOW.md`, and `CLAUDE.md` are preserved.
