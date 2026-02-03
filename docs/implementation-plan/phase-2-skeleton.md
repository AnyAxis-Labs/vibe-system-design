# Phase 2: Skeleton (Days 4-5)

**Goal:** Establish thin-thread connectivity between all components. End-to-end "hello world" flow working.

---

## Day 4: Backend Skeleton Services

### 4.1 API Server Skeleton
**Owner:** Backend  
**Estimated:** 4 hours  
**Source:** C4 Container (API Server), ADR-002

**Tasks:**
- Implement basic Echo/Fiber server
- Add middleware: logging, recovery, CORS
- Create health endpoint: `GET /health`
  ```json
  {"status": "healthy", "checks": {"database": "ok", "redis": "ok"}}
  ```
- Create placeholder endpoints:
  - `POST /api/auth/nonce` - Generate nonce for signing
  - `POST /api/auth/verify` - Verify signature (placeholder)
  - `POST /api/bets` - Place bet (returns 202, no logic)
  - `GET /api/balances` - Get balance (placeholder)
- Add graceful shutdown

**DoD:**
- [ ] Server starts on :8080
- [ ] Health endpoint returns 200
- [ ] Logs requests in JSON format
- [ ] Graceful shutdown works (SIGTERM)

---

### 4.2 WebSocket Server Skeleton
**Owner:** Backend  
**Estimated:** 3 hours  
**Source:** C4 Container (WebSocket Server)

**Tasks:**
- Implement WebSocket server (gorilla/websocket or nhooyr)
- Handle connection upgrade at `/ws`
- Implement connection management (add/remove clients)
- Broadcast message to all clients (test endpoint)
- Add ping/pong keepalive

**DoD:**
- [ ] WS server starts on :8081
- [ ] Can connect via `wscat` or browser
- [ ] Broadcast reaches all connected clients
- [ ] Disconnections handled cleanly

---

### 4.3 Bet Batcher Skeleton
**Owner:** Backend  
**Estimated:** 4 hours  
**Source:** ADR-005, ADR-009, ADR-010

**Tasks:**
Implement channel-based batcher:

```go
type BetBatcher struct {
    channel chan PendingBet
    db      *sql.DB
    batchSize int
    timeout   time.Duration
}

type PendingBet struct {
    BetID    string
    UserID   string
    CellID   string
    Amount   int64
    Done     chan BetResult
}
```

- Create buffered channel (10,000 capacity)
- Implement batch accumulation (size OR timeout)
- Stub transaction logic (BEGIN, inserts, COMMIT)
- Return result via Done channel

**DoD:**
- [ ] Batcher accepts bets via channel
- [ ] Flushes on batch size (100) or timeout (100ms)
- [ ] Returns success/failure via Done channel
- [ ] No SQLite contention (single writer)

---

### 4.4 Price Ingestion Skeleton
**Owner:** Backend  
**Estimated:** 3 hours  
**Source:** C4 Container (Price Ingestion), ADR-008

**Tasks:**
- Connect to Binance WebSocket: `wss://stream.binance.com:9443/ws/btcusdt@trade`
- Parse trade messages (extract price)
- Publish to Redis pub/sub channel: `prices:btc:usd`
- Store in Redis time-series (sorted set with TTL)
- Handle reconnection with exponential backoff

**DoD:**
- [ ] Receives price ticks from Binance
- [ ] Publishes to Redis
- [ ] WebSocket subscribers receive prices
- [ ] Reconnects on disconnect

---

## Day 5: Connectivity & Auth Integration

### 5.1 Signature Validator
**Owner:** Backend  
**Estimated:** 3 hours  
**Source:** ADR-007

**Tasks:**
Implement EIP-191 signature verification:

```go
func (v *Validator) ValidateSignature(
    message string, 
    signature string, 
    expectedAddress string,
) error {
    // Recover address from signature
    // Verify matches expectedAddress
    // Check timestamp freshness (30s window)
    // Check nonce uniqueness (Redis SET NX, 90s TTL)
}
```

- Use `go-ethereum/crypto` for recovery
- Verify timestamp in message (±30s)
- Check/store nonce in Redis (SET NX EX 90)
- Reject replayed signatures

**DoD:**
- [ ] Valid signatures accepted
- [ ] Invalid signatures rejected
- [ ] Old timestamps rejected
- [ ] Reused nonces rejected

---

### 5.2 Circuit Breaker
**Owner:** Backend  
**Estimated:** 2 hours  
**Source:** ADR-008

**Tasks:**
Implement health monitoring:

```go
type CircuitBreaker struct {
    status Status // HEALTHY, DEGRADED, CRITICAL
    lastPriceTime time.Time
    lastHealthyTime time.Time // For hysteresis
}
```

- Subscribe to Redis price channel
- Track `lastPriceTime`
- Evaluate status:
  - >10s stale → CRITICAL
  - 3-10s stale → DEGRADED
  - <3s → HEALTHY (only after 30s stable)
- Middleware rejects bets when CRITICAL

**DoD:**
- [ ] Status updates correctly
- [ ] Enters CRITICAL at >10s
- [ ] Recovers only after 30s stable
- [ ] Middleware blocks/allows bets correctly

---

### 5.3 Frontend → Backend Integration
**Owner:** Frontend (Lead), Backend (Support)  
**Estimated:** 4 hours  
**Source:** C4 Relationships

**Tasks:**
- Configure API client (axios) with base URL
- Configure WebSocket connection
- Implement message signing flow:
  1. Generate UUID nonce
  2. Create message with timestamp
  3. Sign with wallet
  4. Send to `/api/auth/verify`
- Display connection status

**DoD:**
- [ ] Can call `/health` and get response
- [ ] WebSocket connects and receives prices
- [ ] Can sign and send auth payload
- [ ] Connection status visible in UI

---

### 5.4 End-to-End Thin Thread
**Owner:** Both  
**Estimated:** 3 hours  
**Source:** General

**Tasks:**
Complete flow:
```
1. User opens app → sees grid
2. WebSocket connects → receives BTC price
3. User connects wallet → sees balance
4. User taps cell → "Pending" state
5. Frontend signs message → POST /api/bets
6. API validates → submits to batcher
7. Batcher flushes → writes to SQLite
8. WebSocket confirms → "Confirmed" state
```

**DoD:**
- [ ] Full flow works end-to-end
- [ ] Logs visible at each step
- [ ] No 500 errors
- [ ] Data persists in SQLite

---

## Phase 2 Exit Criteria

- [ ] API server handles requests
- [ ] WebSocket broadcasts prices
- [ ] Bet batcher flushes to SQLite
- [ ] Signature validation works
- [ ] Circuit breaker tracks health
- [ ] Frontend connects to all backends
- [ ] End-to-end thin thread functional

---

## Traceability Matrix (Phase 2)

| Task | ADR Reference | C4 Container/Relationship |
|------|---------------|---------------------------|
| 4.1 API Server | ADR-002 | API Server |
| 4.2 WebSocket | - | WebSocket Server |
| 4.3 Batcher | ADR-005, ADR-009, ADR-010 | Bet Batcher |
| 4.4 Price Ingest | ADR-008 | Price Ingestion |
| 5.1 Validator | ADR-007 | Signature Validator |
| 5.2 Circuit Breaker | ADR-008 | Circuit Breaker |
| 5.3 FE/BE Integration | - | All Relationships |
| 5.4 E2E Thread | - | Full System |
