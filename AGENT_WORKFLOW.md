# Maestro Agent Workflow

This document describes how the multi-agent pipeline is orchestrated, how context flows between agents, and what each agent produces and consumes.

---

## Overview

The **maestro** agent is the conductor. It accepts a feature request from the user and drives a strict 8-stage sequential pipeline, with an optional 9th stage. No mandatory stage is skippable. No parallelism. Each stage's outputs become the next stage's inputs.

Before maestro is invoked, Claude Code (the caller) assesses the project state and asks the user clarifying questions when needed — such as framework/language preferences for new projects, missing architecture documentation, new project type expansions, or ambiguous feature requests. All gathered context is included in the prompt passed to maestro, which forwards it to the solution-architect.

Two additional agents — `dev-build-launcher` and `deployment-config-agent` — can be run independently at the developer's discretion. The `dev-build-launcher` can also be triggered automatically at the end of a pipeline run by appending `--run local` to the maestro prompt.

---

## Pipeline Stages

```
User Prompt
    │
    ▼
[1] solution-architect      → architecture.md, skills.md
    │
    ▼
[2] execution-planner       → execution_plan.md
    │
    ▼
[3] backend-engineer        → api-docs.md, backend new_feature_implementation.md
    │
    ▼
[4] frontend-engineer       → frontend new_feature_implementation.md
    │
    ▼
[5] code-reviewer           → code_review_suggestions.md  (READ-ONLY — no source changes)
    │
    ▼
[6] code-review-cleanup     → applies fixes, updates api-docs.md + both implementation files
    │
    ▼
[7] feature-test-engineer   → test files, test_results.md
    │
    ▼
[8] doc-cleanup-organizer   → maestro-features/<timestamp>/, maestro-api-documentation/api-docs.md
                               commit-message.md, clears all agent output directories
    │
    ▼
[9] dev-build-launcher     → (OPTIONAL — only when --run local is specified)
                               builds and launches the project locally
```

---

## Agent-by-Agent Details

### maestro
**Model:** sonnet
**Role:** Pipeline conductor. Does not implement any work itself — only delegates and verifies.

**Responsibilities:**
- Receives the user's feature request along with all pre-gathered context (technology choices, clarification answers) from the caller
- Detects the `--run local` flag and strips it from the feature request before passing to sub-agents
- Invokes each of the 8 mandatory sub-agents in order, plus the optional 9th stage if `--run local` was specified
- After each stage, reads the required output files from disk to confirm success (verbal confirmation is not sufficient)
- Retries failed stages up to 2 times before halting
- Displays a running pipeline status board to the user
- Reports the final completion summary only after all stages complete

**Pre-pipeline context gathering (handled by the caller, NOT by maestro):** Before maestro is invoked, Claude Code (the caller) assesses the project state and asks the user any necessary clarifying questions — framework/language choices for new projects, missing architecture documentation, technology choices for new project type expansions, and targeted questions for ambiguous requirements. All answers are included in the prompt passed to maestro, which forwards them to the solution-architect.

**Context forwarded at each stage:** original feature request + all pre-gathered clarification context + all prior stage outputs + a summary of what is complete and what remains.

**Pipeline enforcement:** Maestro is forbidden from skipping mandatory stages, treating any single agent's output as "done", or returning control to the user before all 8 mandatory stages and file verification are complete. If `--run local` was specified, stage 9 is also mandatory.

---

### Stage 1 — solution-architect
**Model:** opus
**Reads:** entire codebase (on first run); `.claude/agents/solution-architect/outputs/architecture.md` and `.claude/agents/skills/skills.md` (on subsequent runs)
**Writes:**
- `.claude/agents/solution-architect/outputs/architecture.md` — baseline codebase analysis + new feature architectural decisions (technology choices, layer placement, rationale)
- `.claude/agents/skills/skills.md` — living reference of code conventions, patterns, and library decisions

