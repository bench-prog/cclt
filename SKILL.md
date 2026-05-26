---
name: cclt-progressive-spec
description: Use when collaborating with AI on code changes in existing projects where requirements need structured documentation before implementation, context is expensive, or ad-hoc chat coding leads to excessive trial-and-error rounds. Apply when working with terminal AI coding agents like Claude Code, opencode, or similar tools.
---

# Progressive Spec Coding

## Overview

A lightweight, spec-driven framework for human-AI collaborative coding. Core principle: **Code is cheap, context is expensive.** Structured documentation (Spec) reduces trial-and-error rounds by giving the AI high-quality input upfront.

## When to Use

- Adding features or fixing bugs in existing codebases with AI assistance
- Trial-and-error chat coding exceeds 5-10 rounds for a single change
- Team needs consistent quality and traceability across AI-generated changes
- Working with terminal AI agents (Claude Code, opencode, Cline, Aider)

When NOT to use: greenfield prototypes where velocity matters more than structure, or one-liner fixes where chat suffices.

## Core Principles

1. **No Spec, No Code** — Without a structured spec document, no code may be written.
2. **Spec is Truth** — When spec and code conflict, the code is wrong.
3. **Reverse Sync** — During execution, if reality diverges from spec, update the spec first, then the code.
4. **Progressive Complexity** — Different complexity levels expose different workflow depths. A 5-minute field change should not pay the cost of a full spec-review-archive cycle.

## Progressive Complexity Levels

This is the framework's core differentiator. Do not force all changes through the full pipeline.

**Decision Tree (top-down, first match wins):**

1. Does it touch money / state machine / permissions? -> :red_circle: **Complex**
2. Does it cross modules or modify public APIs / data models? -> :red_circle: **Complex**
3. Are boundaries unclear (lots of unknowns, requires exploratory coding)? -> :red_circle: **Complex**
4. File count > 10? -> :red_circle: **Complex**
5. File count 3-10, single module, clear boundaries, no state changes? -> :yellow_circle: **Medium**
6. Everything else (<=3 files, no business rule changes, no cross-module impact) -> :green_circle: **Simple**

| Level | Workflow | Review |
|-------|----------|--------|
| :green_circle: Simple | Chat-driven change, record in `changes/<name>/log.md` only | Self-check |
| :yellow_circle: Medium | `cclt-propose` (lightweight spec, may skip task breakdown) -> `cclt-apply` -> spot-check | Human spot review |
| :red_circle: Complex | Full `cclt-propose` -> `cclt-apply` -> `cclt-review` -> `cclt-archive` | Mandatory two-phase review |

## Command Reference

All commands use the `cclt-` prefix to avoid colliding with shell or tool builtins.

| Command | Purpose | Human / AI Role |
|---------|---------|-----------------|
| `cclt-init` | Analyze project structure and populate `rules/project-context.md` | AI executes once per project |
| `cclt-propose <description>` | Research -> clarifying questions -> generate `spec.md` + `tasks.md` | Human-led, AI-assisted |
| `cclt-apply <change-name>` | Execute tasks atomically; verify with evidence after each | AI-led, Human-reviewed |
| `cclt-fix <change-name> [desc]` | Incremental fixes after review; sync docs first | AI-led, Human-reviewed |
| `cclt-review <change-name>` | Two-phase review: Spec Compliance -> Code Quality (subagents) | Subagent-only, read-only |
| `cclt-test <change-name>` | Generate test spec and execute Red/Green TDD (standalone phase) | AI-led, Human-reviewed |
| `cclt-archive <change-name>` | Move to `archives/`, confirm knowledge capture into `knowledge/` | Human-confirmed, AI-executed |

**cclt-test placement**: Independent command for test-generation phases. Inside `cclt-apply`, every task must still show compile/build verification; `cclt-test` is for dedicated test coverage work after core implementation.

## Workflow

```
Complex (:red_circle:): cclt-propose -> cclt-apply -> cclt-review -> cclt-archive
Medium (:yellow_circle:): cclt-propose -> cclt-apply -> spot-check -> cclt-archive
Simple (:green_circle:): chat -> log -> done
                    ^
              cclt-fix (any level)
```

**cclt-propose (HARD-GATE)**
- AI researches current code first; every claim cites file path + class/function name.
- Ask one question at a time, with 2-3 options + a recommendation.
- Generate spec in segments; wait for human confirmation between segments.
- All clarifications resolved + explicit human confirmation required before `cclt-apply`.
- Output: `changes/<name>/spec.md` + `changes/<name>/tasks.md` + `changes/<name>/log.md`.

**cclt-apply (Zero Deviation)**
- Plan is the contract; AI is the printer.
- Default step-by-step mode (pause after each task for confirmation).
- Each task must show verifiable evidence (compile output / test output / invocation result). No "should be fine" claims.
- One task = one atomic commit. Auto-commit after each task.
- Real-time knowledge capture: after each task, check for pitfalls or implicit rules; write immediately to `log.md`.

