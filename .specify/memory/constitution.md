<!--
  ============================================================================
  SYNC IMPACT REPORT
  ============================================================================
  Version change: N/A (initial) → 1.0.0

  Modified principles: None (initial version)

  Added sections:
    - Core Principles (6 principles: Extension-First Architecture,
      Context Isolation, Spec-Kit Compatibility, Progress Persistence,
      Agent Agnosticism, Graceful Termination)
    - Extension Compliance section
    - Development Workflow section
    - Governance section

  Removed sections: None (initial version)

  Templates requiring updates:
    ✅ plan-template.md - Constitution Check section compatible
    ✅ spec-template.md - Requirements alignment compatible
    ✅ tasks-template.md - Task categorization compatible

  Follow-up TODOs: None
  ============================================================================
-->

# Spec Kit Ralph Extension Constitution

## Core Principles

### I. Extension-First Architecture

All functionality MUST be packaged as a valid spec-kit extension
following the extension manifest schema (`extension.yml`). The
extension MUST NOT modify or monkey-patch spec-kit core internals.

- Commands MUST follow the `speckit.ralph.{command}` naming pattern
  as defined by the extension API (`^speckit\.[a-z0-9-]+\.[a-z0-9-]+$`).
- The extension manifest MUST declare all provided commands, hooks,
  configuration templates, and dependency requirements.
- Scripts (PowerShell and Bash) MUST be self-contained within the
  extension directory and referenced via the manifest or command
  frontmatter.
- Configuration MUST use the extension config system
  (`.specify/extensions/ralph/`) rather than ad-hoc file locations.

**Rationale**: Spec-kit extensions are the sanctioned integration
point. Bypassing the extension API creates fragile coupling to core
internals that breaks on upstream updates and prevents catalog
distribution.

### II. Context Isolation (NON-NEGOTIABLE)

Each ralph loop iteration MUST spawn a completely fresh agent
context. No iteration may inherit in-memory state from a previous
iteration.

- The orchestrator script MUST invoke the agent CLI as a new
  process per iteration, never reuse a running session.
- Inter-iteration knowledge transfer MUST occur exclusively through
  on-disk artifacts: `progress.md`, `tasks.md` checkbox state, and
  committed source files.
- A single iteration MUST complete at most one work unit (one
  phase, one user story, or one task group) to prevent context
  degradation.
- The agent prompt for each iteration MUST instruct the agent to
  read disk-persisted context rather than assume prior knowledge.

**Rationale**: Context isolation is the defining property of the
ralph loop methodology. Without it, agent context windows fill with
stale or conflicting information, causing implementation quality to
degrade over successive iterations.

### III. Spec-Kit Compatibility

The extension MUST maintain backward compatibility with the spec-kit
extension API and MUST NOT require users to modify their spec-kit
installation.

- The `requires.speckit_version` field MUST specify the minimum
  supported spec-kit version using semantic version specifiers.
- All commands, hooks, and config schemas MUST conform to the
  extension API reference (schema version 1.0).
- The extension MUST work with projects initialized by any
  supported `specify init` invocation without additional setup
  beyond `specify extension add`.
- Breaking changes to the extension's own API MUST follow semantic
  versioning: MAJOR for incompatible changes, MINOR for additive
  features, PATCH for fixes.

**Rationale**: Users adopt extensions expecting drop-in
functionality. Breaking the spec-kit contract or requiring manual
patching undermines trust and blocks catalog distribution.

### IV. Progress Persistence

All iteration state MUST be persisted to disk. No state that would
be lost on process termination is acceptable for tracking progress.

- Task completion MUST be tracked via `tasks.md` checkbox state
  (`[ ]` → `[x]`). No separate tracking database or file is
  permitted for task status.
- Iteration history MUST be appended to
  `specs/{feature}/progress.md` after each iteration with:
  timestamp, work unit attempted, tasks completed, files changed,
  and learnings discovered.
- Codebase patterns discovered during implementation MUST be
  recorded in the `## Codebase Patterns` section of `progress.md`
  so subsequent iterations can leverage them.
- The `<promise>COMPLETE</promise>` signal MUST only be emitted
  when all tasks in `tasks.md` are verified complete.

