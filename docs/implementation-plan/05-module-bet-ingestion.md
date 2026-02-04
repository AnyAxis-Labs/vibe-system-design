# Module 3: Bet Ingestion (Vertical)

**Goal:** Implement the complete bet placement flow with write batching, conditional balance deduction, and hybrid ledger.

**Duration:** Days 5-7  
**Dependencies:** Module 1 (Core), Module 2 (Auth)  
**Priority:** CRITICAL (core business logic)

---

## Module Scope

This module implements:
- Write batching channel with flush logic
- Conditional SQL balance deduction (ADR-009)
- Hybrid transaction ledger (ADR-010)
- Bet placement API endpoint
- WebSocket confirmation

**Source ADRs:** ADR-005 (Write Batching), ADR-009 (Conditional Deduction), ADR-010 (Hybrid Ledger)  
**Source C4:** Bet Batcher, API Server, WebSocket Server

> **⚠️ Critical:** ADR-010 EXTENDS ADR-009 — both must be implemented together in the same transaction.

---

## Task 1: Write Batching (ADR-005)

### Task 1.1: Implement Bet Batcher Core
**Owner:** Backend Engineer  
**Estimated:** 5 hours  
**Source ADR:** ADR-005 (Write Batching for Bet Ingestion)  
**Source C4:** Bet Batcher container

**Description:**
Implement channel-based bet batching with size and time triggers.

**DoD:**
- [ ] Buffered channel for bet submission (10K capacity)
- [ ] Batch flusher goroutine
- [ ] Size trigger: 100 items
- [ ] Time trigger: 100ms timeout
- [ ] Per-bet result signaling via channel
- [ ] Graceful shutdown with drain

**Implementation:**
```go
// pkg/batcher/batcher.go
package batcher

import (
    "context"
    "database/sql"
    "sync"
    "time"
    
    "github.com/tap-trading/backend/pkg/db"
    "github.com/tap-trading/backend/pkg/models"
)

const (
    BatchSize     = 100
    BatchTimeout  = 100 * time.Millisecond
    ChannelBuffer = 10000
)

type PendingBet struct {
    BetID       string
    UserID      string
    CellID      string
    Amount      int64
    RewardRate  float64
    ResultChan  chan BetResult
}

type BetResult struct {
    Success    bool
    Status     string
    NewBalance int64
    Error      error
}

type Batcher struct {
    db       *db.Client
    redis    *redis.Client
    channel  chan PendingBet
    shutdown chan struct{}
    wg       sync.WaitGroup
}

func New(db *db.Client, redis *redis.Client) *Batcher {
    return &Batcher{
        db:       db,
        redis:    redis,
        channel:  make(chan PendingBet, ChannelBuffer),
        shutdown: make(chan struct{}),
    }
}

func (b *Batcher) Start() {
    b.wg.Add(1)
    go b.run()
}

func (b *Batcher) Stop() {
    close(b.shutdown)
    b.wg.Wait()
}

func (b *Batcher) Submit(bet PendingBet) {
    select {
    case b.channel <- bet:
        // Submitted successfully
    default:
        // Channel full - apply backpressure
        bet.ResultChan <- BetResult{
            Success: false,
            Error:   fmt.Errorf("batcher overloaded"),
        }
    }
}

func (b *Batcher) run() {
    defer b.wg.Done()
    
    ticker := time.NewTicker(BatchTimeout)
    defer ticker.Stop()
    
    batch := make([]PendingBet, 0, BatchSize)
    
    for {
        select {
        case bet := <-b.channel:
            batch = append(batch, bet)
            
            if len(batch) >= BatchSize {
                b.flush(batch)
                batch = batch[:0]
                ticker.Reset(BatchTimeout)
            }
            
        case <-ticker.C:
            if len(batch) > 0 {
                b.flush(batch)
                batch = batch[:0]
            }
            
        case <-b.shutdown:
            // Drain remaining
            for len(b.channel) > 0 && len(batch) < BatchSize {
                select {
                case bet := <-b.channel:
                    batch = append(batch, bet)
                default:
                }
            }
            if len(batch) > 0 {
                b.flush(batch)
            }
            return
        }
    }
}

func (b *Batcher) flush(bets []PendingBet) {
    // Implementation in Task 1.2
}
```

