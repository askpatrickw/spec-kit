<!--
Sync Impact Report
- Version change: [CONSTITUTION_VERSION] → 1.0.0
- Modified principles: [PRINCIPLE_1_NAME] → I. Spec-Driven Workflow, [PRINCIPLE_2_NAME] → II. Testable, Independent Stories, [PRINCIPLE_3_NAME] → III. Simplicity by Default, [PRINCIPLE_4_NAME] → IV. Traceability & Documentation, [PRINCIPLE_5_NAME] → V. Quality Gates & Compliance
- Added sections: Additional Constraints, Workflow & Reviews
- Removed sections: None
- Templates requiring updates:
  - `.specify/templates/plan-template.md` ✅ updated
  - `.specify/templates/spec-template.md` ✅ checked (no change)
  - `.specify/templates/tasks-template.md` ✅ checked (no change)
- Follow-up TODOs: TODO(RATIFICATION_DATE): original ratification date unknown
-->
# Spec Kit Constitution

## Core Principles

### I. Spec-Driven Workflow
All work MUST follow the sequence: constitution → spec → plan → tasks →
implementation. Each feature MUST produce a spec, plan, and tasks artifact before any
implementation begins. Rationale: consistent artifacts create repeatable, auditable results.

### II. Testable, Independent Stories
User stories MUST be prioritized, independently testable, and deliver value on their own.
Acceptance scenarios MUST be explicit and verifiable. Rationale: independence enables MVP
delivery, parallel execution, and clearer validation.

### III. Simplicity by Default
Choose the simplest architecture and dependencies that satisfy the spec. Any added
complexity MUST be justified in the plan (and recorded in Complexity Tracking when
violations exist). Rationale: simpler systems are faster to build, review, and maintain.

### IV. Traceability & Documentation
Every requirement MUST map to a user story, tasks MUST map to stories, and each task
MUST reference concrete file paths. Update plan and quickstart documentation whenever
behavior or setup changes. Rationale: traceability prevents gaps and keeps onboarding
reliable.

### V. Quality Gates & Compliance
Constitution checks MUST run before plan research and after plan design. Tests are
OPTIONAL and only included when explicitly requested in the feature spec; when included,
they MUST be written to fail before implementation. Rationale: quality gates reduce rework
while preserving flexibility.

## Additional Constraints

- The constitution is the highest-order document for this repo and overrides templates.
- Changes that increase scope or complexity MUST be captured in the plan and tasks.
- Use placeholder TODOs for unknowns and resolve before implementation begins.

## Workflow & Reviews

- Plans MUST include a Constitution Check section derived from these principles.
- Reviews MUST verify traceability from requirements → stories → tasks → code.
- Deviations from simplicity or scope MUST be documented in Complexity Tracking.

## Governance

- Amendments require a documented rationale, semantic version bump, and review in PRs.
- Major changes MUST include migration guidance for existing specs/plans/tasks.
- Compliance is reviewed during plan creation and before implementation starts.
- This constitution supersedes templates and local guidance when conflicts exist.

**Version**: 1.0.0 | **Ratified**: TODO(RATIFICATION_DATE): original ratification date unknown | **Last Amended**: 2026-01-30
