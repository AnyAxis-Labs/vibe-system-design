# Architecture Hardening Audit: BTC Tap Trading

**Date:** 2026-02-03
**Reviewer:** System Architecture Guardian (Hardening Skill)
**Status:** CHALLENGING DESIGN

## Executive Summary
The current design (Single VM, SQLite, Go) is theoretically capable of meeting the 3,000 RPS target *if* the workload is read-heavy. However, the "Tap" nature implies a high write urgency. The primary risks are **SQLite write contention** at peak load and **disk space exhaustion** due to the "Forever" retention policy on a small SSD.

---

## 🔨 Risk Analysis & Critiques

### 1. Data & State Management (High Risk)

**Critical Vulnerability: SQLite Write Contention at 3k RPS**
- **Context:** SQLite allows only *one* writer at a time, even in WAL mode.
- **Attack Vector:** If 3,000 users "tap" simultaneously (3k writes/sec), distinct transactions will queue. If a transaction takes > 0.3ms (very easy with disk I/O, strict durability, or potential index updates), the queue backs up, latency spikes > 100ms (NFR violation), and the app may crash with `SQLITE_BUSY`.
- **NFR Impact:** Latency (p99), Availability.
- **Suggested Hardening:**
    - **Batching:** The API Server *must* buffer taps in memory (e.g., channel) and flush to SQLite in batches (100-500ms windows or 50-item batches). This reduces transaction overhead drastically.
    - **Optimistic UI:** Client assumes success; Server responds async via WebSocket if failed (though this conflicts with "No confirmation dialogs").
    - **Stress Test Request:** Must verify `fsync` time on the target SSD.

**Critical Vulnerability: "Forever" Retention vs. 256GB SSD**
- **Context:** PRD requires "Bet history retained indefinitely".
- **Attack Vector:**
    - Assumed object size: 200 bytes per bet (UUID, UserID, Timestamp, Price, Amount, Status, Metadata).
    - Traffic: 3,000 RPS * 20% write ratio (guess) = 600 bets/sec.
    - Daily Volume: 600 * 86400 * 200 bytes ≈ 10 GB/day.
    - **Time to Death:** ~25 days. The system will fail before the first month is over.
- **NFR Impact:** Availability (Disk Full = Outage).
- **Suggested Hardening:**
    - **Pruning Policy:** Challenge the "Forever" requirement on the hot node.
    - **Cold Storage:** Implement daily `rsync` of older SQLite partitions or CSV exports to S3 (or external storage), then `DELETE` from main DB. SQLite `VACUUM` will be needed (which blocks operation!).
    - **Solution:** Use **auto-vacuum = incremental** or partition DBs by week.

### 2. Failure, Resilience & Degradation (Medium Risk)

**Vulnerability: Betting vs. Resolution Contention**
- **Context:** A "Bet Resolver" worker runs on the same VM/DB.
- **Attack Vector:** Resolution likely requires a "Range Scan" (Find all bets in Time Column X). Using the same SQLite file for aggressive reads (Resolution) + aggressive writes (Taps) creates lock contention.
- **NFR Impact:** Latency.
- **Suggested Hardening:**
    - **Read Replica:** Can we use a Read-Only connection for the Resolver? (SQLite WAL supports this well).
    - **Index Strategy:** Ensure covering index on `(target_time_column, min_price, max_price)` to avoid table scans during resolution.

**Vulnerability: Thundering Herd on WS Reconnect**
- **Context:** Single VM means deployments or crashes drop 100% of WebSocket connections.
- **Attack Vector:** 10,000 users reconnect simultaneously. Go server handles connections, but the validation logic (Session check in Redis) might spike Redis CPU.
- **Suggested Hardening:**
    - **Jitter:** Client *must* implement randomized exponential backoff (0-5s).
    - **Shedding:** Nginx must limit total concurrent connections if exceeding 4 core capacity.

### 3. Security & Abuse (Medium Risk)

**Vulnerability: Replay Attacks on Web3 Auth**
- **Context:** No password, signature only.
- **Attack Vector:** Attacker captures a valid "Place Bet" payload + signature and replays it 1000 times to drain user funds or grief the system.
- **Suggested Hardening:**
    - **Nonce/Timestamp:** Every signed message *must* include a timestamp (< 30s diff) and a nonce, tracked in Redis (with TTL) to ensure uniqueness.

### 4. Observability (The 3 A.M. Test)

**Missing: "Stale Price" Circuit Breaker**
- **What happens if the price feed stops?**
    - Do users keep betting on old prices?
    - Do resolutions happen with old data?
- **Suggested Hardening:** Application *must* assume UNHEALTHY if no price tick received in 2x expected interval (e.g., 2 seconds). All betting suspended automatically.

## Summary of Required Changes for Acceptance
1.  **Mandatory:** Implement Write Batching for Bets. Single-row inserts will not survive 3k RPS.
2.  **Mandatory:** Define an Archival Strategy immediately (256GB is insufficient).
3.  **Mandatory:** Add Replay Protection (Nonce/Timestamp) to signature payloads.
4.  **Mandatory:** "Stale Price" Circuit Breaker.
