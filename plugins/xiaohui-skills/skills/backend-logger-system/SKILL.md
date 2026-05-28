---
name: backend-logger-system
description: Use when creating, reviewing, or standardizing a logger system in a Node.js + TypeScript backend project, especially when the project has inconsistent logger usage, scattered console calls, or no unified logging conventions.
---

# Backend Logger System

Use this skill to audit, design, implement, or standardize a backend logger system in Node.js + TypeScript services.

This skill is for project-wide logger consistency. It is not for one-off print statements or ad hoc debugging.

## Workflow

Follow these steps in order:

1. Audit the current logger landscape
2. Decide on one strategy: reuse, enhance, or rebuild
3. Apply the logger system changes
4. Update the project's canonical logging convention document
5. Verify the logger system end-to-end

Do not skip the audit step.
Do not implement multiple competing logger patterns in one pass.

If the project is not Node.js + TypeScript backend code, stop and explain that this skill does not apply cleanly.

## Audit

During the audit, inspect:

- existing logger modules
- direct `console.log`, `console.error`, `console.warn`, and `console.debug` usage
- startup logging
- entrypoint logging such as HTTP requests, job triggers, or CLI commands
- business-service logging
- error and stack-trace logging
- environment-specific debug behavior

Read `references/logger-checklist.md` and use it as the minimum inspection baseline.

## Decision Rules

Choose exactly one primary strategy:

- **Reuse** when the project already has a coherent logger interface and only needs minor cleanup
- **Enhance** when the project has a usable base logger but lacks conventions, coverage, or consistency
- **Rebuild** when logging is fragmented, semantically inconsistent, or mostly raw `console` calls

State the chosen strategy explicitly before editing code.

## Implementation Requirements

Your logger changes must preserve or introduce:

- one primary logger interface
- `log`, `warn`, `error`, and `debug` support
- context-aware logging
- explicit error-message and trace handling
- consistent log fields and level semantics across the project
- environment-aware debug behavior
- reduced direct `console` usage in application code

Read `references/logger-conventions.md` before implementing.

## Logging Convention Document

The canonical destination for the project logging convention is:

- the existing project logging document, if the repository already has one
- otherwise `docs/logging-conventions.md`

Update that file when applying this skill, and use it as the authoritative source for logger usage rules.

## Required Outputs

When you use this skill, your final output must include:

- the chosen strategy and why
- the files changed
- the logging convention file added or updated
- the verification steps you ran
- any remaining migration debt if raw `console` usage still exists

## Verification

Verify as many of these as the project supports:

- application startup emits a logger-based message
- a normal business-path log is visible
- an error path prints both message and trace
- context labels render correctly
- debug output is appropriately gated by environment
- application code does not introduce new scattered raw `console` calls

If you cannot run the application, use targeted tests, type checks, and source inspection, and state what was and was not verified.