**cclt-review (Two-Phase Isolation)**
- Phase 1: Spec Compliance - verify every spec item exists in actual code. "Don't trust reports, trust code."
- Phase 2: Code Quality - check rules, security, naming, error handling. Critical/Important/Minor分级.
- Phase 1 must PASS before Phase 2 starts. Either FAIL -> return to `cclt-apply` or `cclt-fix`.
- **Reviewer constraints** (best-effort isolation):
  - Run in a fresh subagent/session with no prior implementation context.
  - Enforce read-only behavior via prompt engineering: the reviewer prompt must explicitly state "You are a read-only reviewer. Do NOT write, edit, or propose code changes. Only report findings."
  - Human fallback: if subagent isolation is unavailable, human performs Phase 1 (spec compliance) using the provided checklist, AI performs Phase 2 (code quality).

**cclt-fix (Reverse Sync)**
- Every fix must sync update `spec.md`, `tasks.md`, and `log.md` before touching code.

**cclt-archive (Knowledge Flywheel)**
- Present each item from `log.md`; ask human whether to capture into `knowledge/index.md`.
- Move `changes/<name>/` to `archives/<name>/`.

## Directory Layout

Create `cclt/` at project root:

```
cclt/
├── agents/
│   ├── copilot-prompt.md          # Main agent prompt (loads rules/knowledge/changes)
│   ├── spec-reviewer.md           # Phase-1 review agent prompt
│   └── code-quality-reviewer.md   # Phase-2 review agent prompt
├── rules/                         # Always-loaded constraints
│   ├── project-context.md         # Stack, structure, dependencies
│   ├── coding-style.md            # Naming, formatting, patterns
│   ├── security.md                # Security red lines
│   └── domain-rules.md            # Business-domain constraints
├── knowledge/                     # Loaded on demand via keywords
│   └── index.md                   # Lightweight trigger-keyword index
├── changes/
│   ├── templates/                 # spec.md, tasks.md, test-spec.md, log.md
│   └── <change-name>/             # One directory per change
└── archives/                      # Completed changes moved here
```

## Template Skeletons

When creating `agents/copilot-prompt.md`, include these template structures so the AI can generate consistent documents.

### spec.md (filled example)
```markdown
# Add JWT Refresh Token Endpoint
> status: propose
> complexity: :yellow_circle:

## 1. Background & Goal
Currently users must re-login after JWT accessToken expires. Bad UX.
Goal: Add `/auth/refresh` endpoint to exchange refreshToken for new accessToken.

## 2. Current Code (Research Findings)
- Token generation logic in `src/auth/jwt.ts::generateToken()` (L45-62); only returns accessToken.
- Login endpoint `src/auth/controller.ts::login()` (L88-102) returns `{ accessToken }`, no refreshToken.
- DB table `users` has `id`, `password_hash`, but no `refresh_token` field.

## 3. Functional Requirements
- [ ] Feature 1: Issue refreshToken during login
  - Input: unchanged (username/password)
  - Processing: `generateToken()` creates accessToken + random refreshToken -> persist to DB
  - Output: `{ accessToken, refreshToken, expiresIn: 3600 }`
- [ ] Feature 2: Add `/auth/refresh` POST endpoint
  - Input: Body `{ refreshToken: string }`
  - Processing: validate refreshToken signature -> query DB for match and expiry -> generate new accessToken
  - Output: `{ accessToken, expiresIn: 3600 }` or `401 Unauthorized`

## 4. Business Rules
- refreshToken valid for 7 days, non-refreshable.
- Max 5 active refreshTokens per user (prevent unbounded growth).

## 5. Data Changes
| Operation | Table | Field/Index | Note |
|-----------|-------|-------------|------|
| ADD | users | refresh_tokens (JSON array) | stores {token, expiresAt} list |

## 6. Interface Changes
| Operation | Endpoint | Method | Change |
|-----------|----------|--------|--------|
| MODIFY | /auth/login | POST | Response adds refreshToken |
| ADD | /auth/refresh | POST | New endpoint |

## 7. Impact Scope
- Login response format changes -> frontend adaptation required (already notified).
- No state machine / money / permission changes.

## 8. Risks & Concerns
- DB migration: existing users need default empty array for refresh_tokens field.

## 9. Open Questions
- [x] Q1: Store refreshToken in separate table or JSON field in users table?
  -> Answer: JSON field in users table to reduce migration complexity.

## 10. Technical Decisions
- Decision: refreshToken uses crypto.randomUUID(), 256-bit random string.
- Rejected: JWT self-contained refreshToken (hard to revoke).

## 11. Execution Log
| Task | Status | Files Changed | Note |

## 12. Confirmation Record (HARD-GATE)
- [x] All Open Questions resolved
- [x] Technical design confirmed (no major disagreements)
- [x] Impact scope assessed and accepted
- [x] Risks identified and mitigations planned
- **Confirmed by**: ___  **Date**: ___
```

