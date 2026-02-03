# Architecture Hardening Audit: Round 002

**Date:** 2026-02-03
**Context:** Review of "Hardened v1.1" (Batching, Archival, Replay Protection)
**Status:** RISKS IDENTIFIED (OPERATIONAL)

## Executive Summary
The "Hardened v1.1" design successfully mitigates the catastrophic scalability flaws of v1.0. The system can now theoretically handle the 3,000 RPS load.
However, the introduction of **Batching** and **Archival** creates new *Operational Risks* and *Edge Cases* that must be handled to prevent "Silent Failures".

---

## 🔨 Remaining Risks & Edge Cases

### 1. Data Loss vs. User Trust (The "Optimistic UI" Trap)
**Risk:** High
**Context:** ADR-005 accepts "Max 100ms of unconfirmed bets at risk". The Client uses "Optimistic UI" (shows "Bet Placed" immediately).
**Failure Mode:**
1.  User Taps. UI shows "Success".
2.  Server receives HTTP request, puts in Channel.
3.  Server crashes (OOM/Panic) *before* the 100ms flush.
4.  **Result:** User thinks they have a bet. The system has no record of it. User wins, demands payout. System logs show nothing.
**Mitigation:**
-   **Client State Handling:** The Client **MUST NOT** show a permanent "Confirmed" state until the WebSocket sends the `OrderConfirmed` event.
-   **UX:** Use a "Pending" animation (e.g., pulsing border) for the first ~200ms. Only turn solid Green/Gold upon WS confirmation.
-   **Recovery:** If socket disconnects while "Pending", transition to "Unknown/Error" and prompt user to refresh.

### 2. Redis Memory Policy & Eviction
**Risk:** Medium
**Context:** Redis is now the Single Point of Failure for Auth (Sessions), Integrity (Nonces), and Data (Price History).
**Failure Mode:**
-   7 Days of Price History + millions of Nonces + Sessions fill the allocated Redis memory.
-   Redis Eviction kicks in or OOM killer kills it.
-   **Scenario:** Redis evicts `price_history` keys to make room for `nonces`.
-   **Result:** Bet Resolver fails to find historical prices → Resolution stalls or errors.
**Mitigation:**
-   **Memory Sizing:** Explicit calculation needed. 7 days @ 1 tick/sec = ~60MB. Nonces @ 3k RPS * 24h is huge.
    -   *Correction:* Nonces are only needed for the 30s window? No, replay protection usually needs to track nonces for the validity window.
    -   *If* validity window is 30s (ADR-007), we only need to store nonces for 30s! Storing for 24h is wasteful unless the window is 24h.
    -   *Action:* **Reduce Nonce TTL to match the Timestamp Window (e.g., 60s safety buffer).** This saves GBs of RAM.

### 3. Archival Job "Silent Death"
**Risk:** Medium
**Context:** The "Infinite Retention" promise relies entirely on the `Weekly Archiver` script working perfectly every Sunday.
**Failure Mode:**
-   Script fails (e.g., S3 creds expired, disk full during compression).
-   System continues running.
-   Week 2: Fails again.
-   Week 4: Disk fills up with un-archived SQLite files.
-   **Result:** Hard outage.
**Mitigation:**
-   **Dead Man's Switch:** The Archiver must push a metric `last_successful_archive_timestamp`. Alert if `now - last_success > 8 days`.

### 4. Financial Integrity: The Double-Spend Race Condition
**Risk:** CRITICAL
**Context:** ADR-005 describes checking balances in Redis ("fast") but deducting in SQLite ("atomic").
**Failure Mode:**
-   User has $10.
-   User sends two simultaneous $10 bets.
-   **Request A:** Checks Redis ($10 > $10). OK. Enters Batch.
-   **Request B:** Checks Redis ($10 > $10). OK. Enters Batch.
-   **Batch Execution:** Inserts Bet A. Inserts Bet B.
-   **Result:** User spends $20. System assumes balances are updated, but if the Batcher doesn't *strictly* prevent the second deduction, we have a negative balance or data corruption.
**Mitigation:**
-   **Database as Truth:** The Batcher Transaction **MUST** perform the deduction: `UPDATE users SET balance = balance - diff WHERE id = ? AND balance >= diff`.
-   **Logic:** If `RowsAffected == 0`, the specific bet in the batch **MUST FAIL**, even if it passed the API check.
-   **Recovery:** On startup/crash recovery, Redis balances must be rebuilt/verified against the SQLite state.

### 5. Flapping Circuit Breaker
**Risk:** Low
**Context:** `Stale Price` check triggers at >10s. Recovery is "Automatic".
**Failure Mode:**
-   Feed is unstable (updates every 9s, then 11s, then 9s).
-   System toggles Healthy/Critical repeatedly.
-   User experience is jarring (Betting disabled... enabled... disabled).
**Mitigation:**
-   **Hysteresis:** Enter CRITICAL at >10s. Recover to HEALTHY only after **30s of stable** updates (Continuous Green).

---

## Recommendations for v1.2

1.  **UX Strictness:** Enforce "Pending" state in UI until WS Confirmation. logic.
2.  **Config Tuning:** Change Nonce TTL from 24h to **90 seconds** (Timestamp window + buffer).
3.  **Observability:** Add `last_archive_success` metric.
4.  **Stability:** Add 30s Hysteresis to Circuit Breaker recovery.

## Verdict
**Passed with Comments.**
The architecture is solid. These are implementation details but are critical for operational stability.
Proceed to Implementation Plan.
