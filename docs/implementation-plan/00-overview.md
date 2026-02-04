# Implementation Plan: BTC Tap Trading Backend

## Project Context

| Attribute | Value |
|-----------|-------|
| **Architecture Version** | Hardened v1.3 |
| **Timeline** | 2 weeks |
| **Team** | 1 Senior Backend Engineer |
| **Target** | 3,000 RPS, ≤100ms latency, 99.5% uptime |
| **Infrastructure** | Single VM (4c/8GB/256GB SSD) |

## Implementation Strategy

This plan follows the **hybrid horizontal-vertical execution model**:

1. **Phase 1: Foundation** — Establish infrastructure and shared components
2. **Phase 2: Skeleton** — Deploy minimal versions of all containers with connectivity
3. **Phase 3: Vertical Modules** — Implement module-by-module with full depth
4. **Phase 4: System Hardening** — End-to-end validation and load testing

## Active ADRs (Source of Truth)

All 10 ADRs are `Accepted` and must be implemented:

| ADR | Title | Implementation In |
|-----|-------|-------------------|
| 001 | SQLite as Primary Database | Phase 1, 3 |
| 002 | Go as Backend Language | Phase 1 |
| 003 | Monolithic Single-VM Deployment | Phase 1, 4 |
| 004 | Redis for Cache and Pub/Sub | Phase 1, 3 |
| 005 | Write Batching for Bet Ingestion | Phase 3 (Bet Ingestion Module) |
| 006 | Archival Strategy for Bet History | Phase 3 (Archival Module) |
| 007 | Replay Protection for Web3 Auth | Phase 3 (Auth Module) |
| 008 | Stale Price Circuit Breaker | Phase 3 (Price Feed Module) |
| 009 | Conditional Balance Deduction | Phase 3 (Bet Ingestion Module) |
| 010 | Hybrid Transaction Ledger | Phase 3 (Bet Ingestion Module) |

> **Note:** ADR-010 extends ADR-009 — both must be implemented together in the same transaction.

## C4 Container Inventory

All containers from the C4 diagram must be provisioned:

| Container | Technology | Phase |
|-----------|------------|-------|
| API Server | Go (Echo/Fiber) | Phase 2, 3 |
| Bet Batcher | Go (channel-based) | Phase 3 |
| Signature Validator | Go | Phase 3 |
| Circuit Breaker | Go middleware | Phase 3 |
| WebSocket Server | Go | Phase 2, 3 |
| Session Manager | Go | Phase 3 |
| Price Ingestion | Go | Phase 3 |
| Price Health Monitor | Go | Phase 3 |
| Bet Resolver | Go | Phase 3 |
| Weekly Archiver | Go + bash | Phase 3 |
| SQLite (Current Week) | SQLite 3 (WAL) | Phase 1 |
| SQLite (Historical) | SQLite 3 (Read-Only) | Phase 3 |
| Redis | Redis 7 | Phase 1 |
| Nginx | Nginx | Phase 1, 2 |

## Module Breakdown (Phase 3 Vertical Slices)

The backend is divided into 5 vertical modules for implementation:

1. **[Module 1: Core Infrastructure](./01-module-core-infrastructure.md)** — Database, Redis, shared libraries
2. **[Module 2: Authentication & Security](./02-module-auth-security.md)** — Web3 auth, replay protection, sessions
3. **[Module 3: Bet Ingestion](./03-module-bet-ingestion.md)** — Write batching, conditional deduction, ledger
4. **[Module 4: Price Feed](./04-module-price-feed.md)** — Ingestion, circuit breaker, WebSocket streaming
5. **[Module 5: Resolution & Archival](./05-module-resolution-archival.md)** — Bet resolution, weekly archival, integrity checks

## Traceability Matrix

| ADR | Phase 1 | Phase 2 | Phase 3 Module | Phase 4 |
|-----|---------|---------|----------------|---------|
| 001 | ✅ DB setup | ✅ | All modules | ✅ |
| 002 | ✅ Go scaffold | ✅ | All modules | ✅ |
| 003 | ✅ VM setup | ✅ All containers | All modules | ✅ |
| 004 | ✅ Redis setup | ✅ | All modules | ✅ |
| 005 | | | Module 3 | ✅ |
| 006 | | | Module 5 | ✅ |
| 007 | | | Module 2 | ✅ |
| 008 | | | Module 4 | ✅ |
| 009 | | | Module 3 | ✅ |
| 010 | | | Module 3 | ✅ |

## Deliverables

| Phase | Output |
|-------|--------|
| Phase 1 | Infrastructure ready, CI/CD running, shared libraries published |
| Phase 2 | All containers deployable, health endpoints responding, connectivity verified |
| Phase 3 | 5 modules implemented with tests, integration passing |
| Phase 4 | 3K RPS sustained, 100ms p99 latency, all failure modes tested |

## Success Criteria

- [ ] All ADRs implemented per specification
- [ ] C4 container diagram fully realized
- [ ] 3,000 RPS sustained load test passing
- [ ] p99 latency ≤100ms
- [ ] Circuit breaker functional (verified by stale feed simulation)
- [ ] Replay attack prevention verified
- [ ] Double-spend prevention verified
- [ ] Weekly archival functional
- [ ] Transaction ledger integrity check passing
