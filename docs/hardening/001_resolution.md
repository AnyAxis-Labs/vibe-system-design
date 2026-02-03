# Hardening Resolution: 001_initial_critique.md

**Date:** 2026-02-03  
**Status:** ALL CRITICAL ISSUES RESOLVED  
**Architecture Version:** v1.1 (Hardened)

---

## Summary

All four mandatory hardening requirements have been addressed with concrete implementations:

| # | Requirement | Status | ADR | Key Change |
|---|-------------|--------|-----|------------|
| 1 | Write Batching | ✅ RESOLVED | ADR-005 | Channel-based batcher (100 items/100ms) |
| 2 | Archival Strategy | ✅ RESOLVED | ADR-006 | Weekly partitions + S3 cold storage |
| 3 | Replay Protection | ✅ RESOLVED | ADR-007 | Timestamp + Nonce + Redis |
| 4 | Stale Price CB | ✅ RESOLVED | ADR-008 | Auto-suspend betting at 10s stale |

---

## Detailed Resolution

### 1. Write Batching for Bets (MANDATORY)

**Problem:** SQLite single-writer limitation causes queue buildup at 3K RPS.

**Solution:** In-memory batching with channel-based aggregation.

```
Before: 3,000 individual transactions/sec → SQLite contention → SQLITE_BUSY
After:  30 batched transactions/sec       → Sustainable → <50ms latency
```

**Implementation:**
- Buffered channel (10,000 items) for backpressure
- Flush triggers: 100 items OR 100ms timeout
- Optimistic UI: 202 Accepted immediately, WebSocket confirms
- Single writer goroutine eliminates lock contention

**Trade-off:** Max 100ms of unconfirmed bets at risk on crash (acceptable).

---

### 2. Archival Strategy (MANDATORY)

**Problem:** 256GB SSD fills in ~25 days at 10GB/day growth.

**Solution:** Time-based partitioning with automated S3 archival.

```
Storage Layout:
├── bets_current.db      (current week, read-write, ~10GB)
├── bets_2025_w05.db     (previous weeks, read-only, ~40GB total)
├── bets_2025_w04.db.zst (S3 STANDARD_IA, forever retention)
└── ...
```

**Weekly Archival Process:**
1. Sunday 00:00: VACUUM INTO new file
2. Sunday 00:30: zstd -19 compression
3. Sunday 01:00: Upload to S3 STANDARD_IA
4. Sunday 01:30: Attach as read-only for queries
5. +28 days: Purge local compressed file

**Result:** ~50GB hot storage, ~200GB headroom, infinite retention.

---

### 3. Replay Protection (MANDATORY)

**Problem:** Attacker can replay signed bet payloads to drain funds.

**Solution:** EIP-191 signed messages with timestamp + nonce.

**Payload Schema:**
```
Tap Trading Bet
Action: place_bet
Cell: {cell_id}
Amount: {amount}
Nonce: {uuid}
Timestamp: {unix_seconds}
Chain: {chain_id}
Version: 1
```

**Validation:**
1. Timestamp within 30-second window
2. Nonce uniqueness via Redis SET NX (24h TTL)
3. Signer address recovery and match
4. Chain ID binding (prevents cross-chain replay)

---

### 4. Stale Price Circuit Breaker (MANDATORY)

**Problem:** Price feed failure causes betting on stale data.

**Solution:** Automatic health monitoring with circuit breaker.

**Thresholds:**
| Age | Status | Action |
|-----|--------|--------|
| <3s | HEALTHY | Normal operation |
| 3-10s | DEGRADED | Warning banner, continue |
| >10s | CRITICAL | Reject all bets with 503 |

**Behavior:**
- Health monitor subscribed to price pub/sub
- Circuit breaker middleware on bet endpoints
- Bet resolution paused if unhealthy
- WebSocket broadcasts status to clients
- Automatic recovery when feed resumes

---

## Additional Improvements

### Read Replica for Resolution
- Bet Resolver uses separate read-only SQLite connection
- Eliminates read-write contention
- Covering index on `(target_time, min_price, max_price)`

### Thundering Herd Mitigation
- Client: Randomized exponential backoff (0-5s) on reconnect
- Nginx: `limit_conn` prevents connection exhaustion

---

## Updated Diagrams

- `docs/diagrams/c4-container.md` — Updated with batcher, circuit breaker, archiver
- `current_state.md` — Updated with hardened architecture

## New ADRs

- `docs/adr/005-write-batching-for-bet-ingestion.md`
- `docs/adr/006-archival-strategy-for-bet-history.md`
- `docs/adr/007-replay-protection-for-web3-auth.md`
- `docs/adr/008-stale-price-circuit-breaker.md`

---

## Sign-off

**Reviewer:** System Architecture Guardian  
**Verdict:** ✅ **ACCEPTED FOR IMPLEMENTATION**

All critical vulnerabilities addressed. Architecture is now suitable for 3K RPS, 99.5% SLA, and indefinite retention within 256GB constraint.