---

### Task 1.2: Implement Flush with Conditional Deduction + Ledger (ADR-009 + ADR-010)
**Owner:** Backend Engineer  
**Estimated:** 6 hours  
**Source ADR:** ADR-009 (Conditional Deduction), ADR-010 (Hybrid Ledger)  
**Source C4:** Bet Batcher → SQLite

**Description:**
Implement the batched flush with conditional SQL balance deduction and hybrid ledger write. This is the critical financial transaction.

**DoD:**
- [ ] Single transaction for entire batch
- [ ] Conditional UPDATE: `balance >= amount` check
- [ ] Ledger INSERT with `balance_after` snapshot
- [ ] Bet INSERT with status
- [ ] RowsAffected check for insufficient funds
- [ ] Best-effort Redis balance sync (post-commit)

**Implementation:**
```go
// pkg/batcher/flush.go
package batcher

import (
    "context"
    "database/sql"
    "encoding/json"
    "fmt"
    "time"
    
    "github.com/oklog/ulid/v2"
)

const (
    StatusConfirmed         = "confirmed"
    StatusInsufficientFunds = "insufficient_funds"
)

func (b *Batcher) flush(bets []PendingBet) {
    ctx := context.Background()
    
    tx, err := b.db.BeginTx(ctx, &sql.TxOptions{
        Isolation: sql.LevelSerializable,
    })
    if err != nil {
        b.failAll(bets, err)
        return
    }
    defer tx.Rollback()

    // Prepare statements
    deductStmt, err := tx.PrepareContext(ctx, `
        UPDATE users 
        SET balance = balance - ?,
            version = version + 1,
            updated_at = CURRENT_TIMESTAMP
        WHERE id = ? 
        AND balance >= ?
        RETURNING balance
    `)
    if err != nil {
        b.failAll(bets, err)
        return
    }
    defer deductStmt.Close()
    
    ledgerStmt, err := tx.PrepareContext(ctx, `
        INSERT INTO transaction_ledger 
        (id, user_id, amount, balance_after, ref_type, ref_id, metadata, created_at)
        VALUES (?, ?, ?, ?, ?, ?, ?, ?)
    `)
    if err != nil {
        b.failAll(bets, err)
        return
    }
    defer ledgerStmt.Close()
    
    betStmt, err := tx.PrepareContext(ctx, `
        INSERT INTO bets 
        (id, user_id, cell_id, amount, reward_rate, status, placed_at)
        VALUES (?, ?, ?, ?, ?, ?, ?)
    `)
    if err != nil {
        b.failAll(bets, err)
        return
    }
    defer betStmt.Close()

    results := make(map[string]BetResult)
    now := time.Now()

    for _, bet := range bets {
        // Step 1: Conditional deduction with RETURNING
        var newBalance int64
        err := deductStmt.QueryRowContext(ctx, bet.Amount, bet.UserID, bet.Amount).Scan(&newBalance)
        
        if err == sql.ErrNoRows {
            // Insufficient funds - record failed bet
            _, _ = betStmt.ExecContext(ctx,
                bet.BetID, bet.UserID, bet.CellID, bet.Amount,
                bet.RewardRate, StatusInsufficientFunds, now,
            )
            
            results[bet.BetID] = BetResult{
                Success: false,
                Status:  StatusInsufficientFunds,
                Error:   fmt.Errorf("insufficient funds"),
            }
            continue
        }
        
        if err != nil {
            results[bet.BetID] = BetResult{
                Success: false,
                Error:   fmt.Errorf("deduction failed: %w", err),
            }
            continue
        }

        // Step 2: Record in ledger (ADR-010)
        ledgerID := ulid.Make().String()
        metadata := map[string]interface{}{
            "cell_id":   bet.CellID,
            "timestamp": now.Unix(),
        }
        metadataJSON, _ := json.Marshal(metadata)
        
        _, err = ledgerStmt.ExecContext(ctx,
            ledgerID,
            bet.UserID,
            -bet.Amount,  // Negative for debit
            newBalance,   // Balance after deduction
            "BET",
            bet.BetID,
            metadataJSON,
            now,
        )
        if err != nil {
            results[bet.BetID] = BetResult{
                Success: false,
                Error:   fmt.Errorf("ledger insert failed: %w", err),
            }
            continue
        }

        // Step 3: Record bet
        _, err = betStmt.ExecContext(ctx,
            bet.BetID,
            bet.UserID,
            bet.CellID,
            bet.Amount,
            bet.RewardRate,
            StatusConfirmed,
            now,
        )
        if err != nil {
            results[bet.BetID] = BetResult{
                Success: false,
                Error:   fmt.Errorf("bet insert failed: %w", err),
            }
            continue
        }

        results[bet.BetID] = BetResult{
            Success:    true,
            Status:     StatusConfirmed,
            NewBalance: newBalance,
        }
    }

    // Commit transaction
    if err := tx.Commit(); err != nil {
        b.failAll(bets, err)
        return
    }

    // Signal results to waiting goroutines
    for _, bet := range bets {
        select {
        case bet.ResultChan <- results[bet.BetID]:
        case <-time.After(time.Second):
            // Client not listening, log warning
        }
    }

    // Best-effort Redis sync (non-critical)
    b.syncRedisBalances(bets, results)
}

func (b *Batcher) failAll(bets []PendingBet, err error) {
    for _, bet := range bets {
        select {
        case bet.ResultChan <- BetResult{Success: false, Error: err}:
        default:
        }
    }
}

func (b *Batcher) syncRedisBalances(bets []PendingBet, results map[string]BetResult) {
    ctx := context.Background()
    pipe := b.redis.Pipeline()
    
    for _, bet := range bets {
        result := results[bet.BetID]
        if result.Success {
            key := fmt.Sprintf("balance:%s", bet.UserID)
            pipe.Set(ctx, key, result.NewBalance, 24*time.Hour)
        }
    }
    
    // Fire and forget - Redis is cache, not source of truth
    _, _ = pipe.Exec(ctx)
}
```

