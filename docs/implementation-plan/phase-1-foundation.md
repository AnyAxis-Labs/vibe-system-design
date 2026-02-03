# Phase 1: Foundation (Days 1-3)

**Goal:** Provision infrastructure, set up CI/CD, establish database schema, create project skeleton.

---

## Day 1: Infrastructure & VM Setup

### 1.1 Provision Single VM
**Owner:** Backend  
**Estimated:** 4 hours  
**Source:** C4 Container (Single VM), ADR-003

**Tasks:**
- Provision VM: 4 vCPU, 8GB RAM, 256GB SSD
- Configure Ubuntu 22.04 LTS
- Set up SSH keys, firewall (ufw), fail2ban
- Create directory structure: `/opt/app`, `/data`, `/var/log/tap-trading`

**DoD:**
- [ ] VM accessible via SSH
- [ ] Disk mounted at `/data` (SSD)
- [ ] Basic security hardening applied

---

### 1.2 Install Nginx
**Owner:** Backend  
**Estimated:** 2 hours  
**Source:** C4 Container (Nginx)

**Tasks:**
- Install Nginx: `apt install nginx`
- Configure reverse proxy:
  - `/api/*` → `localhost:8080`
  - `/ws/*` → `localhost:8081` (WebSocket upgrade)
  - `/` → static files