**Key behavior:** On first run, performs a full codebase review before analyzing the feature request. On subsequent runs, reads existing documents and appends new decisions. Never writes source code.

**Context passed forward:** architecture.md and skills.md are referenced by every downstream agent as ground truth for architectural constraints and coding conventions.

---

### Stage 2 — execution-planner
**Model:** sonnet
**Reads:** `.claude/agents/solution-architect/outputs/architecture.md`
**Writes:** `.claude/agents/execution-planner/outputs/execution_plan.md`

**Output structure:** overview, architecture alignment, prerequisites, numbered execution steps (each with task, details, rationale, dependencies, complexity), testing/validation criteria, risks, and out-of-scope items.

**Key behavior:** If `architecture.md` is missing, warns the user and asks whether to proceed. Validates all planned steps against architectural constraints before writing.

**Context passed forward:** `execution_plan.md` is the primary directive for backend-engineer, frontend-engineer, code-reviewer, and feature-test-engineer.

---

### Stage 3 — backend-engineer
**Model:** opus
**Reads:**
- `.claude/agents/execution-planner/outputs/execution_plan.md`
- `.claude/agents/solution-architect/outputs/architecture.md`
- `.claude/agents/skills/skills.md`
- `.claude/agents/maestro-rules/rules.md` (backend section only)

**Writes:**
- Source code files in project directories
- `.claude/agents/backend-engineer/outputs/api-docs.md` — full current API documentation (all endpoints, request/response shapes, auth requirements)
- `.claude/agents/backend-engineer/outputs/implementation/new_feature_implementation.md` — handoff summary for frontend and downstream agents

**Key behavior:** Always invoked, even when no backend tasks exist (must still produce `api-docs.md` reflecting current API state). Asks for credentials before writing code that requires them — never uses placeholders. Verifies compilation before running migrations.

**Context passed forward:** `api-docs.md` is the sole source of truth for frontend API integration. `new_feature_implementation.md` is reviewed by code-reviewer and cleaned up by code-review-cleanup.

---

### Stage 4 — frontend-engineer
**Model:** opus
**Reads:**
- `.claude/agents/execution-planner/outputs/execution_plan.md`
- `.claude/agents/solution-architect/outputs/architecture.md`
- `.claude/agents/skills/skills.md`
- `.claude/agents/maestro-rules/rules.md` (frontend section only)
- `.claude/agents/backend-engineer/outputs/api-docs.md`

**Writes:**
- Source code files (UI components, pages, state, routing, styles)
- `.claude/agents/frontend-engineer/outputs/implementation/new_feature_implementation.md`

**Key behavior:** Only integrates backend endpoints that are documented in `api-docs.md` — never guesses endpoint shapes. Verifies compilation before writing its summary. Stays strictly within frontend boundaries.

**Context passed forward:** `new_feature_implementation.md` is reviewed by code-reviewer and cleaned up by code-review-cleanup.

---

### Stage 5 — code-reviewer
**Model:** sonnet
**Reads:**
- `.claude/agents/solution-architect/outputs/architecture.md`
- `.claude/agents/skills/skills.md`
- `.claude/agents/execution-planner/outputs/execution_plan.md`
- `.claude/agents/maestro-rules/rules.md` (relevant section only)
- Both implementation files from backend-engineer and frontend-engineer
- All source files referenced in the implementation documents

**Writes:** `.claude/agents/code-reviewer/outputs/code_review_suggestions.md`

**Key behavior:** READ-ONLY. Must not modify any source file, config, or documentation. Reviews for: architectural compliance, skills/standards compliance, security (credentials, injection, auth, data exposure), and code quality. Maestro also verifies via `git diff` that no source files were changed by this agent.

**Context passed forward:** `code_review_suggestions.md` is the task list for code-review-cleanup.

---

