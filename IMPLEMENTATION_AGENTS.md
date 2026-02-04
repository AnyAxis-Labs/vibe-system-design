# Vibe System Implementation Rules

## Purpose

This file defines **mandatory rules** for any agent participating in the Vibe System implementation.

It exists to:
- prevent architectural drift
- enforce execution discipline
- ensure progress is traceable and restart-safe
- standardize agent behavior across phases

Failure to follow this file invalidates implementation work.

---

## 1. Mandatory Context Loading (Before Any Work)

Before executing any phase, the agent MUST:

1. Load architecture context using  
   `.agent/skills/architecture-context-loader`
2. Load the implementation execution model by reading:
   - `docs/implementation-plan/README.md`
   - `docs/implementation-plan/00-overview.md`

No code, tasks, or plans may be produced before this completes.

---

## 2. Required MCPs (Tooling)

The agent MUST identify and use appropriate MCPs when needed.

### Mandatory MCP Categories

- **Third-party / API Integration MCP**  
  For interacting with external services, infrastructure APIs, and system tools.

- **Context7 MCP**  
  For tasks requiring web search, specification lookup, or verification.

- **Dependency Knowledge MCPs**  
  Used to select correct libraries and versions:
  - `docs.rs` (Rust)
  - `pkg.go.dev` (Go)
  - ecosystem equivalents when applicable

Dependency selection MUST prioritize:
- stable releases
- active maintenance
- compatibility with ADR constraints

---

## 3. Mandatory Skills

The agent should look for skills that are relevant to the task before performing it. Examples:

- **Testing Skill**  
  Unit, integration, and end-to-end tests aligned with phase and module exit criteria.

- **Benchmarking Skill**  
  Performance tests on declared hot paths with recorded results.

- **Pull Request Writing Skill**  
  Pull requests MUST include:
  - scope summary
  - referenced ADRs
  - test and benchmark evidence

Skipping any required skill is considered a process failure.

---

## 4. Progress Logging and State Tracking (Non-Negotiable)

For every meaningful unit of work, the agent MUST:

- Append to  
  `progress-logs/YYYY-MM-DD-<phase-or-module>.md`
- Update  
  `implementation_state.md`

Rules:
- Logs are append-only
- Logs must reflect completed work, not intent
- `implementation_state.md` must always be consistent with logs
- Phases or modules may not be marked complete without evidence

Silent progress is considered invalid.

---

## 5. Phase Discipline

The agent MUST respect phase boundaries:

- Phase 0 / 0.5: context and knowledge preparation only
- Phase 1–4: execute strictly according to the implementation plan
- No phase skipping
- No scope leakage between phases

All work MUST reference the corresponding file in `docs/implementation-plan/`.

---

## 6. Phase 3 Execution Model (Vertical Modules)

During **Phase 3**, modules SHOULD be implemented using the **Ralph Wiggum Loop**:

