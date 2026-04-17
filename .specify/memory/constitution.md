<!--
SYNC IMPACT REPORT
==================
Version change: (none) → 1.0.0
Modified principles: N/A (initial constitution)
Added sections:
  - Core Principles (5 principles)
  - Data Integrity & Consistency
  - Development Workflow
  - Governance
Templates updated:
  - .specify/templates/plan-template.md ✅ (Constitution Check section is generic/compatible)
  - .specify/templates/spec-template.md ✅ (no principle-specific references)
  - .specify/templates/tasks-template.md ✅ (no principle-specific references)
Deferred TODOs:
  - TODO(RATIFICATION_DATE): Project ratification date unknown — set when formally adopted by team.
-->

# Managed Node Counting Constitution

## Core Principles

### I. Accuracy First

Node counts MUST be accurate and authoritative at all times. Stale, estimated, or
approximate counts are not acceptable in any production code path. Every count
operation MUST reflect the true state of managed nodes at the moment of query.
Caching is permitted only when cache invalidation is explicitly implemented and
tested.

### II. Idempotent Operations

All counting and reconciliation operations MUST be idempotent. Running any
operation N times MUST produce the same result as running it once. Callers
MUST be able to safely retry without risk of double-counting or state corruption.
Side effects (writes, external calls) MUST be bounded and documented.

### III. Test-First (NON-NEGOTIABLE)

TDD is mandatory. Tests MUST be written and confirmed to fail before any
implementation code is written. The Red-Green-Refactor cycle is strictly
enforced. No feature is considered done without a failing test that becomes
passing. Integration tests against real node sources are preferred over mocks
for count-correctness verification.

### IV. Observability

All node counting operations MUST emit structured logs capturing: operation
name, node scope/filter, count result, and elapsed time. Errors MUST include
sufficient context to diagnose root cause without access to source code.
Silent failures are prohibited — a failed count MUST surface an explicit error.

### V. Simplicity

Complexity MUST be justified. Start with the simplest implementation that
satisfies the requirement. Abstractions are introduced only when three or more
concrete cases demand them (rule of three). YAGNI applies: do not design for
hypothetical future node types, sources, or scale requirements.

## Data Integrity & Consistency

Node count results MUST be reproducible: given the same input state, the same
count MUST always be returned. Eventual consistency in node sources MUST be
handled explicitly — the code MUST document its consistency guarantees (e.g.,
"reflects state as of last sync"). No implicit assumptions about source
freshness are permitted.

- All data sources MUST be versioned or timestamped in query responses.
- Schema changes to node representations MUST be backward-compatible or
  accompanied by a migration with rollback path.
- Count discrepancies between sources MUST be surfaced as explicit errors or
  warnings, never silently reconciled.

## Development Workflow

- All changes require a passing test suite before merge.
- Features follow the Specify workflow: spec → plan → tasks → implement.
- Each user story MUST be independently deliverable and testable.
- Commits MUST be atomic: one logical change per commit.
- PRs MUST reference the spec/plan that motivated the change.
- Complexity violations (deviations from Principle V) MUST be documented in
  plan.md under the Complexity Tracking section with explicit justification.

## Governance

This constitution supersedes all other development practices and conventions
for this project. Amendments require:

1. A documented rationale (why the current principle is inadequate).
2. An impact assessment (which existing features/tests are affected).
3. A migration plan if existing code violates the new principle.
4. Version increment per semantic versioning rules (see below).

**Versioning policy**:
- MAJOR: Removal or redefinition of an existing principle.
- MINOR: New principle or section added.
- PATCH: Clarification, wording fix, or non-semantic refinement.

All PRs and code reviews MUST verify compliance with the five core principles.
Violations MUST be called out explicitly; they cannot be merged without a
justified exception recorded in the PR description.

Use `.specify/memory/constitution.md` as the authoritative runtime reference
during development planning and review.

**Version**: 1.0.0 | **Ratified**: TODO(RATIFICATION_DATE): set when formally adopted | **Last Amended**: 2026-04-17