**Rationale**: Persistent state enables resumability after
interruptions, provides an audit trail for debugging failed
iterations, and is the sole mechanism for cross-iteration learning
given the context isolation constraint.

### V. Agent Agnosticism

The loop orchestration layer MUST decouple from any specific AI
agent CLI to enable future multi-agent support.

- The orchestrator MUST accept the agent CLI command as a
  configurable parameter rather than hardcoding a specific binary.
- Agent-specific invocation flags (e.g., `--agent`, `--model`,
  `--allow-all-tools`) MUST be configurable, not embedded in
  business logic.
- The completion detection mechanism (`<promise>COMPLETE</promise>`)
  MUST be agent-agnostic—it is detected in stdout regardless of
  which agent produced it.
- Initial release MAY support only GitHub Copilot CLI, but the
  architecture MUST NOT preclude adding Claude, Gemini, or other
  CLI-based agents without major refactoring.

**Rationale**: The AI tooling landscape evolves rapidly. Tight
coupling to one agent CLI creates vendor lock-in and limits adoption
by teams using different tools.

### VI. Graceful Termination

The ralph loop MUST handle all termination paths cleanly, preserving
progress and providing actionable status information.

- **Completion**: When the agent emits `<promise>COMPLETE</promise>`
  or all `tasks.md` checkboxes are marked `[x]`, the loop MUST
  exit with code 0 and a success summary.
- **Iteration limit**: When `MaxIterations` is reached with tasks
  remaining, the loop MUST exit with a non-zero code and a summary
  of completed vs. remaining tasks.
- **User interruption** (Ctrl+C): The loop MUST catch the signal,
  preserve all progress written to disk, and exit with code 130.
- **Consecutive failures**: After 3 consecutive iteration failures
  (non-zero exit from agent CLI), the loop MUST terminate with an
  error summary rather than waste resources.
- Every termination path MUST produce a summary block reporting:
  iterations run, tasks completed, tasks remaining, and final
  status.

**Rationale**: Autonomous loops that fail silently or lose progress
erode user trust. Predictable termination behavior enables
integration into CI/CD pipelines and unattended workflows.

## Extension Compliance

All changes MUST satisfy these gates before merge:

1. **Manifest Gate**: `extension.yml` MUST validate against the
   spec-kit extension schema (version 1.0). All commands MUST
   resolve to existing command files.
2. **Script Gate**: Both PowerShell (`*.ps1`) and Bash (`*.sh`)
   variants of orchestrator scripts MUST exist and produce
   identical behavior for cross-platform support.
3. **Integration Gate**: The extension MUST install cleanly via
   `specify extension add --dev` and all provided commands MUST
   execute without error on a sample project with completed
   `tasks.md`.
4. **Documentation Gate**: README, command descriptions, and
   configuration templates MUST be updated for any user-facing
   change.
5. **Compatibility Gate**: The extension MUST not break when
   installed alongside other spec-kit extensions.

## Development Workflow

The Spec-Driven Development workflow governs all changes:

1. **Specification Phase**: Requirements captured in spec documents
   with acceptance criteria and user stories.
2. **Planning Phase**: Constitution Check validates alignment with
   the six core principles before design proceeds.
3. **Task Phase**: Work items link to spec requirements, organized
   by user story for independent implementation.
4. **Implementation Phase**: Test-first development where applicable;
   ralph loop self-hosting is encouraged (use the extension to build
   the extension).
5. **Review Phase**: PRs verified against specification, constitution
   compliance, and cross-platform script parity.

## Governance

This constitution is the authoritative source for project standards:

- **Supremacy**: Constitution principles supersede conflicting
  practices, preferences, or inherited defaults from the parent
  spec-kit project.
- **Amendments**: Changes require documented rationale, version
  increment (semantic versioning), and propagation check across
  templates.
- **Compliance**: All pull requests MUST verify adherence to
  applicable principles. Reviewers MUST reference specific
  principle numbers (I–VI) when flagging violations.
- **Exceptions**: Deviations MUST be documented with justification
  and a time-boxed resolution plan in the PR description.
- **Review Cadence**: Constitution is reviewed when the spec-kit
  extension API changes or when a new major version is released.

**Version**: 1.0.0 | **Ratified**: 2026-03-06 | **Last Amended**: 2026-03-06