**⚠️ Critical Implementation Notes:**
1. **Single Transaction:** All three operations (UPDATE, INSERT ledger, INSERT bet) must be in one transaction
2. **Conditional UPDATE:** `RETURNING balance` gives us new balance atomically
3. **Ledger Always Written:** Even on failure, we could log a ledger entry (optional)
4. **Redis is Cache Only:** Redis sync happens AFTER commit and is best-effort

---

## Task 2: Bet Placement API

### Task 2.1: Implement Bet Handler
**Owner:** Backend Engineer  
**Estimated:** 4 hours  
**Source ADR:** ADR-005, ADR-009, ADR-010  
**Source C4:** API Server

**Description:**
Implement the complete bet placement endpoint with all validations.

**DoD:**
- [ ] `POST /api/bets` endpoint
- [ ] Auth middleware integration
- [ ] Request validation (cell_id, amount)
- [ ] Grid availability check
- [ ] Balance check (Redis cache - early rejection)
- [ ] Submit to batcher
- [ ] Wait for result with timeout
- [ ] Return 202 Accepted or error

**Implementation:**
```go
// cmd/api/handlers/bet.go
package handlers

import (
    "net/http"
    "time"
    
    "github.com/labstack/echo/v4"
    "github.com/tap-trading/backend/pkg/batcher"
    "github.com/tap-trading/backend/pkg/redis"
)

type BetRequest struct {
    CellID     string  `json:"cell_id" validate:"required"`
    Amount     int64   `json:"amount" validate:"required,min=1"`
    RewardRate float64 `json:"reward_rate"`
}

type BetHandler struct {
    batcher *batcher.Batcher
    redis   *redis.Client
}

func NewBetHandler(batcher *batcher.Batcher, redis *redis.Client) *BetHandler {
    return &BetHandler{batcher: batcher, redis: redis}
}

func (h *BetHandler) PlaceBet(c echo.Context) error {
    ctx := c.Request().Context()
    userID := c.Get("user_id").(string)
    
    var req BetRequest
    if err := c.Bind(&req); err != nil {
        return c.JSON(http.StatusBadRequest, map[string]string{
            "error": "invalid request body",
        })
    }
    
    // Validate request
    if err := c.Validate(&req); err != nil {
        return c.JSON(http.StatusBadRequest, map[string]string{
            "error": err.Error(),
        })
    }
    
    // Check cell availability (via Redis cache)
    // This would check grid state - implementation depends on grid module
    
    // Early balance check via Redis cache (not authoritative)
    balanceKey := fmt.Sprintf("balance:%s", userID)
    cachedBalance, err := h.redis.Get(ctx, balanceKey).Int64()
    if err == nil && cachedBalance < req.Amount {
        return c.JSON(http.StatusBadRequest, map[string]string{
            "error": "insufficient_funds",
        })
    }
    
    // Generate bet ID
    betID := ulid.Make().String()
    
    // Submit to batcher
    resultChan := make(chan batcher.BetResult, 1)
    h.batcher.Submit(batcher.PendingBet{
        BetID:      betID,
        UserID:     userID,
        CellID:     req.CellID,
        Amount:     req.Amount,
        RewardRate: req.RewardRate,
        ResultChan: resultChan,
    })
    
    // Wait for result with timeout
    select {
    case result := <-resultChan:
        if !result.Success {
            if result.Status == batcher.StatusInsufficientFunds {
                return c.JSON(http.StatusBadRequest, map[string]string{
                    "error": "insufficient_funds",
                })
            }
            return c.JSON(http.StatusInternalServerError, map[string]string{
                "error": "internal_error",
                "message": result.Error.Error(),
            })
        }
        
        // Return 202 Accepted
        return c.JSON(http.StatusAccepted, map[string]interface{}{
            "status":      "accepted",
            "bet_id":      betID,
            "new_balance": result.NewBalance,
        })
        
    case <-time.After(5 * time.Second):
        // Timeout - bet may or may not be processed
        return c.JSON(http.StatusGatewayTimeout, map[string]string{
            "error":   "timeout",
            "message": "Bet status unknown, check history",
            "bet_id":  betID,
        })
    }
}
```

