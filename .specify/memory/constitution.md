<!--
Sync Impact Report - Constitution v1.0.0
═══════════════════════════════════════════════════════════════════════════════
Version Change: [UNVERSIONED] → 1.0.0
Ratification Date: 2025-10-08
Last Amendment: 2025-10-08

Version Bump Rationale: MAJOR (1.0.0)
- Initial constitution ratification
- Establishes three core principles from user input
- Defines foundational governance structure

Core Principles Established:
1. Local-First & Security (新增)
2. User Experience Excellence (新增)
3. Scope Discipline (新增)

Template Synchronization Status:
✅ spec-template.md - Aligned with scope discipline principle (single-feature specs)
✅ plan-template.md - Aligned with local-first and security principles (constitution check)
✅ tasks-template.md - Aligned with scope discipline (user story breakdown)
⚠️  Command files (.claude/commands/*.md) - Should be reviewed for agent-specific references

Follow-up Items:
- None - all placeholders resolved
═══════════════════════════════════════════════════════════════════════════════
-->

# Local Assistant v2 Constitution

## Core Principles

### I. Local-First & Security

**Non-negotiable rules:**

- All data processing MUST occur locally unless explicitly approved by user for external operations
- User credentials, API keys, and sensitive information MUST be stored securely (encrypted at rest, never logged)
- External network calls MUST be explicit, documented, and require user consent
- Dependencies MUST be audited for security vulnerabilities before inclusion
- No telemetry or analytics without explicit opt-in from user

**Rationale**: This project serves as a personal assistant running on user machines. Privacy and data sovereignty are paramount—users must maintain full control over their data without unexpected external exposure.

### II. User Experience Excellence

**Non-negotiable rules:**

- User interfaces (CLI, UI, API) MUST prioritize clarity and intuitiveness over feature density
- Error messages MUST be actionable (explain what went wrong AND how to fix it)
- Response times MUST be optimized (target: interactive operations <200ms, batch operations show progress)
- User workflows MUST be tested with real user journeys before completion
- Documentation MUST include quickstart guides and common use-case examples

**Rationale**: A personal assistant is only valuable if it's pleasant and efficient to use. Poor UX creates friction that undermines the tool's utility, regardless of technical sophistication.

### III. Scope Discipline

**Non-negotiable rules:**

- Each feature specification MUST address a single, well-defined user problem
- Specifications exceeding 3 independent user stories MUST be split into multiple specs
- Feature creep during implementation MUST be rejected (captured as separate future specs)
- Every requirement MUST map to a concrete user scenario (no "nice-to-have" abstractions)
- Implementation MUST deliver MVP (P1 user story) before expanding to P2/P3 stories

**Rationale**: Large, unfocused specifications lead to scope creep, delayed delivery, and brittle architecture. Disciplined scope boundaries enable incremental value delivery and maintainable systems.

## Security Requirements

- Credential storage MUST use OS-native secure storage (Keychain on macOS, Credential Manager on Windows, Secret Service on Linux)
- File permissions for configuration files MUST restrict access to owner-only (0600)
- All external API integrations MUST support token refresh and graceful degradation on auth failure
- Logs MUST NOT contain sensitive data (use redaction patterns for credentials, PII)

## Development Workflow

### Planning Gates

Before implementation begins, every feature MUST have:

1. **Specification Review**: User scenarios are clear, testable, and prioritized (P1 = MVP)
2. **Constitution Compliance**: Feature aligns with Local-First, UX Excellence, and Scope Discipline principles
3. **Scope Validation**: Feature does not exceed 3 user stories; if larger, split into multiple specs

### Implementation Quality Gates

- **Local-First Verification**: All data paths reviewed to ensure local processing; external calls documented
- **UX Testing**: Core user journey (P1 story) validated manually or with users before P2/P3 implementation
- **Scope Adherence**: Feature creep identified and deferred to separate specs; only spec-defined stories implemented

## Governance

This constitution supersedes all other project practices and guidelines. All feature specifications, implementation plans, and code reviews MUST verify compliance with these principles.

### Amendment Procedure

1. Proposed amendment MUST be documented with rationale and impact analysis
2. Amendment MUST include updates to affected templates (spec, plan, tasks)
3. Version MUST be bumped according to semantic versioning:
   - **MAJOR**: Principle removed, redefined, or governance structure changed
   - **MINOR**: New principle added or existing principle materially expanded
   - **PATCH**: Clarifications, wording improvements, non-semantic refinements
4. Amendment date MUST be recorded in `Last Amended` metadata

### Complexity Justification

Any violation of constitutional principles (e.g., introducing external-first features, creating unfocused specs >3 stories) MUST be justified in the implementation plan's "Complexity Tracking" section with:

- Why the violation is necessary for user value
- What simpler alternatives were considered and why they were insufficient
- How the violation will be contained (e.g., external calls isolated, large spec phased across iterations)

**Version**: 1.0.0 | **Ratified**: 2025-10-08 | **Last Amended**: 2025-10-08
