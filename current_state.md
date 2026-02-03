# Current System State

## Project: BTC Tap Trading Application (HARDENED v1.3)

### Architecture Overview

**Deployment Model**: Monolithic single-VM deployment with hardening  
**VM Specs**: 4-core CPU @ 2.5 GHz, 8 GB RAM, 256 GB SSD  
**SLA Target**: 99.5% uptime (scheduled maintenance acceptable)  
**Peak Load**: 3,000 RPS (handled via write batching)  
**Latency Target**: ≤100ms end-to-end (optimistic + batched)

### Hardening Summary

#### Round 001 (Catastrophic Risks)
| Risk | Mitigation | ADR |
|------|------------|-----|
| SQLite write contention at 3K RPS | In-memory write batching | ADR-005 |
| Disk exhaustion (25 days to death) | Weekly partitioning + S3 archival | ADR-006 |
| Replay attacks on Web3 auth | Timestamp + Nonce validation | ADR-007 |
| Stale price feed | Automatic circuit breaker | ADR-008 |

#### Round 002 (Operational Risks)
| Risk | Mitigation | ADR/Note |
|------|------------|----------|
| Double-spend race condition | Conditional SQL balance deduction | ADR-009 |
| Redis nonce memory bloat | TTL: 24h → 90s (config change) | Config |
| Archival silent failure | Dead man's switch metric | Ops |
| Circuit breaker flapping | 30s hysteresis on recovery | Config |
| Optimistic UI trust gap | "Pending" state until WS confirm | UX Spec |
| Read-write contention | SQLite read replica for resolution | Diagram update |
| Thundering herd on reconnect | Client jitter + nginx conn limits | Diagram update |

#### Round 003 (Financial Auditability)
| Risk | Mitigation | ADR |
|------|------------|-----|
| Mutable balance (no audit trail) | Hybrid Transaction Ledger | ADR-010 |
| Dispute resolution | Query ledger by user/ref/time | ADR-010 |
| Data corruption undetected | Daily integrity check (SUM validation) | ADR-010 |

### Technology Stack

| Layer | Technology | Version |
|-------|------------|---------|
| Frontend | React or Vue | Latest LTS |
| API Server | Go (Echo/Fiber) | 1.21+ |
| Bet Batcher | Go (channel-based) | - |
| Signature Validator | Go | - |
| Circuit Breaker | Go middleware | - |
| WebSocket Server | Go | 1.21+ |
| Price Health Monitor | Go | 1.21+ |
| Bet Resolution Worker | Go | 1.21+ |
| Weekly Archiver | Go + bash + zstd | - |
| Primary Database | SQLite (bets + ledger, WAL) | 3.x |
| Historical Database | SQLite (bets + ledger, read-only) | 3.x |
| Cache/PubSub | Redis | 7.x |
| Reverse Proxy | Nginx | Latest |
| Cold Storage | AWS S3 (STANDARD_IA) | - |
| Process Supervision | systemd | - |

### Data Retention

| Data Type | Retention | Storage |
|-----------|-----------|---------|
| Current week bets | Current week | SQLite (hot, local) |
| Recent history | 4 weeks | SQLite (read-only, local) |
| Historical bets | Forever | S3 STANDARD_IA |
| Market price data | 7 days | Redis (TTL) |
| User sessions | 24 hours | Redis (TTL) |
| Used nonces | 90 seconds | Redis (TTL) |

### Storage Budget (256GB SSD)

| Component | Size | Notes |
|-----------|------|-------|
| SQLite current week | ~18 GB | Bets + transaction_ledger |
| SQLite historical (4 weeks) | ~72 GB | Queryable locally |
| Redis (maxmemory) | ~512 MB | Configured |
| Application binaries/logs | ~5 GB | Generous estimate |
| **Total Used** | **~95 GB** | **~160 GB headroom** |

### Services Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Single VM (4c/8GB/256GB)                         │
│                                                                          │
│  ┌─────────────┐  ┌─────────────────────────────────────────────────┐   │
│  │   Nginx     │  │           Application Layer (Go binaries)        │   │
│  │   :80/443   │  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌────────┐ │   │
│  │             │  │  │  API    │ │   WS    │ │  Price  │ │  Arch  │ │   │
│  │  ┌─────┐    │  │  │ Server  │ │ Server  │ │ Ingest  │ │  iver  │ │   │
│  │  │static│◄───┼──┼──┤:8080   │ │  :8081  │ │         │ │        │ │   │
│  │  │files │    │  │  │         │ │         │ │         │ │        │ │   │
│  │  └─────┘    │  │  │┌───────┐│ │┌───────┐│ │┌───────┐│ │┌──────┐│ │   │
│  │             │  │  ││Batcher││ ││Session││ ││Health ││ ││Weekly││ │   │
│  │  rate limit │  │  ││Channel││ ││Manager││ ││Monitor││ ││S3    ││ │   │
│  │  conn limit │  │  │└───┬────┘│ │└───────┘│ │└───┬───┘│ │└──┬───┘│ │   │
│  └─────────────┘  │  └────┼─────┘ └─────────┘ └────┼────┘ └──┼────┘ │   │
│                   │       │                        │         │      │   │
│  ┌──────────────────────────────────────────────────────────────────┐ │   │
│  │                         Data Layer                                │ │   │
│  │  ┌────────────────┐  ┌────────────────┐  ┌──────────────────┐    │ │   │
│  │  │  SQLite RW     │  │  SQLite RO     │  │  Redis 7         │    │ │   │
│  │  │  (current week)│  │  (historical)  │  │  - Sessions      │    │ │   │
│  │  │  bets_2025_w06 │  │  bets_2025_w*  │  │  - Nonces        │    │ │   │
│  │  │  WAL mode      │  │  Attached read │  │  - Prices        │    │ │   │
│  │  └────────────────┘  └────────────────┘  │  - Pub/Sub       │    │ │   │
│  │                                          └──────────────────┘    │ │   │
│  └──────────────────────────────────────────────────────────────────┘ │   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
                         ┌─────────────────────┐
                         │   S3 Cold Storage   │
                         │   STANDARD_IA       │
                         │   bets_*.db.zst     │
                         └─────────────────────┘
