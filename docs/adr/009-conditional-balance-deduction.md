# ADR 009: Conditional Balance Deduction for Double-Spend Prevention

## Status
Accepted (Hardening Response 002)

## Context
Hardening Audit #002 identified a critical race condition: Redis balance check + SQLite batch deduction creates a window for double-spending.

## The Race Condition

```
User Balance: $10

Request A                    Request B
─────────────────────────────────────────────────
Check Redis ($10 >= $10)    Check Redis ($10 >= $10)
    ↓ OK                         ↓ OK
Enter Batch                  Enter Batch
─────────────────────────────────────────────────
                    ↓
            Batcher Flushes
                    ↓
    INSERT Bet A (deduct $10)  
    INSERT Bet B (deduct $10)  ← Overdraft! Balance = -$10
```

## Decision
Use **conditional SQL UPDATE** as the single source of truth for balance deduction.

## Solution Design

### Database Schema Change

```sql
-- Users table with optimistic locking
CREATE TABLE users (
    id TEXT PRIMARY KEY,
    wallet_address TEXT UNIQUE NOT NULL,
    balance INTEGER NOT NULL DEFAULT 0,  -- Stored in cents/smallest unit
    version INTEGER NOT NULL DEFAULT 1    -- Optimistic locking
);

-- Bets table tracks deduction outcome
CREATE TABLE bets (
    id TEXT PRIMARY KEY,
    user_id TEXT NOT NULL,
    cell_id TEXT NOT NULL,
    amount INTEGER NOT NULL,
    status TEXT NOT NULL,  -- 'pending', 'confirmed', 'insufficient_funds'
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id)
);

-- Partial index for fast pending queries
CREATE INDEX idx_bets_pending ON bets(user_id, status) WHERE status = 'pending';
```

### Batcher Implementation with Conditional Deduction

```go
const (
    StatusPending            = "pending"
    StatusConfirmed          = "confirmed"
    StatusInsufficientFunds  = "insufficient_funds"
)

type BetBatch struct {
    Bets []PendingBet
}

type PendingBet struct {
    BetID    string
    UserID   string
    CellID   string
    Amount   int64  // in cents
    Done     chan BetResult
}

type BetResult struct {
    Success bool
    Status  string
    Error   error
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

    // Prepare statements for efficiency
    deductStmt, _ := tx.Prepare(`
        UPDATE users 
        SET balance = balance - ?,
            version = version + 1
        WHERE id = ? 
        AND balance >= ?
    `)
    
    insertStmt, _ := tx.Prepare(`
        INSERT INTO bets (id, user_id, cell_id, amount, status)
        VALUES (?, ?, ?, ?, ?)
    `)

    results := make(map[string]BetResult)

    for _, bet := range bets {
        // Step 1: Attempt conditional balance deduction
        res, err := deductStmt.Exec(bet.Amount, bet.UserID, bet.Amount)
        if err != nil {
            results[bet.BetID] = BetResult{false, "", err}
            continue
        }

        rowsAffected, _ := res.RowsAffected()
        
        if rowsAffected == 0 {
            // Insufficient funds - record as failed
            insertStmt.Exec(bet.BetID, bet.UserID, bet.CellID, bet.Amount, StatusInsufficientFunds)
            results[bet.BetID] = BetResult{
                Success: false,
                Status:  StatusInsufficientFunds,
                Error:   ErrInsufficientFunds,
            }
            continue
        }

        // Step 2: Insert confirmed bet
        _, err = insertStmt.Exec(bet.BetID, bet.UserID, bet.CellID, bet.Amount, StatusConfirmed)
        if err != nil {
            // This shouldn't happen with proper constraints, but handle it
            results[bet.BetID] = BetResult{false, "", err}
            continue
        }

        results[bet.BetID] = BetResult{
            Success: true,
            Status:  StatusConfirmed,
            Error:   nil,
        }
    }

    if err := tx.Commit(); err != nil {
        b.failAll(bets, err)
        return
    }

    // Signal results to waiting goroutines
    for _, bet := range bets {
        bet.Done <- results[bet.BetID]
    }

    // Update Redis cache (best-effort, non-critical)
    b.syncRedisBalances(bets, results)
}

// syncRedisBalances updates Redis cache after successful DB commit
func (b *BetBatcher) syncRedisBalances(bets []PendingBet, results map[string]BetResult) {
    for _, bet := range bets {
        result := results[bet.BetID]
        if result.Success {
            // Decrement Redis balance (best effort)
            b.redis.DecrBy(context.Background(), 
                fmt.Sprintf("balance:%s", bet.UserID), 
                bet.Amount)
        }
    }
}
```

