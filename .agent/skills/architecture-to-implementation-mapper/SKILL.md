---
name: architecture-to-implementation-mapper
description: Transforms high-level ADRs and C4 diagrams into a structured technical implementation plan. Use this when transitioning from the design phase to active development.
---

# Architecture to Implementation Mapper

This skill bridges the gap between architectural "intent" (the **why** and **what**) and engineering "execution" (the **how** and **when**). It ensures that every architectural decision is translated into a trackable work item and every container in the C4 model is accounted for in the backlog.

## When to use this skill

- **Transitioning to Execution:** Use this once Architecture Decision Records (ADRs) are finalized and C4 Context/Container diagrams are completed.
- **Backlog Generation:** This is helpful for generating an initial technical backlog for **Sprint Zero** or project kickoff.
- **Gap Analysis:** Use this to identify missing infrastructure or integration requirements that are implied by the architecture but not yet planned.

## How to use it

### 1. Filter for "Active" Decisions
Before creating tasks, the agent must check the **Status** field of every ADR.
- **IGNORE:** Any record with a status of `Superseded`, `Deprecated`, or `Rejected`.
- **IMPLEMENT:** Only records with a status of `Accepted` or `Proposed` (if immediate action is required).
- **TRACE:** If a decision supersedes another (e.g., ADR-010 replaces ADR-002), the agent must look for existing tasks related to ADR-002 and mark them for "Refactor" or "Removal" instead of "New Implementation".

### 2. Deconstruct the C4 Container Diagram
Analyze the C4 Container level to identify the **Physical Units of Work**.
- **Provisioning Tasks:** Create specific tickets for provisioning every unique container (e.g., "Set up Azure SQL Database" or "Configure S3 Bucket for Static Assets").
- **Connectivity & Routing:** Create tasks for the interactions (arrows) between containers (e.g., "Configure Load Balancer rules" or "Implement gRPC client for Service X").
- **Contract Definition:** Generate tasks for defining schemas (OpenAPI, AsyncAPI, or Protobuf) based on the data flows between containers.

### 3. Extract Constraints from ADRs
Review the ADR log to define the **Standards and Tech Stack**.
- **Scaffolding Tasks:** If an ADR specifies a framework (e.g., "Use Go with Gin"), create a task for project scaffolding
- **Quality Gates:** Map ADR requirements (e.g., "Must have 80% test coverage" or "Use OAuth2 for auth") into sub-tasks or "Definition of Done" criteria for all containers.

### 4. Sequence the Implementation Roadmap
Organize the extracted tasks into a logical hierarchy to manage dependencies:
- **Phase 1: Foundation (The Environment):** Infrastructure, CI/CD pipelines, and shared libraries derived from ADRs.
- **Phase 2: Skeleton (The Connectivity):** Deploying "thin-thread" versions of containers to verify network and security paths defined in C4.
- **Phase 3: Core Logic (The Features):** Building out internal components and business logic.
- **Phase 4: Hardening (Testing & Benchmark)**: Write unit/integration/e2e tests & benchmark hot paths

### 5. Maintain Traceability
- For every generated task, ensure it includes a reference to its source. 
- This ensures that if a decision is superseded, the team knows exactly which implementation tasks are impacted. Refer to the [C4 Model Guidelines](https://c4model.com) for naming consistency.

---