---

### Task 2.2: WebSocket Confirmation
**Owner:** Backend Engineer  
**Estimated:** 3 hours  
**Source ADR:** ADR-005 (Optimistic UI)  
**Source C4:** WebSocket Server

**Description:**
Implement WebSocket notification when bet is confirmed.

**DoD:**
- [ ] WebSocket server subscribes to bet confirmations
- [ ] Channel: `user:{user_id}:bets`
- [ ] Broadcast `bet:confirmed` event
- [ ] Frontend updates grid state

**Implementation:**
```go
// cmd/ws/hub.go
package main

// After batcher confirms bet, publish to Redis
type BetConfirmation struct {
    BetID      string `json:"bet_id"`
    UserID     string `json:"user_id"`
    CellID     string `json:"cell_id"`
    Status     string `json:"status"`
    NewBalance int64  `json:"new_balance"`
}

func (b *Batcher) notifyConfirmation(bet PendingBet, result BetResult) {
    if !result.Success {
        return
    }
    
    confirmation := BetConfirmation{
        BetID:      bet.BetID,
        UserID:     bet.UserID,
        CellID:     bet.CellID,
        Status:     result.Status,
        NewBalance: result.NewBalance,
    }
    
    data, _ := json.Marshal(confirmation)
    channel := fmt.Sprintf("user:%s:bets", bet.UserID)
    b.redis.Publish(context.Background(), channel, data)
}
```

---

## Task 3: Grid State Management

