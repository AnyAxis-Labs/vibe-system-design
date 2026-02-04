# Backend Implementation Plan

## Overview

This implementation plan translates the **BTC Tap Trading Application (Hardened v1.3)** architecture into an actionable, dependency-aware technical roadmap.

**Architecture Reference:**
- `prd.md` — Product requirements
- `current_state.md` — Current system state (Hardened v1.3)
- `docs/adr/` — 10 Accepted ADRs
- `docs/diagrams/` — C4 Context and Container diagrams

---

## Plan Structure

This plan follows the **hybrid horizontal-vertical execution model**:

### Phase 1: Foundation (Horizontal)
**File:** [`01-phase-foundation.md`](./01-phase-foundation.md)

Establish shared infrastructure and scaffolding:
- VM provisioning and system setup
- SQLite with WAL mode (ADR-001)
- Redis configuration (ADR-004)
- Go project scaffolding (ADR-002)
- CI/CD and deployment scripts
- Nginx reverse proxy
- Observability baseline

**Duration:** Days 1-2  
**Dependencies:** None

---

### Phase 2: Skeleton (Horizontal)
**File:** [`02-phase-skeleton.md`](./02-phase-skeleton.md)

Deploy minimal versions of all containers to validate connectivity:
- Minimal API server with health endpoint
- Minimal WebSocket server
- Minimal price ingestion
- Connectivity validation
- systemd service definitions

**Duration:** Days 2-3  
**Dependencies:** Phase 1

---

### Phase 3: Vertical Modules (Iterative)

Five independent vertical slices, prioritized by dependency order:

| Order | Module | File | Duration | Dependencies |
|-------|--------|------|----------|--------------|
| 1 | Core Infrastructure | [`03-module-core-infrastructure.md`](./03-module-core-infrastructure.md) | Days 3-4 | Phase 2 |
| 2 | Auth & Security | [`04-module-auth-security.md`](./04-module-auth-security.md) | Days 4-5 | Module 1 |
| 3 | Bet Ingestion | [`05-module-bet-ingestion.md`](./05-module-bet-ingestion.md) | Days 5-7 | Modules 1, 2 |
| 4 | Price Feed | [`06-module-price-feed.md`](./06-module-price-feed.md) | Days 7-8 | Module 1 |
| 5 | Resolution & Archival | [`07-module-resolution-archival.md`](./07-module-resolution-archival.md) | Days 8-9 | Modules 1, 4 |

---

### Phase 4: System Hardening (Horizontal)
**File:** [`08-phase-hardening.md`](./08-phase-hardening.md)

Validate emergent system behavior and operational readiness:
- End-to-end workflow testing
- Load testing (3K RPS target)
- Failure mode testing
- Stress testing (spike and soak)
- Operational readiness (metrics, alerts, runbooks)
- Security validation

**Duration:** Days 9-10  
**Dependencies:** All Phase 3 modules

---

## ADR to Module Mapping

| ADR | Title | Implemented In |
|-----|-------|----------------|
| 001 | SQLite as Primary Database | Phase 1, Module 1 |
| 002 | Go as Backend Language | Phase 1 |
| 003 | Monolithic Single-VM Deployment | Phase 1, 2, 4 |
| 004 | Redis for Cache and Pub/Sub | Phase 1, Modules 1, 4 |
| 005 | Write Batching for Bet Ingestion | Module 3 |
| 006 | Archival Strategy for Bet History | Module 5 |
| 007 | Replay Protection for Web3 Auth | Module 2 |
| 008 | Stale Price Circuit Breaker | Module 4 |
| 009 | Conditional Balance Deduction | Module 3 |
| 010 | Hybrid Transaction Ledger | Module 3 |

> **Note:** ADR-010 **extends** ADR-009 — both must be implemented together in the same transaction.

---

## Critical Path

```
Phase 1 ──► Phase 2 ──► Module 1 ──► Module 2 ──► Module 3
                          │                       ▲
                          ▼                       │
                        Module 4 ────────────────┘
                          │
                          ▼
                        Module 5 ──► Phase 4
```

**Critical Path:** Phase 1 → Phase 2 → Module 1 → Module 2 → Module 3 → Phase 4

Modules 4 and 5 can be worked in parallel with Module 3 once Module 1 is complete.

---

## Task Summary

