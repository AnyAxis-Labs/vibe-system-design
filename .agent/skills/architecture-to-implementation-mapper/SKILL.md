---
name: architecture-to-implementation-mapper
description: Transforms high-level ADRs and C4 diagrams into a structured, dependency-aware technical implementation plan using a hybrid horizontal–vertical execution model.
---

# Architecture to Implementation Mapper

This skill bridges the gap between architectural **intent** (the why and what) and engineering **execution** (the how and when).  
It ensures that every architectural decision is translated into a traceable work item, every container in the C4 model is accounted for, and implementation risk is reduced early through vertical, module-oriented delivery.

## When to use this skill

- **Transitioning to Execution:** Use once Architecture Decision Records (ADRs) are finalized and C4 Context/Container diagrams are completed.
- **Sprint Zero / Kickoff Planning:** Generate an initial technical backlog with clear dependency ordering.
- **Gap Analysis:** Identify missing infrastructure, integration, or quality requirements implied by architecture but not yet planned.

## How to use it

### 1. Filter for "Active" Decisions

Before creating tasks, the agent must evaluate the **Status** field of every ADR.

- **IGNORE:** `Superseded`, `Deprecated`, `Rejected`
- **IMPLEMENT:** `Accepted`
- **CONDITIONAL:** `Proposed` (only if execution must begin immediately)
- **TRACE:** If an ADR supersedes another (e.g., ADR-010 replaces ADR-002), locate all tasks derived from the older ADR and mark them for **Refactor** or **Removal**, not new implementation.

All generated tasks must retain a reference to their source ADR.

---

### 2. Deconstruct the C4 Diagrams

Analyze the C4 diagrams to identify concrete units of work.

- **Provisioning Tasks:**  
  Create explicit tasks for provisioning or configuring each container and backing service (compute, database, cache, queue, object storage).

- **Connectivity & Routing:**  
  Generate tasks for every interaction between containers (network rules, service discovery, load balancers, client SDKs).

- **Contract Definition:**  
  For each data flow, create tasks to define and version contracts (OpenAPI, AsyncAPI, Protobuf), including backward-compatibility rules.

Each task must reference the relevant C4 container and relationship.

---

### 3. Extract Constraints from ADRs

Translate architectural decisions into enforceable implementation constraints.

- **Scaffolding Tasks:**  
  If an ADR specifies languages, frameworks, or runtime standards, create explicit scaffolding tasks (project structure, base libraries, build tooling).

- **Quality Gates:**  
  Map ADR requirements (security, testing, performance, observability) into:
  - Reusable checklists
  - Definition-of-Done criteria
  - CI/CD enforcement rules

These constraints apply consistently across all containers and modules.

---

### 4. Sequence the Implementation Roadmap (Hybrid Model)

Organize work using a hybrid execution strategy to minimize risk and maximize early feedback.

#### Phase 1: Foundation (Horizontal)

Establish shared, system-wide prerequisites.

- Infrastructure provisioning
- CI/CD pipelines with build gates, testing gates & artifact versioning
- Identity, secrets, and configuration management
- Observability baseline (logs, metrics, traces)
- Shared libraries and platform services

This phase enables all downstream work and is completed once.

---

#### Phase 2: Skeleton / Thin Thread (Horizontal)

Validate system connectivity with minimal logic.

- Deploy minimal versions of all containers
- Establish authentication and authorization paths
- Implement contract stubs
- Verify end-to-end request flow with placeholder logic

The goal is architectural validation, not feature completeness.

---

#### Phase 3: Vertical Module Execution (Iterative)

Divide the total system into **module-by-module vertical slices**.

For each module or feature slice:

- Implement core business logic
- Fully implement its public contracts
- Write unit tests for domain logic
- Write integration tests against real dependencies
- Run module-scoped benchmarks on known hot paths
- Enforce ADR-derived quality gates locally

Modules are prioritized by:
- Low dependency fan-in
- High architectural or technical risk
- High business value

Each module exits this phase in a production-grade, independently verifiable state.

Plans for each vertical slide should be written down to a separated .md file.

---

#### Phase 4: System-Level Hardening (Horizontal, Final)

Focus on emergent system behavior rather than individual correctness.

- End-to-end workflow testing across modules
- Stress and load testing with realistic traffic
- Cross-module hot path benchmarking
- Failure mode validation (timeouts, retries, backpressure)
- Operational readiness checks (alerts, dashboards, SLOs)

This phase assumes modules are already hardened in isolation.

---

### 5. Maintain Traceability

For every generated task:

- Reference the originating ADR(s)
- Reference the relevant C4 container and relationship
- Identify whether the task belongs to:
  - Foundation
  - Skeleton
  - A specific module slice
  - System-level hardening

This ensures that architectural changes can be traced directly to impacted implementation work and vice versa.

Refer to the [C4 Model Guidelines](https://c4model.com) for naming and modeling consistency.