### API Flow with Database-First Validation

```go
func (h *BetHandler) PlaceBet(c echo.Context) error {
    var req BetRequest
    if err := c.Bind(&req); err != nil {
        return err
    }

    // 1. Validate signature (anti-replay)
    if err := h.validator.Validate(req); err != nil {
        return c.JSON(401, map[string]string{"error": "invalid_signature"})
    }

    // 2. Quick Redis balance check (early rejection, not authoritative)
    balanceKey := fmt.Sprintf("balance:%s", req.UserID)
    cachedBalance, err := h.redis.Get(c.Request().Context(), balanceKey).Int64()
    
    // Early rejection if definitely insufficient
    if err == nil && cachedBalance < req.Amount {
        return c.JSON(400, map[string]string{"error": "insufficient_funds"})
    }

    // 3. Check cell availability (Redis cache)
    if !h.grid.IsCellAvailable(req.CellID) {
        return c.JSON(400, map[string]string{"error": "cell_unavailable"})
    }

    // 4. Submit to batcher (database is authoritative)
    result := h.batcher.Submit(PendingBet{
        BetID:  generateUUID(),
        UserID: req.UserID,
        CellID: req.CellID,
        Amount: req.Amount,
        Done:   make(chan BetResult, 1),
    })

    // 5. Wait for batch result with timeout
    select {
    case result := <-result.Done:
        if !result.Success {
            if result.Status == StatusInsufficientFunds {
                return c.JSON(400, map[string]string{"error": "insufficient_funds"})
            }
            return c.JSON(500, map[string]string{"error": "internal_error"})
        }
        
        // Return 202 Accepted, WebSocket will confirm
        return c.JSON(202, map[string]string{
            "status": "accepted",
            "bet_id": result.BetID,
        })
        
    case <-time.After(5 * time.Second):
        // Batcher timeout - bet may or may not be processed
        return c.JSON(504, map[string]string{
            "error": "timeout",
            "message": "Bet status unknown, check history",
        })
    }
}
```

### Redis Cache Consistency

Redis balance is a **cache**, not authoritative:

| Scenario | Action |
|----------|--------|
| Cache hit | Use for early rejection only |
| Cache miss | Query SQLite, populate cache |
| After successful bet | Decrement cache (best effort) |
| After failed bet | No cache change |
| Startup/recovery | Rebuild Redis from SQLite: `SELECT id, balance FROM users` |

### Startup Recovery

```go
func (b *BetBatcher) RebuildRedisCache() error {
    rows, err := b.db.Query("SELECT id, balance FROM users")
    if err != nil {
        return err
    }
    defer rows.Close()

    pipe := b.redis.Pipeline()
    for rows.Next() {
        var userID string
        var balance int64
        rows.Scan(&userID, &balance)
        pipe.Set(context.Background(), 
            fmt.Sprintf("balance:%s", userID), 
            balance, 
            24*time.Hour)
    }
    _, err = pipe.Exec(context.Background())
    return err
}
```

## Trade-off Analysis

| Alternative | Why Rejected |
|-------------|--------------|
| Redis as truth (decrement there) | Redis not durable; crash = lost funds |
| Distributed locking (Redis Redlock) | Complex, slower, still has edge cases |
| Two-phase commit | Overkill for single VM; adds latency |
| Pessimistic locking (SELECT FOR UPDATE) | Doesn't work well with SQLite's single writer |

## NFR Compliance

| NFR | Compliance |
|-----|------------|
| Double-spend prevention | ✅ Conditional SQL UPDATE enforces balance >= amount |
| Atomicity | ✅ Single transaction per batch |
| Correctness | ✅ Database is single source of truth |
| Performance | ✅ Still batched; conditional check is fast (index on user_id) |
| ≤100ms latency | ✅ Early rejection from cache; batch commit async |

## Decision Date
2026-02-03

## Author
System Architect (Hardening Response 002)
