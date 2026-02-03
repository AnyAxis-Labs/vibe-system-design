# Implementation Plan: BTC Tap Trading Application

**Architecture Version:** Hardened v1.3  
**Timeline:** 2 Weeks (10 working days)  
**Team:** 1 Senior Backend Engineer, 1 Senior Frontend Engineer

---

## Overview

This implementation plan translates the Hardened v1.3 architecture into actionable work items. Each task includes:
- **Traceability:** Reference to source ADR or C4 Container
- **Dependencies:** Prerequisites that must be completed first
- **Definition of Done (DoD):** Clear acceptance criteria
- **Owner:** Backend (BE) or Frontend (FE)

---

## Implementation Phases

| Phase | Duration | Focus | Output |
|-------|----------|-------|--------|
| [Phase 1: Foundation](./phase-1-foundation.md) | Days 1-3 | Infrastructure, CI/CD, database schema | Deployable skeleton |
| [Phase 2: Skeleton](./phase-2-skeleton.md) | Days 4-5 | Thin-thread connectivity, auth, health checks | End-to-end "hello world" |
| [Phase 3: Core Logic](./phase-3-core-logic.md) | Days 6-9 | Betting, resolution, archival | Feature-complete system |
| [Phase 4: Hardening](./phase-4-hardening.md) | Day 10 | Testing, monitoring, documentation | Production-ready |

---

## Quick Reference: ADR to Implementation Mapping

| ADR | Key Implementation Tasks |
|-----|-------------------------|
| ADR-001 SQLite | Database schema, WAL config, backup scripts |
| ADR-002 Go | Project scaffolding, module structure |
| ADR-003 Single-VM | Systemd service files, deployment scripts |
| ADR-004 Redis | Redis config, connection pooling, pub/sub |
| ADR-005 Batching | BetBatcher component, channel logic |
| ADR-006 Archival | Weekly archiver script, S3 integration |
| ADR-007 Replay Protection | Signature validator, nonce middleware |
| ADR-008 Circuit Breaker | Health monitor, CB middleware |
| ADR-009 Conditional Deduction | SQL UPDATE with RETURNING, transaction mgmt |
| ADR-010 Hybrid Ledger | Ledger table, audit queries, integrity check |

---

## Critical Path

```
Day 1: VM Provision → Nginx → Redis → SQLite Schema
Day 2: Go Project Scaffold → API Skeleton → DB Migrations
Day 3: Web3 Auth → Signature Validator → Session Manager
Day 4: Bet Batcher → Ledger Write → WebSocket Server
Day 5: Price Ingestion → Health Monitor → Circuit Breaker
Day 6: Grid UI → Bet Placement Flow
Day 7: Bet Resolver → Resolution Logic
Day 8: Weekly Archiver → S3 Upload → Integrity Checks
Day 9: Frontend Hardening → Pending States → Reconnect Jitter
Day 10: Load Testing → Monitoring → Documentation
```

---

## Risk Mitigation

| Risk | Mitigation |
|------|------------|
| SQLite performance at 3K RPS | Test batching early (Day 4), tune batch size |
| Web3 integration complexity | Use tested libraries (go-ethereum, ethers.js) |
| 2-week deadline | Parallel FE/BE work, thin-thread first |
| Single point of failure | Document restore procedures, test backups |

---

## Definition of Ready (DoR) per Task

- [ ] Clear acceptance criteria
- [ ] No unresolved dependencies
- [ ] ADR reference included
- [ ] Estimated effort (hours)

## Definition of Done (DoD) per Task

- [ ] Code implemented
- [ ] Unit tests pass (>80% coverage)
- [ ] Integration tested
- [ ] Documented (README/comments)
- [ ] Code reviewed
