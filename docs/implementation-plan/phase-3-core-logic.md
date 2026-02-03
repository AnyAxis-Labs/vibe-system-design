# Phase 3: Core Logic (Days 6-9)

**Goal:** Implement complete betting, resolution, and archival functionality. Feature-complete system.

---

## Day 6: Betting Flow & Grid UI

### 6.1 Complete Bet Placement Flow
**Owner:** Backend  
**Estimated:** 4 hours  
**Source:** ADR-005, ADR-009, ADR-010

**Tasks:**
Complete the bet batcher with full logic:

```go
func (b *BetBatcher) flush(bets []PendingBet) {
    tx := b.db.Begin()
    defer tx.Rollback()
    
    for _, bet := range bets {
        // 1. Conditional balance deduction
        var newBalance int64
        err := tx.QueryRow(`
            UPDATE users 
            SET balance = balance - ?, version = version + 1
            WHERE id = ? AND balance >= ?
            RETURNING balance
        `, bet.Amount, bet.UserID, bet.Amount).Scan(&newBalance)
        
        if err == sql.ErrNoRows {
            // Insufficient funds
            tx.Exec(`INSERT INTO bets (...) VALUES (...,'insufficient_funds')`)
            b.notifyFailure(bet)
            continue
        }
        
        // 2. Write to ledger (ADR-010)
        tx.Exec(`
            INSERT INTO transaction_ledger 
            (id, user_id, amount, balance_after, ref_type, ref_id)
            VALUES (?, ?, ?, ?, 'BET', ?)
        `, generateULID(), bet.UserID, -bet.Amount, newBalance, bet.BetID)
        
        // 3. Record bet
        tx.Exec(`
            INSERT INTO bets (id, user_id, cell_id, amount, reward_rate, status)
            VALUES (?, ?, ?, ?, ?, 'confirmed')
        `, bet.BetID, bet.UserID, bet.CellID, bet.Amount, bet.RewardRate)
        
        b.notifySuccess(bet, newBalance)
    }
    
    tx.Commit()
    b.updateRedisCache(bets)
}
```

**DoD:**
- [ ] Conditional deduction prevents overdraft
- [ ] Ledger entry written for each bet
- [ ] WebSocket notification sent
- [ ] Redis cache updated (best-effort)

---

### 6.2 Grid UI Implementation
**Owner:** Frontend  
**Estimated:** 4 hours  
**Source:** PRD Functional Requirements

**Tasks:**
- Create 2D grid component (X: time columns, Y: price ranges)
- Color cells based on availability (past/current = disabled)
- Show reward rates per cell (from API)
- Implement tap-to-bet interaction
- Show bet size selector (presets)

**Grid Rules:**
- Cells in past: disabled
- Current column: disabled
- Within 1 column ahead: disabled
- All other cells: enabled with reward rate

**DoD:**
- [ ] Grid renders correctly
- [ ] Cell availability rules enforced
- [ ] Reward rates displayed
- [ ] Tap interaction works

---

### 6.3 Pending/Confirmed State Management
**Owner:** Frontend  
**Estimated:** 3 hours  
**Source:** UX Spec (Hardening 002)

**Tasks:**
Implement strict state machine:

```typescript
type BetState = 
  | { status: 'idle' }
  | { status: 'pending'; betId: string; cellId: string }
  | { status: 'confirmed'; betId: string; cellId: string }
  | { status: 'failed'; error: string }
  | { status: 'unknown'; betId: string }; // Disconnect while pending
```

- On tap: Show "Pending" (pulsing border)
- On WebSocket `bet:confirmed`: Show "Confirmed" (solid)
- On WebSocket `bet:failed`: Show "Failed" with error
- On disconnect while pending: Show "Unknown", prompt refresh

**DoD:**
- [ ] Pending state visible (pulsing)
- [ ] Confirmed state solid
- [ ] Failed state shows error
- [ ] Unknown state on disconnect

---

## Day 7: Bet Resolution & Payouts

### 7.1 Bet Resolution Worker
**Owner:** Backend  
**Estimated:** 4 hours  
**Source:** PRD Bet Resolution Logic

**Tasks:**
Implement resolution algorithm:

```go
func (r *Resolver) Run(ctx context.Context) {
    ticker := time.NewTicker(10 * time.Second)
    
    for range ticker.C {
        // Find bets ready for resolution
        bets := r.findPendingBets(time.Now())
        
        for _, bet := range bets {
            // Query price history for the time window
            prices := r.getPriceHistory(bet.TargetTimeStart, bet.TargetTimeEnd)
            
            // Check if price entered cell range
            won := false
            for _, price := range prices {
                if price >= bet.CellMinPrice && price <= bet.CellMaxPrice {
                    won = true
                    break
                }
            }
            
            if won {
                payout := int64(float64(bet.Amount) * bet.RewardRate)
                r.resolveWin(bet, payout)
            } else {
                r.resolveLoss(bet)
            }
        }
    }
}
```

**DoD:**
- [ ] Polls for resolvable bets
- [ ] Queries price history from Redis
- [ ] Correctly identifies wins/losses
- [ ] Updates bet status

---

### 7.2 Win/Loss Processing
**Owner:** Backend  
**Estimated:** 3 hours  
**Source:** ADR-010

**Tasks:**
Implement payout logic:

```go
func (r *Resolver) resolveWin(bet Bet, payout int64) {
    tx := r.db.Begin()
    
    // 1. Credit balance
    tx.Exec(`
        UPDATE users SET balance = balance + ? WHERE id = ?
    `, payout, bet.UserID)
    
    // 2. Write ledger entry (ADR-010)
    tx.Exec(`
        INSERT INTO transaction_ledger 
        (id, user_id, amount, balance_after, ref_type, ref_id)
        SELECT ?, ?, ?, balance, 'PAYOUT', ?
        FROM users WHERE id = ?
    `, generateULID(), bet.UserID, payout, bet.ID, bet.UserID)
    
    // 3. Update bet status
    tx.Exec(`
        UPDATE bets 
        SET status = 'won', resolved_at = ?, payout_amount = ?
        WHERE id = ?
    `, time.Now(), payout, bet.ID)
    
    tx.Commit()
    
    // Notify user
    r.ws.Notify(bet.UserID, BetResolvedEvent{BetID: bet.ID, Won: true, Payout: payout})
}
```

**DoD:**
- [ ] Winners receive payout
- [ ] Ledger entry for payout
- [ ] Bet status updated to 'won'/'lost'
- [ ] WebSocket notification sent

---

### 7.3 Price History Storage
**Owner:** Backend  
**Estimated:** 2 hours  
**Source:** PRD Market Data

**Tasks:**
- Store prices in Redis sorted set: `prices:btc:usd:ts`
- Score = timestamp, Value = price
- Set TTL to 7 days
- Implement query function: `getPrices(start, end)`

**DoD:**
- [ ] Prices stored with timestamp
- [ ] 7-day TTL applied
- [ ] Range queries work
- [ ] Old prices auto-expire

---

## Day 8: Archival & Integrity

### 8.1 Weekly Archiver Implementation
**Owner:** Backend  
**Estimated:** 4 hours  
**Source:** ADR-006

**Tasks:**
Implement archiver:

```bash
#!/bin/bash
# archive-weekly.sh

WEEK=$(date -d 'last week' +%Y_w%U)
DB_FILE="/data/sqlite/bets_${WEEK}.db"
ARCHIVE_FILE="/tmp/bets_${WEEK}.db.zst"

# 1. VACUUM INTO
sqlite3 /data/sqlite/tap_trading.db \
    "VACUUM INTO '${DB_FILE}'"

# 2. Compress
zstd -19 -T4 -o "${ARCHIVE_FILE}" "${DB_FILE}"

# 3. Upload to S3
aws s3 cp "${ARCHIVE_FILE}" \
    s3://tap-trading-archives/bets/ \
    --storage-class STANDARD_IA

# 4. Record metadata
sqlite3 /data/sqlite/tap_trading.db \
    "INSERT INTO archive_metadata (week, s3_path, size_bytes) 
     VALUES ('${WEEK}', 's3://tap-trading-archives/bets/bets_${WEEK}.db.zst', 
     $(stat -c%s ${ARCHIVE_FILE}))"

# 5. Update metrics (dead man's switch)
echo "last_successful_archive_timestamp $(date +%s)" | \
    curl -X POST localhost:9091/metrics/job/archiver

# 6. Cleanup (keep 4 weeks local)
find /data/sqlite/bets_*.db -mtime +28 -delete
```

**DoD:**
- [ ] VACUUM INTO creates clean copy
- [ ] zstd compression works
- [ ] S3 upload succeeds
- [ ] Metadata recorded
- [ ] Old files purged

---

