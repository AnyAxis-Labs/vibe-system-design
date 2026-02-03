# Architecture Hardening Audit: Round 003

**Date:** 2026-02-03
**Context:** Final Review of "Hardened v1.2" & Analysis of Financial Persistence Model
**Status:** RISKS IDENTIFIED (FINANCIAL AUDITABILITY)

## Executive Summary
The system is operationally solid (3K RPS, Anti-Double-Spend, Archival).
However, the **Financial Integrity Model** (ADR-009) relies on a **Mutable Balance Column** as the Source of Truth.
The User's critique is valid: In financial systems, *Events* (Ledger) should be the source of truth to allow reconstruction.
Relying solely on `UPDATE users SET balance = balance - x` risks "Drift" that cannot be debugged.

---

## 🔨 Risk Analysis: Mutable Balance vs. Event Ledger

### The Current Approach (ADR-009)
- **Mechanism:** `UPDATE users SET balance = balance - 10`.
- **Pros:** Extremely fast, simple, low storage.
- **Cons:** **Destructive Write.** If a bug causes a double-deduction or a manual DB edit happens, the previous state is lost forever. There is no "Paper Trail" to prove *why* a user has $50.

### The User's Suggestion (Event Sourcing)
- **Mechanism:** Store `BalanceDelta` (+10, -10). Replay to get Balance.
- **Pros:** Perfect auditability.
- **Cons:** **Scaling.** Replaying 100,000 bets to check if a user can afford a $1 bet at 3,000 RPS is impossible on SQLite without heavy snapshots/caching.

### The Hybrid "V2" Solution (The Missing Piece)
We can satisfy *both* constraints (Speed + Auditability) without full Event Sourcing complexity.

**Recommendation:** Introduce a **`transaction_ledger`** table.
1.  **Strict Rule:** *Every* balance change must have a corresponding Row in `transaction_ledger`.
2.  **Implementation:** The Batcher inserts *both* the Bet AND a Ledger Entry in the same transaction.
    ```sql
    BEGIN;
    -- 1. Deduct (Speed Constraint)
    UPDATE users SET balance = balance - 10 WHERE id = 1 AND balance >= 10;
    
    -- 2. Audit (Integrity Constraint)
    INSERT INTO transaction_ledger (user_id, amount, ref_type, ref_id) 
    VALUES (1, -10, 'BET', 'bet_uuid_123');
    
    -- 3. Record Bet
    INSERT INTO bets ...
    COMMIT;
    ```
3.  **Result:** `users.balance` is a "Cached Snapshot" of the Sum of `transaction_ledger`.
4.  **Recovery:** If data corruption is suspected, we can `SUM(amount)` from the ledger to verify/fix the `users` table.

---

## 🔍 Other Hardening Checks (Round 3)

### 1. UX "Pending" State
- **Status:** **PASS**. The new UX spec requiring WebSocket confirmation before "Solid Green" solves the trust gap.

### 2. Redis Memory (Nonce TTL)
- **Status:** **PASS**. Reducing TTL to 90s fixes the OOM risk.

### 3. Circuit Breaker Hysteresis
- **Status:** **PASS**. The 30s stability window prevents flapping.

---

## Final Recommendations for Implementation
1.  **Adopt the Hybrid Ledger:** Do not Rely on Mutable Balance alone. Add a `transaction_ledger` table.
    -   *Constraint Check:* Does this kill performance?
    -   *Analysis:* It adds 1 INSERT per Bet. SQLite can handle 3,000 inserts/sec easily *if batched*. Since we are already batching, this is practically free.
2.  **Schema Definition:** defining the `users` table without a `balance` column is suicide for performance. Keep the column, but treat it as a "Materialized View" of the ledger.

## Verdict
**Conditional Pass.**
The architecture is approved **ONLY IF** a Ledger Table is added to support the Mutable Balance.
Without a Ledger, a financial dispute ("Where did my money go?") is unanswerable.
