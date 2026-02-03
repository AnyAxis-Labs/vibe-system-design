# ADR 010: Hybrid Transaction Ledger for Financial Auditability

## Status
Accepted (Hardening Response 003)

## Context
Hardening Audit #003 identified that ADR-009's mutable balance approach lacks financial auditability. Without a ledger, disputes ("Where did my money go?") are unanswerable.

## The Problem

### Mutable Balance Only (ADR-009)
```sql
UPDATE users SET balance = balance - 10 WHERE id = ?;
-- Previous state is LOST. No paper trail.
```

**Risk:** If a bug causes double-deduction or manual DB corruption, previous state is unrecoverable.

### Full Event Sourcing (Rejected)
```sql
-- Balance = SUM(transaction_ledger.amount) for user
-- Replay 100K events to check balance at 3K RPS = impossible
```

**Risk:** Too slow for our performance requirements.

## Decision
Implement a **Hybrid Ledger**: Mutable balance for speed, append-only ledger for auditability.

## Solution Design

### Schema

```sql
-- Users table: Mutable balance (performance cache)
CREATE TABLE users (
    id TEXT PRIMARY KEY,
    wallet_address TEXT UNIQUE NOT NULL,
    balance INTEGER NOT NULL DEFAULT 0,  -- Materialized view of ledger
    version INTEGER NOT NULL DEFAULT 1,  -- Optimistic locking
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Transaction Ledger: Append-only, immutable audit trail
CREATE TABLE transaction_ledger (
    id TEXT PRIMARY KEY,                    -- ULID for sortability
    user_id TEXT NOT NULL,
    amount INTEGER NOT NULL,                -- Negative for debit, positive for credit
    balance_after INTEGER NOT NULL,         -- Snapshot for quick verification
    ref_type TEXT NOT NULL,                 -- 'BET', 'PAYOUT', 'DEPOSIT', 'WITHDRAWAL'
    ref_id TEXT NOT NULL,                   -- Foreign key to bets/deposits/etc
    metadata JSON,                          -- Flexible context
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (user_id) REFERENCES users(id),
    INDEX idx_user_time (user_id, created_at),
    INDEX idx_ref (ref_type, ref_id)
) STRICT;

-- Bets table
CREATE TABLE bets (
    id TEXT PRIMARY KEY,
    user_id TEXT NOT NULL,
    cell_id TEXT NOT NULL,
    amount INTEGER NOT NULL,
    reward_rate REAL NOT NULL,
    status TEXT NOT NULL,  -- 'pending', 'confirmed', 'won', 'lost', 'insufficient_funds'
    placed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    resolved_at TIMESTAMP,
    payout_amount INTEGER,  -- NULL if lost
    
    FOREIGN KEY (user_id) REFERENCES users(id)
);
```

### Why `balance_after`?

The `balance_after` column in the ledger serves as a **checksum**:
```sql
-- Quick integrity verification per user
SELECT 
    l1.balance_after - l1.amount AS expected_balance_before,
    LAG(l1.balance_after) OVER (ORDER BY l1.created_at) AS actual_previous_balance
FROM transaction_ledger l1
WHERE l1.user_id = 'user_123'
ORDER BY l1.created_at;
```

### Batcher Implementation (Updated)

```go
type LedgerEntry struct {
    ID            string
    UserID        string
    Amount        int64   // Negative for bets
    BalanceAfter  int64   // New balance after this transaction
    RefType       string  // "BET"
    RefID         string  // Bet ID
    Metadata      map[string]interface{}
}

func (b *BetBatcher) flush(bets []PendingBet) {
    tx, err := b.db.BeginTx(context.Background(), &sql.TxOptions{
        Isolation: sql.LevelSerializable,
    })
    if err != nil {
        b.failAll(bets, err)
        return
    }
    defer tx.Rollback()

    // Prepare statements
    deductStmt, _ := tx.Prepare(`
        UPDATE users 
        SET balance = balance - ?, version = version + 1
        WHERE id = ? AND balance >= ?
        RETURNING balance
    `)
    
    ledgerStmt, _ := tx.Prepare(`
        INSERT INTO transaction_ledger 
        (id, user_id, amount, balance_after, ref_type, ref_id, metadata)
        VALUES (?, ?, ?, ?, ?, ?, ?)
    `)
    
    betStmt, _ := tx.Prepare(`
        INSERT INTO bets (id, user_id, cell_id, amount, reward_rate, status)
        VALUES (?, ?, ?, ?, ?, ?)
    `)

    results := make(map[string]BetResult)

    for _, bet := range bets {
        // Step 1: Conditional deduction + get new balance
        var newBalance int64
        err := deductStmt.QueryRow(bet.Amount, bet.UserID, bet.BetID).Scan(&newBalance)
        
        if err == sql.ErrNoRows {
            // Insufficient funds
            results[bet.BetID] = BetResult{
                Success: false,
                Status:  StatusInsufficientFunds,
                Error:   ErrInsufficientFunds,
            }
            continue
        }
        if err != nil {
            results[bet.BetID] = BetResult{Success: false, Error: err}
            continue
        }

        // Step 2: Record in ledger (audit trail)
        ledgerID := generateULID()
        metadata := map[string]interface{}{
            "cell_id":     bet.CellID,
            "timestamp":   time.Now().Unix(),
        }
        metadataJSON, _ := json.Marshal(metadata)
        
        _, err = ledgerStmt.Exec(
            ledgerID,
            bet.UserID,
            -bet.Amount,        // Negative (debit)
            newBalance,         // Balance after deduction
            "BET",
            bet.BetID,
            metadataJSON,
        )
        if err != nil {
            results[bet.BetID] = BetResult{Success: false, Error: err}
            continue
        }

        // Step 3: Record bet
        _, err = betStmt.Exec(
            bet.BetID,
            bet.UserID,
            bet.CellID,
            bet.Amount,
            bet.RewardRate,
            StatusConfirmed,
        )
        if err != nil {
            results[bet.BetID] = BetResult{Success: false, Error: err}
            continue
        }

        results[bet.BetID] = BetResult{
            Success:    true,
            Status:     StatusConfirmed,
            NewBalance: newBalance,
        }
    }

    if err := tx.Commit(); err != nil {
        b.failAll(bets, err)
        return
    }

    // Signal results
    for _, bet := range bets {
        bet.Done <- results[bet.BetID]
    }

    // Best-effort Redis sync
    b.syncRedisBalances(bets, results)
}
```