### tasks.md (filled example)
```markdown
# Tasks - Add JWT Refresh Token Endpoint
> Breakdown order: data model -> interface protocol -> core impl -> orchestration -> entry layer

## Task 1: DB Migration + refreshToken Storage Model
- **Goal**: Add refresh_tokens JSON field to users table, write migration script
- **Files**:
  - migrations/20240526_add_refresh_tokens.sql - new, add field with default '[]'
  - src/models/user.ts - modify, add refreshTokens array to User type
- **Key signature**:
  ```ts
  interface RefreshToken {
    token: string;
    expiresAt: number; // timestamp
  }
  // Add to User type:
  refreshTokens: RefreshToken[];
  ```
- **Dependencies**: none
- **Acceptance criteria**: migration script runs forward/backward, TypeScript compiles with no type errors
- **Verification command**: `npm run migrate:up && npm run typecheck`

## Task 2: Issue refreshToken at Login
- **Goal**: Modify login logic to generate and persist refreshToken
- **Files**:
  - src/auth/jwt.ts - modify, add `generateRefreshToken()`
  - src/auth/controller.ts - modify, login() response adds refreshToken
  - src/auth/service.ts - modify, saveRefreshToken() writes to DB
- **Key signature**:
  ```ts
  function generateRefreshToken(): string; // returns 256-bit random UUID
  ```
- **Dependencies**: Task 1
- **Acceptance criteria**: login response contains accessToken + refreshToken; DB record exists in users.refresh_tokens
- **Verification command**: `npm test -- auth.login`

## Task 3: Add /auth/refresh Endpoint
- **Goal**: Implement refresh endpoint, validate and issue new accessToken
- **Files**:
  - src/auth/controller.ts - modify, add `refresh()` handler
  - src/auth/service.ts - modify, add `refreshAccessToken()`
  - src/routes.ts - modify, register POST /auth/refresh
- **Key signature**:
  ```ts
  async function refreshAccessToken(refreshToken: string)
    : Promise<{ accessToken: string; expiresIn: number }>;
  ```
- **Dependencies**: Task 2
- **Acceptance criteria**: valid refreshToken returns new accessToken; invalid/expired returns 401
- **Verification command**: `npm test -- auth.refresh`

## Change Summary
Fill after /apply completes
- **Total files**: X files
- **Spec-Plan deviations**:
- **Open issues**:
```

## Git Conventions (Mandatory)

Regardless of tool, enforce these:

1. **No master/main branch edits** - check current branch; stop if on master.
2. **One task = one commit** - auto-commit after each `cclt-apply` task or `cclt-fix`.
3. **Commit must compile** - run build check before committing.
4. **No auto-push** - push is human-triggered only.
5. **Message format**: `[<change-name>] <summary>` (use project's primary communication language)

## Debugging Protocol

Four-phase systematic debugging. Forbidden to modify code before root cause is confirmed.

1. **Root Cause Investigation** - Collect evidence (logs, stack traces, reproduction steps).
2. **Pattern Analysis** - Identify recurring pattern or trigger condition.
3. **Hypothesis Verification** - Form hypothesis; design minimal experiment to confirm.
4. **Fix Implementation** - Apply fix; verify with same reproduction steps.

**Iron law**: Do not skip phases. No "let me try this and see" without a validated hypothesis.

## What Humans Do

AI coding changes the human role from "do everything" to "manage and verify":

| Phase | Human | AI |
|-------|-------|-----|
| Research | Medium freedom | Scout (scan code, collect facts) |
| Design | High freedom | Advisor (propose options, analyze trade-offs) |
| Planning | Low freedom | Drafter (precise file paths and signatures) |
| Execution | Zero freedom | Worker (execute plan exactly) |
| Review | Medium freedom | Inspector (check against standards) |

**Common collaboration mistakes**:
- Mixing exploration and command in one message.
- Letting AI write code during research phase.
- Giving AI high freedom during execution (should be zero).

## Prompt Design Checklist

When writing or adapting `agents/copilot-prompt.md`:

1. **Identity**: Senior engineer partner, not a code generator.
2. **Startup ritual**: Load all `rules/`, check `changes/` for in-flight work, report status.
3. **Command router**: Map natural language to `cclt-*` commands; reject out-of-scope requests politely.
4. **Research discipline**: Every code conclusion must include file path + class/function name. No "I think" or "usually".
5. **Execution strategy**: Default step-by-step (pause for confirmation). Support batch mode and emergency stop.
6. **Safety**: Highlight :warning: for money / state-change logic; require human review.
7. **Knowledge capture**: Propose capturing valuable findings into `knowledge/`.
8. **Progressive gate**: AI must classify complexity (:green_circle:/:yellow_circle:/:red_circle:) before choosing workflow depth.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Mixing discussion and commands in one message | One intent per message: explore / decide / instruct / review |
| AI writes code during the research phase | Stop it: "I don't need code yet; tell me current state and risks first" |
| Giving AI high freedom during execution | Execution freedom = zero; design phase = high freedom |
| Skipping Reverse Sync when reality diverges | Always update spec first, then code |
| Treating spec as a one-time input | Spec is a living contract maintained through the entire change lifecycle |
| Expecting AI to understand the whole system | Use `knowledge/index.md` to feed precise context; never rely on AI "knowing" the domain |
| Forcing simple changes through full pipeline | Use :green_circle:/:yellow_circle:/:red_circle: levels; match process weight to essential complexity |
