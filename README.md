# Maestro Agent

A multi-agent feature development pipeline for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) that takes a single feature request and drives it through architecture, planning, implementation, code review, testing, and documentation — automatically.

## The Problem

Building a feature with an AI coding assistant is often a back-and-forth loop. You prompt for code, review it yourself, ask for fixes, write tests manually, and hope nothing was missed. The AI has no structured process — it just generates code and moves on. There's no architecture phase, no code review, no test validation, and no documentation trail. The quality of the output depends entirely on how much you babysit it.

## What Maestro Does

Maestro replaces that ad-hoc loop with a semi-deterministic, 8-stage pipeline. You describe a feature once. Maestro runs it through a sequence of specialized agents — each with a defined role, constrained scope, and verified output — then hands you back working code with tests, a code review, and archived documentation.

No stage is skipped. No agent moves forward until the previous one's output files are verified on disk. The pipeline either completes fully or stops and tells you exactly where it failed.

That's not to say that you shouldn't still take a look at the code that you plan to move to production and ensure that maestro has in fact produced code that lives up to your coding standards.

## The Pipeline

```
You: "Add user profile pages with avatar upload"
 │
 ▼
[1] Solution Architect    ─  Analyzes the request against your codebase.
 │                            Produces architectural decisions, technology
 │                            choices, and integration points.
 ▼
[2] Execution Planner     ─  Converts the architecture into a numbered,
 │                            step-by-step implementation plan with
 │                            dependencies and task ordering.
 ▼
[3] Backend Engineer      ─  Writes backend code: APIs, data models,
 │                            business logic, migrations. Produces full
 │                            API documentation for the frontend to consume.
 ▼
[4] Frontend Engineer     ─  Writes frontend code: components, pages,
 │                            routing, state management. Integrates only
 │                            with documented API endpoints — never guesses.
 ▼
[5] Code Reviewer         ─  Reviews all implemented code for security,
 │                            architecture compliance, and quality.
 │                            Read-only — does not touch source files.
 ▼
[6] Code Review Cleanup   ─  Applies every fix from the review. Verifies
 │                            compilation after changes.
 ▼
[7] Feature Test Engineer ─  Writes unit and integration tests. Runs them.
 │                            Reports coverage and results.
 ▼
[8] Doc Cleanup Organizer ─  Archives all pipeline outputs into
                              maestro-features/<timestamp>/.
                              Cleans up all intermediate agent files.
```

Each agent reads the outputs of every agent before it. Context flows forward through the entire pipeline — the frontend engineer sees the backend's API docs, the code reviewer sees the architecture decisions and the execution plan, and the test engineer sees everything.

## Why This Is Useful

**Consistent quality regardless of feature complexity.** Every feature — whether it's a one-endpoint CRUD or a multi-service integration — goes through the same architecture review, implementation, code review, and testing cycle.

**Code review is built in, not bolted on.** The code reviewer agent checks for security vulnerabilities, architectural violations, and adherence to your project's coding standards (defined in a rules file you control). A separate cleanup agent applies the fixes. This separation ensures the review is honest — the reviewer can't just silently fix things and skip reporting them.

**Your conventions are enforced automatically.** The `maestro-rules/rules.md` file defines your coding standards — naming conventions, patterns, architectural boundaries. Every agent that writes or reviews code reads this file. Your rules are applied consistently across backend, frontend, review, and testing.

**Full documentation trail for every feature.** Each pipeline run produces a timestamped archive with the execution plan, implementation summaries, API docs, test results, and code review findings. Six months from now, you can look at `maestro-features/1709912345/` and understand exactly what was built, why, and what was reviewed.

**API documentation stays current.** The backend engineer produces API docs every run. The doc cleanup organizer copies them to a permanent location. Your API documentation is always up to date — not because someone remembered to update it, but because the pipeline requires it.

**Separation of concerns between agents.** Each agent has a constrained scope. The backend engineer doesn't touch frontend files. The code reviewer doesn't modify source code. The test engineer doesn't change application code to make tests pass. These boundaries prevent the kind of scope creep and silent modifications that happen when a single AI session tries to do everything at once.

**Persistent knowledge across runs.** The solution architect's analysis and the skills/conventions file persist across pipeline runs. The second feature you build benefits from the architectural context established during the first. The pipeline gets smarter about your project over time.

**Agent memory that grows with your project.** Six agents — solution architect, backend engineer, frontend engineer, code reviewer, code review cleanup, and dev-build-launcher — maintain local memory across executions. At the end of each run, they review what they discovered (code paths, patterns, library locations, architectural decisions) and persist anything valuable to `.claude/agent-memory-local/<agent-name>/`. Future runs start with that context already loaded, so agents don't re-learn the same things. Memory directories are created automatically on first use.

## What You Get After a Run

```
maestro-features/<timestamp>/
├── feature_requirements.md      # The execution plan
├── backend_implementation.md    # What was built on the backend
├── frontend_implementation.md   # What was built on the frontend
├── api_docs.md                  # API documentation snapshot
├── test_results.md              # Test coverage and results
└── code_review.md               # Review findings and resolutions

maestro-api-documentation/
└── api-docs.md                  # Always-current API reference
```

## Additional Agents

Two agents operate outside the pipeline and can be invoked independently:

- **`dev-build-launcher`** — Discovers your tech stack, installs dependencies, builds the project, launches all services, and enters a monitoring loop that auto-repairs compilation errors and runtime crashes as they occur. Use it after a pipeline run to verify everything works end-to-end.

- **`deployment-config-agent`** — Configures deployment CI/CD pipelines for GitHub Actions, Vercel, or CircleCI. Verifies all secrets are provided, writes environment configuration files and build scripts (Dockerfiles, Terraform, etc.), and ensures the pipeline runs the full test suite — including spinning up containers for integration tests — before building the final artifact. This agent writes configuration only and never executes deployments; the pipeline itself handles that.

## Getting Started

See [INSTALL.md](INSTALL.md) for step-by-step installation instructions, including how to copy the agents into your project, configure permissions, and customize coding rules for your stack.

## Requirements

- [Claude Code CLI](https://docs.anthropic.com/en/docs/claude-code) installed and authenticated
- An existing project (or a new one — the pipeline works on greenfield projects too)