### Task 3.1: Implement Grid Cache
**Owner:** Backend Engineer  
**Estimated:** 3 hours  
**Source ADR:** PRD (Grid betting rules)  
**Source C4:** Redis (grid cache)

**Description:**
Cache grid state in Redis for fast cell availability checks.

**DoD:**
- [ ] Store occupied cells per user
- [ ] Check cell availability before bet submission
- [ ] TTL-based expiration for past columns

**Implementation:**
```go
// pkg/grid/grid.go
package grid

type GridCache struct {
    redis *redis.Client
}

func (g *GridCache) IsCellAvailable(ctx context.Context, userID, cellID string) (bool, error) {
    key := fmt.Sprintf("user_cells:%s", userID)
    // Check if cell is in user's occupied set
    exists, err := g.redis.SIsMember(ctx, key, cellID).Result()
    return !exists, err
}

func (g *GridCache) MarkCellOccupied(ctx context.Context, userID, cellID string, ttl time.Duration) error {
    key := fmt.Sprintf("user_cells:%s", userID)
    pipe := g.redis.Pipeline()
    pipe.SAdd(ctx, key, cellID)
    pipe.Expire(ctx, key, ttl)
    _, err := pipe.Exec(ctx)
    return err
}
```

---

## Task 4: Testing

### Task 4.1: Batcher Unit Tests
**Owner:** Backend Engineer  
**Estimated:** 4 hours  
**Source ADR:** ADR-005, ADR-009, ADR-010  
**Source C4:** Bet Batcher

**Description:**
Write comprehensive tests for batcher logic.

**DoD:**
- [ ] Batch flush on size trigger
- [ ] Batch flush on timeout
- [ ] Insufficient funds handling
- [ ] Concurrent bet submission
- [ ] Transaction rollback on error
- [ ] Ledger entry creation verified

---

### Task 4.2: Double-Spend Prevention Test
**Owner:** Backend Engineer  
**Estimated:** 3 hours  
**Source ADR:** ADR-009  
**Source C4:** Bet Batcher

**Description:**
Verify double-spend cannot occur with concurrent requests.

**DoD:**
- [ ] Simulate 100 concurrent bets from same user
- [ ] Verify balance never goes negative
- [ ] Verify sum of bets <= initial balance

**Test:**
```go
func TestDoubleSpendPrevention(t *testing.T) {
    // Setup: User with $100 balance
    // Launch 100 goroutines trying to bet $10 each
    // Only 10 should succeed
    // Balance should be exactly $0 after
}
```

---

### Task 4.3: Integration Load Test
**Owner:** Backend Engineer  
**Estimated:** 3 hours  
**Source ADR:** ADR-005 (3K RPS target)  
**Source C4:** Full flow

**Description:**
Verify system handles 3K RPS with batching.

**DoD:**
- [ ] 3,000 concurrent requests/second
- [ ] p99 latency < 100ms
- [ ] No SQLITE_BUSY errors
- [ ] All bets processed or correctly rejected

---

## Module Deliverables

- [ ] Bet batcher with size/time triggers
- [ ] Conditional SQL balance deduction
- [ ] Hybrid transaction ledger (same transaction)
- [ ] Bet placement API endpoint
- [ ] WebSocket confirmation publishing
- [ ] Grid state caching
- [ ] Unit and integration tests
- [ ] Double-spend prevention verified
- [ ] 3K RPS load test passing

---

## Exit Criteria

This module is complete when:
1. Bets are batched and flushed correctly
2. Conditional deduction prevents overdrafts
3. Ledger entries are created atomically with bets
4. WebSocket confirmations are sent
5. Double-spend is impossible (verified by test)
6. 3K RPS sustained with <100ms p99 latency
7. Zero SQLITE_BUSY errors

---

## Financial Integrity Checklist

- [ ] Conditional UPDATE with `balance >= amount`
- [ ] RowsAffected checked
- [ ] Ledger written in same transaction
- [ ] `balance_after` snapshot recorded
- [ ] Redis sync is post-commit and best-effort
- [ ] Daily integrity check query exists