### Stage 6 — code-review-cleanup
**Model:** sonnet
**Reads:**
- `.claude/agents/code-reviewer/outputs/code_review_suggestions.md`
- `.claude/agents/solution-architect/outputs/architecture.md`
- `.claude/agents/skills/skills.md`
- `.claude/agents/maestro-rules/rules.md` (relevant sections)

**Writes:**
- Source code fixes (backend and frontend)
- Updated `.claude/agents/backend-engineer/outputs/api-docs.md` (if API surface changed)
- Updated `.claude/agents/backend-engineer/outputs/implementation/new_feature_implementation.md`
- Updated `.claude/agents/frontend-engineer/outputs/implementation/new_feature_implementation.md`

**Key behavior:** Works through each review suggestion in dependency order. Verifies compilation after backend and frontend phases. Does not introduce scope creep — only fixes what was identified in the review.

**Context passed forward:** Updated implementation documents reflect the post-cleanup state for the test engineer.

---

### Stage 7 — feature-test-engineer
**Model:** sonnet
**Reads:**
- `.claude/agents/backend-engineer/outputs/implementation/new_feature_implementation.md`
- `.claude/agents/frontend-engineer/outputs/implementation/new_feature_implementation.md`
- `.claude/agents/execution-planner/outputs/execution_plan.md`
- `.claude/agents/maestro-rules/rules.md` (applicable sections)
- Existing test files in the project

**Writes:**
- Test files in project test directories
- `.claude/agents/feature-test-engineer/outputs/test_results.md`

**Key behavior:** Audits existing test coverage first, then writes unit and integration tests for any gaps. Never modifies application code to make tests pass. Only updates existing tests when a change was explicitly planned in the execution plan. Scans all test files for hardcoded secrets before finalizing.

**Context passed forward:** `test_results.md` is archived by doc-cleanup-organizer.

---

### Stage 8 — doc-cleanup-organizer
**Model:** sonnet
**Reads:** all agent output files from stages 1–7
**Writes:**
- `maestro-features/<unix_timestamp>/feature_requirements.md` ← from execution_plan.md
- `maestro-features/<unix_timestamp>/backend_implementation.md` ← from backend new_feature_implementation.md
- `maestro-features/<unix_timestamp>/frontend_implementation.md` ← from frontend new_feature_implementation.md
- `maestro-features/<unix_timestamp>/api_docs.md` ← from api-docs.md
- `maestro-features/<unix_timestamp>/test_results.md` ← from test_results.md
- `maestro-features/<unix_timestamp>/code_review.md` ← from code_review_suggestions.md
- `maestro-features/<unix_timestamp>/commit-message.md` ← generated commit message for engineer review
- `maestro-api-documentation/api-docs.md` ← permanent copy of current API docs

**Also deletes** all agent output files after archiving (except `.claude/agents/skills/` and `.claude/agents/solution-architect/outputs/` which persist across runs).

**Key behavior:** Never fabricates content — only copies files that exist on disk. Generates a structured commit message summarizing all changes (does NOT perform any git operations — the commit message is for the engineer to use after review). Scans for stray `.md` files written to the project root or `features/` root and moves them into the archive. Runs a self-verification checklist confirming all 7 archive files exist and all output directories are empty before reporting completion.

---

## Shared Reference Files (Persistent Across Pipeline Runs)

| File | Written By | Read By |
|------|-----------|---------|
| `.claude/agents/solution-architect/outputs/architecture.md` | solution-architect | execution-planner, backend-engineer, frontend-engineer, code-reviewer, code-review-cleanup |
| `.claude/agents/skills/skills.md` | solution-architect | backend-engineer, frontend-engineer, code-reviewer, code-review-cleanup |
| `.claude/agents/maestro-rules/rules.md` | (static, not written by agents) | backend-engineer, frontend-engineer, code-reviewer, code-review-cleanup, feature-test-engineer, dev-build-launcher |
| `maestro-api-documentation/api-docs.md` | doc-cleanup-organizer | (permanent reference for developers) |

