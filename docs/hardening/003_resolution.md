# Hardening Resolution: 003_followup_critique.md

**Date:** 2026-02-03  
**Status:** ALL RISKS ADDRESSED  
**Architecture Version:** v1.3 (Financially Auditable)

---

## Summary

All critiques from Round 003 have been addressed:

| # | Risk | Severity | Solution | ADR |
|---|------|----------|----------|-----|
| 1 | Mutable Balance (no audit trail) | High | Hybrid Transaction Ledger | ADR-010 |
| 2 | Dispute resolution impossible | High | Query-able ledger with refs | ADR-010 |
| 3 | Data corruption undetected | Medium | Daily integrity check | ADR-010 |

---

## Detailed Resolution

### 1. Mutable Balance vs. Event Ledger

**Problem:** ADR-009's `UPDATE users SET balance = balance - x` loses previous state. No paper trail for disputes.

**Rejected Alternative:** Full Event Sourcing
```sql
-- Replay 100K events to check balance at 3K RPS = impossible
```

**Solution:** Hybrid Ledger (ADR-010)
```sql
-- Same transaction:
BEGIN;
  -- 1. Speed: Mutable balance update
  UPDATE users SET balance = balance - 10 
  WHERE id = 1 AND balance >= 10
  RETURNING balance;  -- new_balance = 90
  
  -- 2. Audit: Append-only ledger
  INSERT INTO transaction_ledger 
    (id, user_id, amount, balance_after, ref_type, ref_id)
  VALUES 
    ('uuid', 1, -10, 90, 'BET', 'bet_123');
    
  -- 3. Record bet
  INSERT INTO bets ...
COMMIT;
```

**Benefits:**
- ✅ Performance: `users.balance` is fast read (materialized view)
- ✅ Auditability: Every change recorded with `ref_type`, `ref_id`, `balance_after`
- ✅ Recoverability: `SUM(amount)` from ledger can rebuild balance
- ✅ Verifiability: `balance_after` column allows quick integrity checks

---

### 2. Ledger Schema

```sql
CREATE TABLE transaction_ledger (
    id TEXT PRIMARY KEY,              -- ULID (sortable)
    user_id TEXT NOT NULL,
    amount INTEGER NOT NULL,          -- -10 for bet, +20 for payout
    balance_after INTEGER NOT NULL,   -- Balance snapshot post-transaction
    ref_type TEXT NOT NULL,           -- 'BET', 'PAYOUT', 'DEPOSIT'
    ref_id TEXT NOT NULL,             -- Foreign key to source record
    metadata JSON,                    -- Flexible context
    created_at TIMESTAMP
);

-- Indexes
CREATE INDEX idx_user_time ON transaction_ledger(user_id, created_at);
CREATE INDEX idx_ref ON transaction_ledger(ref_type, ref_id);
```

---

### 3. Storage Impact

| Metric | Before (ADR-009) | After (ADR-010) |
|--------|------------------|-----------------|
| Statements/bet | 2 | 3 (+1 INSERT) |
| Storage/bet | ~200 bytes | ~350 bytes |
| Daily growth | 10 GB | 17.5 GB |
| Batch throughput | Unchanged | Unchanged (30 batches/sec) |

**Verdict:** Manageable with existing weekly archival strategy.

---

### 4. Audit Procedures

#### Daily Integrity Check
```sql
-- Verify no drift between balance and ledger
WITH ledger_sums AS (
    SELECT user_id, SUM(amount) as ledger_balance
    FROM transaction_ledger
    GROUP BY user_id
)
SELECT u.id, u.balance, ls.ledger_balance
FROM users u
LEFT JOIN ledger_sums ls ON u.id = ls.user_id
WHERE u.balance != COALESCE(ls.ledger_balance, 0);
```

#### User Dispute Query
```sql
-- "Where did my $50 go?"
SELECT created_at, ref_type, ref_id, amount, balance_after
FROM transaction_ledger
WHERE user_id = 'user_123'
ORDER BY created_at DESC
LIMIT 50;
```

#### Recovery from Corruption
```sql
-- Rebuild balance from ledger
UPDATE users 
SET balance = (
    SELECT COALESCE(SUM(amount), 0) 
    FROM transaction_ledger 
    WHERE user_id = users.id
);
```

---

### 5. Pass Checks from Round 003

| Check | Status |
|-------|--------|
| UX "Pending" State | ✅ PASS (v1.2) |
| Redis Memory (Nonce TTL) | ✅ PASS (v1.2, 90s) |
| Circuit Breaker Hysteresis | ✅ PASS (v1.2, 30s) |

---

## Final Architecture State (v1.3)

### Data Flow — Auditable Bet Placement

```
User Tap
    ↓
Frontend: "PENDING" state
    ↓
POST /api/bets
    ↓
API: Validate signature + nonce (90s TTL)
    ↓
API: Check circuit breaker
    ↓
API: Submit to batcher channel
    ↓
Return 202 Accepted
    ↓
Batcher flush (every 100ms or 100 bets):
    BEGIN TRANSACTION
        UPDATE users SET balance = balance - ? WHERE balance >= ?
        INSERT INTO transaction_ledger (amount, balance_after, ref_type, ref_id)
        INSERT INTO bets
    COMMIT
    ↓
WebSocket: "bet:confirmed" event
    ↓
Frontend: "CONFIRMED" state
```

---

## Verdict

**Reviewer:** System Architecture Guardian  
**Verdict:** ✅ **ACCEPTED FOR IMPLEMENTATION**

All financial auditability requirements satisfied. The hybrid ledger approach (mutable balance + append-only events) is the correct trade-off for this system's constraints (3K RPS, SQLite, 2-week timeline).

---

## Implementation Checklist

- [ ] Create `transaction_ledger` table with indexes
- [ ] Update `BetBatcher` to write ledger entries
- [ ] Include ledger in weekly archival partitions
- [ ] Implement daily integrity check (cron)
- [ ] Document dispute resolution queries for support team
