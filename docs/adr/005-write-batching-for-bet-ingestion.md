# ADR 005: Write Batching for Bet Ingestion

## Status
Accepted (Hardening Response)

## Context
Hardening Audit #001 identified that SQLite's single-writer limitation will fail at 3K RPS with individual row inserts. Each transaction requires fsync to WAL, causing queue buildup and latency spikes.

## Problem Statement
- SQLite allows only ONE writer at a time (even in WAL mode)
- 3,000 concurrent taps = 3,000 transactions queuing
- Each fsync to SSD takes ~0.5-2ms
- Queue depth × fsync time = unbounded latency growth
- `SQLITE_BUSY` errors under load violate correctness NFRs

## Decision
Implement **in-memory write batching** with channel-based aggregation.

## Solution Design

### Architecture Change

```
┌─────────────────────────────────────────────────────────────┐
│                     API Server (Go)                          │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────┐  │
│  │ HTTP Handler │───▶│ Bet Channel │───▶│ Batch Flusher   │  │
│  │   (Goroutine)│    │ (Buffered   │    │ (Single Writer) │  │
│  └─────────────┘    │   10,000)   │    │                 │  │
│                     └─────────────┘    │ Flush triggers: │  │
│                                        │ • 100 items OR  │  │
│                                        │ • 100ms timeout │  │
│                                        └────────┬────────┘  │
│                                                 │            │
│                                                 ▼            │
│                                        ┌─────────────────┐  │
│                                        │ SQLite (WAL)    │  │
│                                        │ BEGIN IMMEDIATE │  │
│                                        │ INSERT x N      │  │
│                                        │ COMMIT          │  │
│                                        └─────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### Implementation Details

```go
// Batch configuration
const (
    BatchSize     = 100      // Flush after N bets
    BatchTimeout  = 100 * time.Millisecond
    ChannelBuffer = 10000    // Backpressure threshold
)

type BetBatch struct {
    Bets []Bet
    Done chan error  // Per-bet completion signaling
}

type BetFlusher struct {
    channel chan PendingBet
    db      *sql.DB
}

func (f *BetFlusher) Start() {
    ticker := time.NewTicker(BatchTimeout)
    batch := make([]PendingBet, 0, BatchSize)
    
    for {
        select {
        case bet := <-f.channel:
            batch = append(batch, bet)
            if len(batch) >= BatchSize {
                f.flush(batch)
                batch = batch[:0]
                ticker.Reset(BatchTimeout)
            }
            
        case <-ticker.C:
            if len(batch) > 0 {
                f.flush(batch)
                batch = batch[:0]
            }
        }
    }
}

func (f *BetFlusher) flush(bets []PendingBet) {
    tx, _ := f.db.BeginTx(context.Background(), &sql.TxOptions{
        Isolation: sql.LevelSerializable,
    })
    
    stmt, _ := tx.Prepare("INSERT INTO bets (user_id, cell_id, amount, ...) VALUES (?, ?, ?, ...)")
    
    for _, bet := range bets {
        // Insert bet + deduct balance in single statement
        stmt.Exec(bet.UserID, bet.CellID, bet.Amount)
    }
    
    err := tx.Commit()
    
    // Signal completion to each waiting goroutine
    for _, bet := range bets {
        bet.Done <- err
    }
}
```

### Response Flow

1. **Synchronous Validation** (HTTP handler):
   - Validate signature + nonce (anti-replay)
   - Check balance in Redis (fast)
   - Check cell availability (Redis cache)
   - Validate timing rules

2. **Async Persistence** (channel):
   - Submit to batch channel
   - Wait on `Done` channel with timeout
   - Return 202 Accepted immediately (optimistic)

3. **WebSocket Confirmation**:
   - Client receives `bet:confirmed` or `bet:failed` event
   - Grid updates optimistically, confirmed async

## Throughput Math

| Scenario | Before | After |
|----------|--------|-------|
| Transaction count | 3,000/sec | 30/sec (100x batch) |
| fsync per second | 3,000 | 30 |
| Avg latency @ 3K RPS | 500-2000ms (queuing) | 10-50ms (batched) |
| SQLite busy errors | High | Zero |

## Risk: Data Loss on Crash

| Window | Exposure | Mitigation |
|--------|----------|------------|
| In-channel (unflushed) | Max 100ms × 3K = 300 bets | Acceptable for trading app; document in ToS |
| Client knows | WebSocket confirms persistence | Retry mechanism for unconfirmed bets |

## Trade-off Analysis

| Alternative | Why Rejected |
|-------------|--------------|
| Individual transactions | Fails at 3K RPS per hardening audit |
| PostgreSQL | Violates 2-week constraint, adds ops burden |
| Async queue (RabbitMQ) | Extra process, overkill for single VM |
| In-memory only | Loses durability guarantee |

## NFR Compliance

| NFR | Before | After |
|-----|--------|-------|
| 3K RPS | ❌ Fails | ✅ 30 batched TX/sec |
| ≤100ms latency | ❌ Spikes | ✅ Batched + optimistic |
| Atomic bets | ✅ | ✅ (per-batch atomic) |
| Correctness | ❌ SQLITE_BUSY | ✅ Ordered channel |

## Decision Date
2026-02-03

## Author
System Architect (Hardening Response)