---

## Context Chain Summary

```
User prompt
 └─► solution-architect produces architecture.md + skills.md
      └─► execution-planner reads architecture.md → produces execution_plan.md
           └─► backend-engineer reads execution_plan.md + architecture.md + skills.md
                 → produces api-docs.md + backend implementation summary
                └─► frontend-engineer reads execution_plan.md + architecture.md + skills.md + api-docs.md
                      → produces frontend implementation summary
                     └─► code-reviewer reads all implementation files + architecture.md + skills.md
                           → produces review report (no source changes)
                          └─► code-review-cleanup reads review report + architecture.md + skills.md
                                → fixes source, updates all implementation docs
                               └─► feature-test-engineer reads updated implementation docs + execution_plan.md
                                     → writes tests + produces test_results.md
                                    └─► doc-cleanup-organizer reads all outputs
                                          → archives to maestro-features/<timestamp>/
                                          → clears all agent output directories
```

---

## Agents Outside the Pipeline (or Optionally Integrated)

### dev-build-launcher
Can be run separately or triggered automatically as pipeline stage 9 by appending `--run local` to the maestro prompt. It discovers the tech stack, installs dependencies (with user confirmation), executes the build, launches all services, and then enters a persistent monitoring loop — auto-repairing compilation errors and runtime crashes as they occur.

### deployment-config-agent
Run separately when you need to configure deployment CI/CD pipelines. It supports GitHub Actions, Vercel, and CircleCI. It verifies all secrets and artifact destination URLs are provided, writes environment configuration files and build scripts (Dockerfiles, Terraform, etc.), and ensures the pipeline runs the full test suite before building and pushing artifacts to your cloud provider (AWS, GCP, or Azure). It writes configuration only — it never executes deployments.

---

## Output Directory Structure

```
.claude/agents/
├── solution-architect/outputs/
│   └── architecture.md                    ← PERSISTS across pipeline runs
├── skills/
│   └── skills.md                          ← PERSISTS across pipeline runs
├── execution-planner/outputs/
│   └── execution_plan.md                  ← cleared by doc-cleanup-organizer
├── backend-engineer/outputs/
│   ├── api-docs.md                        ← cleared by doc-cleanup-organizer
│   └── implementation/
│       └── new_feature_implementation.md  ← cleared by doc-cleanup-organizer
├── frontend-engineer/outputs/
│   └── implementation/
│       └── new_feature_implementation.md  ← cleared by doc-cleanup-organizer
├── code-reviewer/outputs/
│   └── code_review_suggestions.md        ← cleared by doc-cleanup-organizer
├── feature-test-engineer/outputs/
│   └── test_results.md                   ← cleared by doc-cleanup-organizer
└── maestro-rules/
    └── rules.md                          ← static coding rules (never modified)

maestro-api-documentation/
└── api-docs.md                           ← permanent API reference

maestro-features/
└── <unix_timestamp>/
    ├── feature_requirements.md
    ├── backend_implementation.md
    ├── frontend_implementation.md
    ├── api_docs.md
    ├── test_results.md
    ├── code_review.md
    └── commit-message.md
```

---

## Maestro File Verification Table

After each stage, maestro reads these files to confirm success before advancing:

| Stage | Files Verified |
|-------|---------------|
| solution-architect | `architecture.md`, `skills.md` |
| execution-planner | `execution_plan.md` |
| backend-engineer | `api-docs.md`, backend `new_feature_implementation.md` |
| frontend-engineer | frontend `new_feature_implementation.md` |
| code-reviewer | `code_review_suggestions.md` + `git diff` confirms no source changes |
| code-review-cleanup | both updated `new_feature_implementation.md` files |
| feature-test-engineer | `test_results.md` |
| doc-cleanup-organizer | all 7 archive files exist (including `commit-message.md`) + all output dirs are empty + `maestro-api-documentation/api-docs.md` exists |