- Configure TLS (Let's Encrypt or self-signed for dev)
- Set rate limiting: `limit_req_zone` (10r/s burst)
- Set connection limits: `limit_conn` (max 1000 concurrent)

**DoD:**
- [ ] Nginx running, ports 80/443 open
- [ ] Reverse proxy routes configured
- [ ] Rate limiting tested with `ab` or `wrk`

---

### 1.3 Install & Configure Redis
**Owner:** Backend  
**Estimated:** 3 hours  
**Source:** ADR-004, C4 Container (Redis)

**Tasks:**
- Install Redis 7: `apt install redis-server`
- Configure `/etc/redis/redis.conf`:
  ```
  maxmemory 512mb
  maxmemory-policy allkeys-lru
  appendonly yes
  appendfsync everysec
  save 900 1
  save 300 10
  save 60 10000
  ```
- Configure persistence (AOF + RDB)
- Test pub/sub functionality

**DoD:**
- [ ] Redis running on port 6379
- [ ] Persistence verified (AOF file growing)
- [ ] Memory policy tested

---

### 1.4 Configure SQLite Environment
**Owner:** Backend  
**Estimated:** 2 hours  
**Source:** ADR-001, ADR-006

**Tasks:**
- Install SQLite 3: `apt install sqlite3`
- Create data directory: `/data/sqlite`
- Set permissions for app user
- Configure WAL mode defaults
- Create backup directory: `/data/backups`

**DoD:**
- [ ] SQLite CLI available
- [ ] `/data/sqlite` directory exists with correct permissions
- [ ] WAL mode understood (PRAGMA journal_mode=WAL)

---

## Day 2: Go Project Scaffold & Database Schema

### 2.1 Go Project Structure
**Owner:** Backend  
**Estimated:** 3 hours  
**Source:** ADR-002

**Tasks:**
```
cmd/
  api/          # API server main
  ws/           # WebSocket server main
  price-ingest/ # Price ingestion service main
  resolver/     # Bet resolution worker main
  archiver/     # Weekly archiver main
internal/
  api/          # HTTP handlers
  batcher/      # Bet batcher (channel-based)
  auth/         # Web3 signature validation
  circuitbreaker/ # Circuit breaker middleware
  config/       # Configuration management
  db/           # Database migrations & queries
  models/       # Domain models
  resolver/     # Bet resolution logic
  websocket/    # WebSocket handling
pkg/
  logger/       # Structured logging (zerolog)
  metrics/      # Prometheus metrics
```

- Initialize Go module: `go mod init tap-trading`
- Add dependencies: `echo`, `gorilla/websocket`, `redis/go-redis`, `mattn/go-sqlite3`
- Create `Makefile` with build targets

**DoD:**
- [ ] Project structure in place
- [ ] `go build` succeeds for all binaries
- [ ] Makefile targets working

---

### 2.2 Database Schema & Migrations
**Owner:** Backend  
**Estimated:** 4 hours  
**Source:** ADR-001, ADR-009, ADR-010

**Tasks:**
Create `internal/db/migrations/001_initial_schema.sql`:

```sql
-- Users table
CREATE TABLE users (
    id TEXT PRIMARY KEY,
    wallet_address TEXT UNIQUE NOT NULL,
    balance INTEGER NOT NULL DEFAULT 0,
    version INTEGER NOT NULL DEFAULT 1,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Transaction Ledger (ADR-010)
CREATE TABLE transaction_ledger (
    id TEXT PRIMARY KEY,
    user_id TEXT NOT NULL,
    amount INTEGER NOT NULL,
    balance_after INTEGER NOT NULL,
    ref_type TEXT NOT NULL CHECK (ref_type IN ('BET', 'PAYOUT', 'DEPOSIT', 'WITHDRAWAL')),
    ref_id TEXT NOT NULL,
    metadata JSON,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id)
);

CREATE INDEX idx_ledger_user_time ON transaction_ledger(user_id, created_at);
CREATE INDEX idx_ledger_ref ON transaction_ledger(ref_type, ref_id);

-- Bets table
CREATE TABLE bets (
    id TEXT PRIMARY KEY,
    user_id TEXT NOT NULL,
    cell_id TEXT NOT NULL,
    amount INTEGER NOT NULL,
    reward_rate REAL NOT NULL,
    status TEXT NOT NULL CHECK (status IN ('pending', 'confirmed', 'won', 'lost', 'insufficient_funds')),
    placed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    resolved_at TIMESTAMP,
    payout_amount INTEGER,
    FOREIGN KEY (user_id) REFERENCES users(id)
);

CREATE INDEX idx_bets_user ON bets(user_id);
CREATE INDEX idx_bets_status_time ON bets(status, placed_at) WHERE status = 'pending';
CREATE INDEX idx_bets_cell ON bets(cell_id);

-- Price history (for resolution)
CREATE TABLE price_history (
    timestamp INTEGER PRIMARY KEY,
    price INTEGER NOT NULL
);

-- Metadata for archival
CREATE TABLE archive_metadata (
    week TEXT PRIMARY KEY,
    s3_path TEXT NOT NULL,
    size_bytes INTEGER NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Enable WAL mode
PRAGMA journal_mode = WAL;
PRAGMA foreign_keys = ON;
```

Create migration runner in Go.

**DoD:**
- [ ] Migrations apply cleanly
- [ ] Schema validated with `.schema`
- [ ] Indexes verified with `.indexes`

---

### 2.3 Configuration Management
**Owner:** Backend  
**Estimated:** 2 hours  
**Source:** General

**Tasks:**
Create `config.yaml`:
```yaml
server:
  api_port: 8080
  ws_port: 8081
  
database:
  path: /data/sqlite/tap_trading.db
  max_open_conns: 20
  max_idle_conns: 5
  
redis:
  addr: localhost:6379
  password: ""
  db: 0
  
batcher:
  batch_size: 100
  batch_timeout_ms: 100
  channel_buffer: 10000
  
auth:
  timestamp_window_sec: 30
  nonce_ttl_sec: 90
  
circuit_breaker:
  degraded_threshold_sec: 3
  critical_threshold_sec: 10
  recovery_stability_sec: 30
  
price_feed:
  source: "wss://stream.binance.com:9443/ws/btcusdt@trade"
  reconnect_interval_sec: 5
  
archival:
  s3_bucket: "tap-trading-archives"
  s3_region: "us-east-1"
  local_retention_weeks: 4
```

**DoD:**
- [ ] Config loads from YAML
- [ ] Environment variable overrides work
- [ ] Config validation implemented

---

## Day 3: Shared Libraries & Frontend Scaffold

### 3.1 Shared Go Libraries
**Owner:** Backend  
**Estimated:** 4 hours  
**Source:** General

**Tasks:**
- **Logger:** Structured JSON logging (zerolog)
  ```go
  logger := zerolog.New(os.Stdout).With().Timestamp().Logger()
  ```
- **Metrics:** Prometheus metrics exporter
  - Request rate/latency
  - Batch queue depth
  - Circuit breaker state
  - Price feed age
- **Database:** Connection pool management
- **Redis:** Client wrapper with retry logic

**DoD:**
- [ ] Logger outputs structured JSON
- [ ] `/metrics` endpoint returns Prometheus format
- [ ] Database connection pool configured

---

### 3.2 Systemd Service Files
**Owner:** Backend  
**Estimated:** 2 hours  
**Source:** ADR-003

**Tasks:**
Create service files:
- `/etc/systemd/system/tap-trading-api.service`
- `/etc/systemd/system/tap-trading-ws.service`
- `/etc/systemd/system/tap-trading-price.service`
- `/etc/systemd/system/tap-trading-resolver.service`
- `/etc/systemd/system/tap-trading-archiver.service`

Configure:
- Auto-restart on failure
- Log to journald
- Resource limits (MemoryMax, CPUQuota)

**DoD:**
- [ ] Services start/stop with systemd
- [ ] Auto-restart tested (kill -9)
- [ ] Logs visible in `journalctl`

---

### 3.3 Frontend Project Scaffold
**Owner:** Frontend  
**Estimated:** 4 hours  
**Source:** C4 Container (Frontend SPA)

**Tasks:**
```
frontend/
  src/
    components/
      Grid/           # Betting grid component
      Wallet/         # Web3 wallet connection
      BetStatus/      # Pending/Confirmed states
    hooks/
      useWebSocket.ts
      useAuth.ts
    services/
      api.ts          # REST API client
      websocket.ts    # WebSocket manager
    stores/
      userStore.ts
      gridStore.ts
    utils/
      signature.ts    # EIP-191 signing
      constants.ts
  public/
  package.json
  vite.config.ts
```

- Initialize with Vite (React or Vue)
- Add dependencies: `ethers`, `axios`, `zustand` (or Pinia)
- Configure TypeScript
- Create `.env` template

**DoD:**
- [ ] `npm run dev` starts dev server
- [ ] TypeScript compiles without errors
- [ ] Build outputs to `dist/` directory

---

### 3.4 Web3 Integration Setup
**Owner:** Frontend  
**Estimated:** 2 hours  
**Source:** ADR-007

**Tasks:**
- Install ethers.js: `npm install ethers`
- Create wallet connection component
- Test MetaMask/WalletConnect integration
- Implement `personal_sign` for EIP-191

**DoD:**
- [ ] Can connect/disconnect wallet
- [ ] Can sign messages
- [ ] Address displayed correctly

---

## Phase 1 Exit Criteria

- [ ] VM provisioned with Nginx, Redis, SQLite
- [ ] Go project builds 5 binaries (api, ws, price-ingest, resolver, archiver)
- [ ] Database migrations apply cleanly
- [ ] Frontend dev server running
- [ ] Systemd service files created (not yet enabled)
- [ ] Configuration management implemented

---

## Traceability Matrix (Phase 1)

| Task | ADR Reference | C4 Container |
|------|---------------|--------------|
| 1.1 VM Provision | ADR-003 | Single VM |
| 1.2 Nginx | - | Nginx |
| 1.3 Redis | ADR-004 | Redis |
| 1.4 SQLite | ADR-001 | SQLite |
| 2.1 Go Scaffold | ADR-002 | - |
| 2.2 Schema | ADR-009, ADR-010 | SQLite |
| 2.3 Config | - | - |
| 3.1 Shared Libs | - | - |
| 3.2 Systemd | ADR-003 | - |
| 3.3 Frontend | - | Frontend SPA |
| 3.4 Web3 | ADR-007 | - |