## Performance Impact Analysis

| Aspect | Before (ADR-009) | After (ADR-010) | Impact |
|--------|------------------|-----------------|--------|
| Statements per bet | 2 (UPDATE + INSERT bets) | 3 (UPDATE + INSERT ledger + INSERT bets) | +1 INSERT |
| Batch of 100 | 200 statements | 300 statements | +50% statements |
| SQLite capacity | 10,000+ writes/sec | 10,000+ writes/sec | **Negligible** |
| Throughput @ 3K RPS | 30 batches/sec | 30 batches/sec | **Unchanged** |
| Storage per bet | ~200 bytes | ~350 bytes (+ledger) | +75% |
| Daily growth | 10 GB | 17.5 GB | Manageable with archival |

**Conclusion:** The ledger adds only 1 INSERT per bet. Since we're batching 100 bets per transaction, this is practically free performance-wise.

## Audit & Recovery Procedures

### 1. Daily Integrity Check

```sql
-- Verify users.balance matches SUM(ledger.amount)
WITH ledger_balances AS (
    SELECT 
        user_id,
        SUM(amount) as ledger_sum
    FROM transaction_ledger
    GROUP BY user_id
)
SELECT 
    u.id,
    u.balance as user_balance,
    lb.ledger_sum,
    u.balance - lb.ledger_sum as drift
FROM users u
LEFT JOIN ledger_balances lb ON u.id = lb.user_id
WHERE u.balance != lb.ledger_sum
   OR (u.balance = 0 AND lb.ledger_sum IS NULL AND u.id IN (SELECT user_id FROM transaction_ledger));
```

### 2. User Dispute Resolution

```sql
-- "Where did my money go?"
SELECT 
    created_at,
    ref_type,
    ref_id,
    amount,
    balance_after,
    metadata
FROM transaction_ledger
WHERE user_id = 'user_123'
ORDER BY created_at DESC
LIMIT 50;
```

### 3. Rebalance on Corruption

```sql
-- If drift detected, recalculate
UPDATE users 
SET balance = (
    SELECT COALESCE(SUM(amount), 0) 
    FROM transaction_ledger 
    WHERE user_id = users.id
)
WHERE id = 'user_123';
```

## Storage Growth Projection

| Timeframe | Ledger Size | With Archival Strategy |
|-----------|-------------|------------------------|
| 1 day | ~7.5 GB | Hot storage |
| 1 week | ~52 GB | Hot storage |
| 1 month | ~200 GB | S3 archival kicks in |
| Forever | Infinite | S3 STANDARD_IA |

**Note:** Ledger is archived with weekly partitions, same as bets table.

## Trade-off Analysis

| Alternative | Why Rejected |
|-------------|--------------|
| Mutable balance only | No audit trail, unanswerable disputes |
| Full event sourcing (no balance column) | Too slow for 3K RPS balance checks |
| Separate event store (Kafka) | Extra process, complexity, out of scope |
| Monthly ledger rotation | Too complex; weekly matches bet archival |

## NFR Compliance

| NFR | Compliance |
|-----|------------|
| 3K RPS | ✅ Batching makes extra INSERT negligible |
| ≤100ms latency | ✅ Unchanged (batch async) |
| Financial auditability | ✅ Complete paper trail in ledger |
| Dispute resolution | ✅ Query ledger by user/time/ref |
| Data integrity | ✅ `balance_after` allows verification |
| Recovery | ✅ Can rebuild balance from ledger |

## Decision Date
2026-02-03

## Author
System Architecture Guardian (Hardening Response 003)
