# Implementation Plan Traceability Matrix

This document maps every Architectural Decision Record (ADR) and C4 Container to specific implementation tasks.

---

## ADR to Implementation Mapping

### ADR-001: SQLite as Primary Database
| Implementation Task | Phase | Day | File |
|---------------------|-------|-----|------|
| VM provision with SSD mount | 1 | 1.1 | `phase-1-foundation.md` |
| SQLite installation & WAL config | 1 | 1.4 | `phase-1-foundation.md` |
| Database schema with PRAGMAs | 1 | 2.2 | `phase-1-foundation.md` |
| Connection pool management | 1 | 3.1 | `phase-1-foundation.md` |
| Backup scripts (hourly) | 4 | 10.3 | `phase-4-hardening.md` |

---

### ADR-002: Go as Backend Language
| Implementation Task | Phase | Day | File |
|---------------------|-------|-----|------|
| Project structure scaffold | 1 | 2.1 | `phase-1-foundation.md` |
| Module initialization | 1 | 2.1 | `phase-1-foundation.md` |
| Makefile with build targets | 1 | 2.1 | `phase-1-foundation.md` |
| Binary builds (api, ws, workers) | 1 | 2.1 | `phase-1-foundation.md` |

---

### ADR-003: Monolithic Single-VM Deployment
| Implementation Task | Phase | Day | File |
|---------------------|-------|-----|------|
| VM security hardening | 1 | 1.1 | `phase-1-foundation.md` |
| Systemd service files (5 services) | 1 | 3.2 | `phase-1-foundation.md` |
| Directory structure (/opt/app, /data) | 1 | 1.1 | `phase-1-foundation.md` |
| Auto-restart configuration | 1 | 3.2 | `phase-1-foundation.md` |

---

### ADR-004: Redis for Cache and Pub/Sub
| Implementation Task | Phase | Day | File |
|---------------------|-------|-----|------|
| Redis installation & config | 1 | 1.3 | `phase-1-foundation.md` |
| AOF + RDB persistence setup | 1 | 1.3 | `phase-1-foundation.md` |
| Memory policy (allkeys-lru) | 1 | 1.3 | `phase-1-foundation.md` |
| Connection pooling | 1 | 3.1 | `phase-1-foundation.md` |
| Pub/sub implementation | 2 | 4.4 | `phase-2-skeleton.md` |

---

### ADR-005: Write Batching for Bet Ingestion
| Implementation Task | Phase | Day | File |
|---------------------|-------|-----|------|
| BetBatcher component scaffold | 2 | 4.3 | `phase-2-skeleton.md` |
| Channel-based buffering (10K) | 2 | 4.3 | `phase-2-skeleton.md` |
| Batch flush logic (100 items/100ms) | 2 | 4.3 | `phase-2-skeleton.md` |
| Done channel for async response | 2 | 4.3 | `phase-2-skeleton.md` |
| Single writer goroutine | 2 | 4.3 | `phase-2-skeleton.md` |
| Complete transaction logic | 3 | 6.1 | `phase-3-core-logic.md` |

---

### ADR-006: Archival Strategy for Bet History
| Implementation Task | Phase | Day | File |
|---------------------|-------|-----|------|
| Weekly archiver script | 3 | 8.1 | `phase-3-core-logic.md` |
| VACUUM INTO implementation | 3 | 8.1 | `phase-3-core-logic.md` |
| zstd compression | 3 | 8.1 | `phase-3-core-logic.md` |
| S3 STANDARD_IA upload | 3 | 8.1 | `phase-3-core-logic.md` |
| Dead man's switch metric | 3 | 8.1 | `phase-3-core-logic.md` |
| Local cleanup (4 week retention) | 3 | 8.1 | `phase-3-core-logic.md` |
| Read replica attachment | 3 | 8.3 | `phase-3-core-logic.md` |

---

### ADR-007: Replay Protection for Web3 Auth
| Implementation Task | Phase | Day | File |
|---------------------|-------|-----|------|
| Frontend Web3 connect | 1 | 3.4 | `phase-1-foundation.md` |
| EIP-191 signing (ethers.js) | 1 | 3.4 | `phase-1-foundation.md` |
| Signature validator (Go) | 2 | 5.1 | `phase-2-skeleton.md` |
| Address recovery (go-ethereum) | 2 | 5.1 | `phase-2-skeleton.md` |
| Timestamp validation (30s window) | 2 | 5.1 | `phase-2-skeleton.md` |
| Nonce tracking (Redis SET NX 90s) | 2 | 5.1 | `phase-2-skeleton.md` |
| Frontend signature flow | 2 | 5.3 | `phase-2-skeleton.md` |

---

### ADR-008: Stale Price Circuit Breaker
| Implementation Task | Phase | Day | File |
|---------------------|-------|-----|------|
| Price ingestion service | 2 | 4.4 | `phase-2-skeleton.md` |
| Health monitor component | 2 | 5.2 | `phase-2-skeleton.md` |
| Status tracking (HEALTHY/DEGRADED/CRITICAL) | 2 | 5.2 | `phase-2-skeleton.md` |
| Thresholds: 3s/10s | 2 | 5.2 | `phase-2-skeleton.md` |
| Hysteresis (30s stable for recovery) | 2 | 5.2 | `phase-2-skeleton.md` |
| Middleware for bet rejection | 2 | 5.2 | `phase-2-skeleton.md` |
| Frontend status display | 3 | 9.3 | `phase-3-core-logic.md` |

---