### 8.2 Database Integrity Checks
**Owner:** Backend  
**Estimated:** 2 hours  
**Source:** ADR-010

**Tasks:**
Implement daily integrity check:

```sql
-- verify-balances.sql
WITH ledger_balances AS (
    SELECT user_id, SUM(amount) as ledger_sum
    FROM transaction_ledger
    GROUP BY user_id
)
SELECT 
    u.id,
    u.balance as user_balance,
    COALESCE(lb.ledger_sum, 0) as ledger_balance,
    u.balance - COALESCE(lb.ledger_sum, 0) as drift
FROM users u
LEFT JOIN ledger_balances lb ON u.id = lb.user_id
WHERE u.balance != COALESCE(lb.ledger_sum, 0)
   OR (u.balance = 0 AND lb.ledger_sum IS NULL AND EXISTS (
       SELECT 1 FROM transaction_ledger WHERE user_id = u.id
   ));
```

Create cron job: `0 3 * * * /opt/app/scripts/integrity-check.sh`

**DoD:**
- [ ] Query detects any drift
- [ ] Alert sent if drift found
- [ ] Can recalculate balance from ledger

---

### 8.3 Read Replica Attachment
**Owner:** Backend  
**Estimated:** 2 hours  
**Source:** C4 Container (SQLite Historical)

**Tasks:**
- Implement `AttachHistoricalDB(week string)` function
- Manage read-only connections to historical partitions
- Query routing: current week → RW, historical → RO

**DoD:**
- [ ] Can attach historical DB files
- [ ] Read queries work on historical data
- [ ] Write queries blocked on historical

---

## Day 9: Frontend Completion & Integration

### 9.1 Balance & History Views
**Owner:** Frontend  
**Estimated:** 3 hours  
**Source:** PRD

**Tasks:**
- Create balance display component
- Create bet history table:
  - Columns: Time, Cell, Amount, Status, Payout
  - Filter: All, Pending, Won, Lost
  - Pagination
- Show transaction ledger (advanced view)

**DoD:**
- [ ] Balance displays correctly
- [ ] History loads from API
- [ ] Filtering works
- [ ] Pagination implemented

---

### 9.2 WebSocket Reconnect Jitter
**Owner:** Frontend  
**Estimated:** 2 hours  
**Source:** Hardening 001

**Tasks:**
Implement exponential backoff with jitter:

```typescript
function getReconnectDelay(attempt: number): number {
    const base = Math.min(1000 * Math.pow(2, attempt), 30000); // Max 30s
    const jitter = Math.random() * 5000; // 0-5s random
    return base + jitter;
}
```

- Track reconnect attempts
- Apply jitter delay before reconnecting
- Reset counter on successful connection

**DoD:**
- [ ] Reconnects with delay
- [ ] Jitter prevents thundering herd
- [ ] Resets on success

---

### 9.3 Admin Dashboard
**Owner:** Frontend  
**Estimated:** 3 hours  
**Source:** C4 Context (Admin)

**Tasks:**
Create simple admin view:
- System health status
- Current price feed age
- Circuit breaker state
- Recent error logs
- Manual archive trigger button

**DoD:**
- [ ] Health status visible
- [ ] Price age displayed
- [ ] CB state shown
- [ ] Can trigger manual archive

---

## Phase 3 Exit Criteria

- [ ] Complete bet placement with ledger write
- [ ] Bet resolution working end-to-end
- [ ] Weekly archival automated
- [ ] Integrity checks running
- [ ] Frontend shows grid, balance, history
- [ ] WebSocket reconnect with jitter
- [ ] Admin dashboard functional

---

## Traceability Matrix (Phase 3)

| Task | ADR Reference | PRD/C4 Reference |
|------|---------------|------------------|
| 6.1 Bet Placement | ADR-005, ADR-009, ADR-010 | Bet Placement |
| 6.2 Grid UI | - | Grid Interface |
| 6.3 Pending States | Hardening 002 | UX Spec |
| 7.1 Resolution Worker | - | Bet Resolution |
| 7.2 Win/Loss | ADR-010 | Payout Logic |
| 7.3 Price History | - | Market Data |
| 8.1 Archiver | ADR-006 | Archival |
| 8.2 Integrity | ADR-010 | Audit |
| 8.3 Read Replica | - | C4 SQLite RO |
| 9.1 History | - | User History |
| 9.2 Reconnect | Hardening 001 | Resilience |
| 9.3 Admin | - | C4 Admin |