```

### Write Batching Flow

```
┌──────────────┐      ┌──────────────┐      ┌──────────────┐
│  HTTP Handler │─────▶│ Bet Channel  │─────▶│   Batcher    │
│  (Goroutine)  │ 202  │ (10K buffer) │      │  (1 writer)  │
└──────────────┘      └──────────────┘      └──────┬───────┘
     ▲                                             │
     │ WebSocket confirm                           │ Batch insert
     │                                             ▼
     └──────────────────────────────────────┌──────────────┐
                                             │ SQLite (WAL) │
                                             └──────────────┘
```

### Key Architectural Decisions

| ADR | Decision | Status | Purpose |
|-----|----------|--------|---------|
| 001 | SQLite as Primary Database | Accepted | Zero ops overhead |
| 002 | Go as Backend Language | Accepted | Concurrency + type safety |
| 003 | Monolithic Single-VM Deployment | Accepted | Team velocity |
| 004 | Redis for Cache and Pub/Sub | Accepted | Speed + decoupling |
| 005 | Write Batching for Bet Ingestion | Accepted (Hardening) | Survive 3K RPS |
| 006 | Archival Strategy for Bet History | Accepted (Hardening) | Infinite retention on 256GB |
| 007 | Replay Protection for Web3 Auth | Accepted (Hardening) | Security |
| 008 | Stale Price Circuit Breaker | Accepted (Hardening) | Fairness |
| 009 | Conditional Balance Deduction | Accepted (Hardening 002) | Double-spend prevention |
| 010 | Hybrid Transaction Ledger | Accepted (Hardening 003) | Financial auditability |

### Constraints & Limits

- **No horizontal scaling** (by design, for simplicity)
- **Single-writer SQLite** (batched to mitigate)
- **Scheduled maintenance** acceptable per SLA
- **Weekly archival required** for sustainability

### Security Controls

| Control | Implementation |
|---------|----------------|
| Replay protection | Timestamp (30s window) + Nonce (Redis SET NX, 90s TTL) |
| Signature validation | EIP-191 personal_sign recovery |
| Rate limiting | Redis-based per wallet |
| Input validation | Server-side constraint enforcement |
| Double-spend prevention | Conditional SQL UPDATE: `balance >= amount` |
| Audit trail | Hybrid ledger: `transaction_ledger` table (append-only) |
| Dispute resolution | Query ledger by user_id, ref_type, ref_id, time |

### Observability

| Metric | Source |
|--------|--------|
| Request rate/latency | API Server Prometheus |
| Error rate | Application logs (structured JSON) |
| Bet resolution outcomes | Worker metrics |
| Price feed health | Health Monitor status |
| Disk usage | Node exporter |
| Circuit breaker state | Health endpoint |
| Archive health | `last_successful_archive_timestamp` gauge |

### Accepted Risks (Post-Hardening)

1. **VM failure** = total outage (mitigated by hourly backups to S3)
2. **In-memory batch loss** = max 100ms of unconfirmed bets (acceptable for trading)
3. **S3 dependency** for historical queries (acceptable, async restore available)

### Deployment Checklist

- [ ] SQLite configured with WAL mode
- [ ] Redis configured with AOF + RDB persistence
- [ ] Nginx configured with rate limiting and connection limits
- [ ] S3 bucket created with STANDARD_IA lifecycle
- [ ] Weekly archive cron job installed
- [ ] Circuit breaker thresholds configured (enter: 10s, recover: 30s stable)
- [ ] Health endpoint monitoring configured
- [ ] Client-side reconnect jitter implemented
- [ ] **Nonce TTL set to 90 seconds** (not 24h)
- [ ] **Dead man's switch metric** `last_successful_archive_timestamp` alerting
- [ ] **UX**: Pending state until WebSocket confirmation
- [ ] **Startup**: Redis balance cache rebuild from SQLite
- [ ] **Database**: Create `transaction_ledger` table with indexes
- [ ] **Cron**: Daily integrity check (`users.balance == SUM(ledger.amount)`)
- [ ] **Archiver**: Include `transaction_ledger` in weekly partitions |

### Last Updated
2026-02-03 (Hardened v1.3)