### ADR-009: Conditional Balance Deduction
| Implementation Task | Phase | Day | File |
|---------------------|-------|-----|------|
| Users table schema | 1 | 2.2 | `phase-1-foundation.md` |
| Conditional UPDATE with RETURNING | 3 | 6.1 | `phase-3-core-logic.md` |
| RowsAffected check | 3 | 6.1 | `phase-3-core-logic.md` |
| Insufficient funds handling | 3 | 6.1 | `phase-3-core-logic.md` |
| Transaction atomicity | 3 | 6.1 | `phase-3-core-logic.md` |
| Redis cache sync (best-effort) | 3 | 6.1 | `phase-3-core-logic.md` |
| Startup cache rebuild | 4 | 10.6 | `phase-4-hardening.md` |

---

### ADR-010: Hybrid Transaction Ledger
| Implementation Task | Phase | Day | File |
|---------------------|-------|-----|------|
| transaction_ledger table schema | 1 | 2.2 | `phase-1-foundation.md` |
| Indexes (user+time, ref) | 1 | 2.2 | `phase-1-foundation.md` |
| Ledger write in batch transaction | 3 | 6.1 | `phase-3-core-logic.md` |
| balance_after column | 3 | 6.1 | `phase-3-core-logic.md` |
| Payout ledger entries | 3 | 7.2 | `phase-3-core-logic.md` |
| Daily integrity check | 3 | 8.2 | `phase-3-core-logic.md` |
| Ledger archival with bets | 3 | 8.1 | `phase-3-core-logic.md` |
| Dispute query support | 3 | 9.1 | `phase-3-core-logic.md` |

---

## C4 Container to Implementation Mapping

### Containers (Backend)

| Container | Implementation Tasks | Total Hours |
|-----------|---------------------|-------------|
| **Nginx** | Install, reverse proxy, TLS, rate limiting | 6 |
| **API Server** | Scaffold, handlers, auth, bet API, health | 12 |
| **Bet Batcher** | Channel, batch logic, transactions | 10 |
| **Signature Validator** | EIP-191, nonce check, timestamp | 5 |
| **Circuit Breaker** | Health monitor, middleware, hysteresis | 5 |
| **WebSocket Server** | Connection mgmt, broadcasting, auth | 8 |
| **Price Ingestion** | Binance WS, Redis pub/sub, reconnect | 6 |
| **Price Health Monitor** | Status tracking, CB integration | 4 |
| **Bet Resolver** | Resolution algo, payout logic, notifications | 8 |
| **Weekly Archiver** | VACUUM, zstd, S3, cleanup | 6 |
| **SQLite (RW)** | Schema, migrations, WAL, indexes | 8 |
| **SQLite (RO)** | Attachment, query routing | 4 |
| **Redis** | Install, config, persistence, pub/sub | 6 |

**Backend Total: ~98 hours (12.25 days)** → Parallelize with FE, cut scope to fit 2 weeks

### Containers (Frontend)

| Container | Implementation Tasks | Total Hours |
|-----------|---------------------|-------------|
| **Frontend SPA** | Scaffold, components, state, build | 8 |
| **Grid UI** | 2D grid, cell logic, reward rates | 6 |
| **Wallet Integration** | Web3 connect, signing | 4 |
| **Bet Flow** | Pending/Confirmed states, WebSocket | 8 |
| **History View** | Balance, bet history, pagination | 5 |
| **Admin Dashboard** | Health, metrics, manual triggers | 4 |
| **Reconnect Logic** | Jitter, backoff, state recovery | 4 |

**Frontend Total: ~39 hours (5 days)**

---

## Critical Path Dependencies

```
[VM] → [Nginx] → [Redis] → [SQLite]
  ↓       ↓         ↓         ↓
[Go] → [API] → [Batcher] → [DB Schema]
  ↓       ↓         ↓           ↓
[FE] → [Grid] → [Wallet] → [Bet Flow]
                    ↓           ↓
              [Signature] → [E2E Test]
```

---

## Risk Mitigation Tasks

| Risk | Mitigation Task | Phase | Day |
|------|----------------|-------|-----|
| SQLite contention | Early batcher test | 2 | 4.3 |
| Disk full | Archiver + monitoring | 3 | 8.1, 10.2 |
| Data loss | Backup + recovery test | 4 | 10.3 |
| Replay attacks | Signature validation | 2 | 5.1 |
| Double-spend | Conditional UPDATE | 3 | 6.1 |
| Silent archiver | Dead man's switch | 3 | 8.1 |
| Feed failure | Circuit breaker | 2 | 5.2 |

---

## Verification Checklist

### Per ADR

- [ ] ADR-001: WAL mode enabled, backups working
- [ ] ADR-002: All binaries build, tests pass
- [ ] ADR-003: Services start with systemd, auto-restart works
- [ ] ADR-004: Redis pub/sub functional, persistence verified
- [ ] ADR-005: 3K RPS with <100ms latency
- [ ] ADR-006: Weekly archive to S3, dead man's switch alerting
- [ ] ADR-007: Replay attacks blocked, nonces expire in 90s
- [ ] ADR-008: CB enters CRITICAL at >10s, recovers after 30s stable
- [ ] ADR-009: No double-spend possible, conditional UPDATE works
- [ ] ADR-010: Ledger entries for all transactions, integrity check passes

### Per C4 Container

- [ ] Nginx: Rate limiting tested, TLS working
- [ ] API Server: All endpoints functional, validated
- [ ] Bet Batcher: Batches correctly, no contention
- [ ] WebSocket: Real-time prices, bet confirmations
- [ ] Price Ingestion: Reconnects on failure
- [ ] Bet Resolver: Correct win/loss detection
- [ ] Archiver: Automated weekly, verified on S3
- [ ] Frontend: Grid interactive, states correct