| Phase/Module | Tasks | Est. Hours | Owner |
|--------------|-------|------------|-------|
| Phase 1: Foundation | 7 | 20 | Backend Engineer |
| Phase 2: Skeleton | 5 | 12 | Backend Engineer |
| Module 1: Core Infrastructure | 4 | 16 | Backend Engineer |
| Module 2: Auth & Security | 5 | 17 | Backend Engineer |
| Module 3: Bet Ingestion | 4 | 18 | Backend Engineer |
| Module 4: Price Feed | 5 | 14 | Backend Engineer |
| Module 5: Resolution & Archival | 4 | 14 | Backend Engineer |
| Phase 4: System Hardening | 5 | 20 | Backend Engineer |
| **Total** | **40** | **131** | |

---

## Key Implementation Notes

### ADR-009 + ADR-010 Coordination
The bet batcher must implement both conditional deduction AND hybrid ledger in the **same transaction**:

```sql
BEGIN;
  -- ADR-009: Conditional deduction
  UPDATE users SET balance = balance - ? WHERE id = ? AND balance >= ? RETURNING balance;
  
  -- ADR-010: Ledger entry
  INSERT INTO transaction_ledger (user_id, amount, balance_after, ref_type, ref_id) VALUES (...);
  
  -- Bet record
  INSERT INTO bets (...) VALUES (...);
COMMIT;
```

### Configuration Values (from Hardening)

| Parameter | Value | ADR |
|-----------|-------|-----|
| Nonce TTL | 90 seconds | ADR-007 (hardened) |
| Circuit breaker enter | 10s stale | ADR-008 |
| Circuit breaker recovery | 30s hysteresis | ADR-008 (hardened) |
| Batch size | 100 bets | ADR-005 |
| Batch timeout | 100ms | ADR-005 |
| Redis maxmemory | 512MB | ADR-004 |

### CI/CD Quality Gates

Per the architecture-to-implementation guidelines, all changes must pass:

| Gate | Checks | Failure Action |
|------|--------|----------------|
| **Build Gate** | `gofmt`, `go vet`, `golangci-lint`, `gosec` on critical paths | Block merge |
| **Test Gate** | Unit tests with race detector, 80% coverage on `pkg/batcher` and `pkg/auth` | Block merge |
| **Artifact Versioning** | Semantic version (v1.2.3-build+sha), version endpoint | Block deploy |

### Testing Requirements

Each module must include:
- Unit tests for domain logic
- Integration tests against real dependencies (SQLite, Redis)
- Module exit criteria verification

Phase 4 includes:
- End-to-end workflow tests
- 3K RPS load test
- Failure mode tests
- Security validation (replay, double-spend)

---

## Deliverables by Phase

### Phase 1
- [ ] Infrastructure provisioned
- [ ] Database configured (WAL mode)
- [ ] Redis running with persistence
- [ ] Go project builds successfully
- [ ] Nginx serving static files
- [ ] **CI/CD pipeline with build gates, test gates, and artifact versioning**

### Phase 2
- [ ] All containers deployable via systemd
- [ ] Health endpoints responding
- [ ] Connectivity verified end-to-end

### Phase 3
- [ ] 5 modules with tests passing
- [ ] Auth flow functional
- [ ] Bet placement with batching
- [ ] Price feed streaming
- [ ] Resolution worker running

### Phase 4
- [ ] 3K RPS sustained
- [ ] p99 latency < 100ms
- [ ] All failure modes tested
- [ ] Metrics and alerts operational
- [ ] Runbooks documented

---

## Risk Mitigation

| Risk | Mitigation |
|------|------------|
| SQLite contention at 3K RPS | Write batching (ADR-005), verified in Phase 4 |
| Double-spend race condition | Conditional SQL update (ADR-009), tested in Module 3 |
| Redis OOM | 90s nonce TTL (hardened), 512MB limit with LRU |
| Disk exhaustion | Weekly archival (ADR-006), integrity check |
| Stale price exploitation | Circuit breaker (ADR-008), 30s hysteresis |

---

## Navigation

- [Overview](./00-overview.md)
- [Phase 1: Foundation](./01-phase-foundation.md)
- [Phase 2: Skeleton](./02-phase-skeleton.md)
- [Module 1: Core Infrastructure](./03-module-core-infrastructure.md)
- [Module 2: Auth & Security](./04-module-auth-security.md)
- [Module 3: Bet Ingestion](./05-module-bet-ingestion.md)
- [Module 4: Price Feed](./06-module-price-feed.md)
- [Module 5: Resolution & Archival](./07-module-resolution-archival.md)
- [Phase 4: System Hardening](./08-phase-hardening.md)
