# Phase 1: Foundation (Horizontal)

**Goal:** Establish shared infrastructure, CI/CD, and scaffolding that enables all downstream work.

**Duration:** Days 1-2

---

## 1.1 VM Provisioning & System Setup

### Task 1.1.1: Provision VM and Base OS
**Owner:** Backend Engineer  
**Estimated:** 2 hours  
**Source ADR:** ADR-003 (Monolithic Single-VM Deployment)  
**Source C4:** Single VM boundary

**Description:**
Provision the single VM with required specifications and configure base OS settings.

**DoD:**
- [ ] VM running with 4-core CPU, 8GB RAM, 256GB SSD
- [ ] Ubuntu 22.04 LTS (or compatible) installed
- [ ] SSH access configured with key-based auth
- [ ] Firewall configured (allow 22, 80, 443, 8080, 8081)
- [ ] Automatic security updates enabled

**⚠️ Constraints:**
- Single VM is the only deployment target (no scaling)
- 256GB disk must be monitored (alert at 70%, 85%, 95%)

---

### Task 1.1.2: Install System Dependencies
**Owner:** Backend Engineer  
**Estimated:** 1 hour  
**Source ADR:** ADR-002 (Go Backend)  
**Source C4:** Application Layer

**Description:**
Install Go, SQLite, Redis, Nginx, and supporting tools.

**DoD:**
- [ ] Go 1.21+ installed (`/usr/local/go`)
- [ ] SQLite 3.x installed with FTS5 support
- [ ] Redis 7.x installed
- [ ] Nginx installed
- [ ] zstd compression tools installed
- [ ] AWS CLI installed (for S3 archival)

**Commands:**
```bash
# Go installation
curl -L https://go.dev/dl/go1.21.5.linux-amd64.tar.gz | sudo tar -C /usr/local -xzf -
echo 'export PATH=$PATH:/usr/local/go/bin' | sudo tee /etc/profile.d/go.sh

# Other dependencies
sudo apt update
sudo apt install -y sqlite3 redis-server nginx zstd awscli
```

---

## 1.2 Database Setup (ADR-001)

### Task 1.2.1: Configure SQLite with WAL Mode
**Owner:** Backend Engineer  
**Estimated:** 2 hours  
**Source ADR:** ADR-001 (SQLite as Primary Database)  
**Source C4:** SQLite (Current Week)

**Description:**
Set up SQLite with WAL mode for concurrent read/write, configure connection pooling parameters.

**DoD:**
- [ ] Data directory created: `/data`
- [ ] WAL mode enabled: `PRAGMA journal_mode = WAL`
- [ ] Synchronous mode set: `PRAGMA synchronous = NORMAL`
- [ ] Cache size configured: `PRAGMA cache_size = -64000` (64MB)
- [ ] Auto-vacuum incremental enabled
- [ ] Backup script created at `/opt/app/scripts/backup-hourly.sh`

**Configuration:**
```sql
-- Run on each new database
PRAGMA journal_mode = WAL;
PRAGMA synchronous = NORMAL;
PRAGMA cache_size = -64000;
PRAGMA auto_vacuum = INCREMENTAL;
PRAGMA temp_store = MEMORY;
PRAGMA mmap_size = 268435456; -- 256MB
```

**⚠️ Critical:**
- WAL mode is mandatory for concurrent reads during writes
- Backup script must use `sqlite3 .backup` (not file copy) for consistency

---

### Task 1.2.2: Create Database Schema Templates
**Owner:** Backend Engineer  
**Estimated:** 3 hours  
**Source ADR:** ADR-009, ADR-010 (Conditional Deduction + Hybrid Ledger)  
**Source C4:** SQLite (Current Week)

**Description:**
Create initial schema for users, bets, and transaction_ledger tables.

**DoD:**
- [ ] Schema file created: `/opt/app/schema/001_initial.sql`
- [ ] `users` table with optimistic locking (version column)
- [ ] `bets` table with status tracking
- [ ] `transaction_ledger` table (ADR-010) with indexes
- [ ] Partial index on bets pending status
- [ ] Migration system in place (using golang-migrate or similar)

**Schema:**
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
    ref_type TEXT NOT NULL, -- 'BET', 'PAYOUT', 'DEPOSIT', 'WITHDRAWAL'
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
    status TEXT NOT NULL, -- 'pending', 'confirmed', 'won', 'lost', 'insufficient_funds'
    placed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    resolved_at TIMESTAMP,
    payout_amount INTEGER,
    
    FOREIGN KEY (user_id) REFERENCES users(id)
);
CREATE INDEX idx_bets_pending ON bets(user_id, status) WHERE status = 'pending';
CREATE INDEX idx_bets_user ON bets(user_id, placed_at);
```

---

## 1.3 Redis Setup (ADR-004)

### Task 1.3.1: Configure Redis
**Owner:** Backend Engineer  
**Estimated:** 1.5 hours  
**Source ADR:** ADR-004 (Redis for Cache, Sessions, Pub/Sub)  
**Source C4:** Redis container

**Description:**
Configure Redis with memory limits, persistence, and proper key eviction.

**DoD:**
- [ ] Redis config at `/etc/redis/redis.conf`
- [ ] `maxmemory 512mb` set
- [ ] `maxmemory-policy allkeys-lru` set
- [ ] `appendonly yes` (AOF persistence)
- [ ] `appendfsync everysec` configured
- [ ] Redis started and responding on :6379

**Configuration:**
```conf
# /etc/redis/redis.conf
maxmemory 512mb
maxmemory-policy allkeys-lru
appendonly yes
appendfsync everysec
save 900 1
save 300 10
save 60 10000
```

**⚠️ Key TTLs:**
- Sessions: 24h
- Nonces: 90s (ADR-007 hardening fix)
- Prices: 1h (latest), 7d (history)
- Rate limits: 1m

---

## 1.4 Go Project Scaffolding (ADR-002)

### Task 1.4.1: Create Go Project Structure
**Owner:** Backend Engineer  
**Estimated:** 2 hours  
**Source ADR:** ADR-002 (Go Backend Language)  
**Source C4:** All Go containers

**Description:**
Set up Go module structure, shared libraries, and build configuration.

**DoD:**
- [ ] Go module initialized: `github.com/tap-trading/backend`
- [ ] Directory structure created per layout below
- [ ] Shared packages: `pkg/config`, `pkg/db`, `pkg/redis`, `pkg/logger`
- [ ] Makefile with build targets
- [ ] Dockerfile for each service

**Project Layout:**
```
/opt/app/backend/
├── cmd/
│   ├── api/           # API Server
│   ├── ws/            # WebSocket Server
│   ├── price/         # Price Ingestion + Health Monitor
│   ├── resolver/      # Bet Resolver
│   └── archiver/      # Weekly Archiver
├── pkg/
│   ├── config/        # Configuration management
│   ├── db/            # SQLite abstraction
│   ├── redis/         # Redis client
│   ├── logger/        # Structured logging (zerolog)
│   ├── auth/          # Signature validation (shared)
│   ├── batcher/       # Write batching logic
│   ├── circuit/       # Circuit breaker
│   └── models/        # Domain models
├── internal/
│   └── ...            # Private implementation
├── scripts/
│   ├── backup-hourly.sh
│   └── archive-weekly.sh
├── schema/
│   └── *.sql
├── Makefile
└── go.mod
```

---

### Task 1.4.2: Implement Shared Libraries
**Owner:** Backend Engineer  
**Estimated:** 4 hours  
**Source ADR:** ADR-001, ADR-004 (Database + Redis)  
**Source C4:** All containers

**Description:**
Implement reusable packages for database, Redis, configuration, and logging.

**DoD:**
- [ ] `pkg/config`: Environment-based config with validation
- [ ] `pkg/db`: SQLite connection pool with WAL, prepared statement cache
- [ ] `pkg/redis`: Redis client with connection pooling, pub/sub support
- [ ] `pkg/logger`: Structured JSON logging (zerolog)
- [ ] Unit tests for all packages

**Key Interfaces:**
```go
// pkg/db/db.go
type DB interface {
    QueryContext(ctx context.Context, query string, args ...interface{}) (*sql.Rows, error)
    QueryRowContext(ctx context.Context, query string, args ...interface{}) *sql.Row
    ExecContext(ctx context.Context, query string, args ...interface{}) (sql.Result, error)
    BeginTx(ctx context.Context, opts *sql.TxOptions) (*sql.Tx, error)
    Close() error
}

// pkg/redis/redis.go
type Client interface {
    Get(ctx context.Context, key string) *redis.StringCmd
    Set(ctx context.Context, key string, value interface{}, ttl time.Duration) *redis.StatusCmd
    SetNX(ctx context.Context, key string, value interface{}, ttl time.Duration) *redis.BoolCmd
    Subscribe(ctx context.Context, channels ...string) *redis.PubSub
    Publish(ctx context.Context, channel string, message interface{}) *redis.IntCmd
    Pipeline() redis.Pipeliner
}
```

---

## 1.5 CI/CD Pipeline and Deployment

### Task 1.5.1: CI/CD Pipeline Setup
**Owner:** Backend Engineer  
**Estimated:** 4 hours  
**Source ADR:** ADR-003 (Monolithic Deployment)  
**Source C4:** Deployment flow

**Description:**
Implement CI/CD pipeline with build gates, test gates, and artifact versioning to ensure code quality and reproducible deployments.

---

#### Build Gate (Compilation & Quality)

**DoD:**
- [ ] Go compilation with `-ldflags` for version injection
- [ ] golangci-lint with custom config (security, performance, error handling)
- [ ] go vet, go fmt checks
- [ ] Static analysis for crypto/transaction code
- [ ] Build fails on any warning in critical paths

**Build Gate Configuration (`.golangci.yml`):**
```yaml
run:
  timeout: 5m
  issues-exit-code: 1

linters:
  enable:
    - gosec        # Security
    - staticcheck  # Static analysis
    - gosimple     # Simplification
    - ineffassign  # Ineffective assignments
    - errcheck     # Unchecked errors
    - govet        # Vet
    - typecheck    # Type checking
    - unused       # Unused code

linters-settings:
  gosec:
    excludes:
      - G104  # Allow unchecked errors in non-critical paths (checked in critical paths)
  
issues:
  exclude-rules:
    - path: _test\.go
      linters:
        - gosec
```

**Build Script (`scripts/build-gate.sh`):**
```bash
#!/bin/bash
set -euo pipefail

echo "=== Build Gate ==="

# Format check
echo "→ Checking formatting..."
if [ -n "$(go fmt ./...)" ]; then
    echo "ERROR: Code is not formatted. Run 'go fmt ./...'"
    exit 1
fi

# Vet
echo "→ Running go vet..."
go vet ./...

# Lint
echo "→ Running golangci-lint..."
golangci-lint run --config .golangci.yml

# Security scan on critical paths
echo "→ Security scan on transaction code..."
gosec -include=G104,G201,G202,G203,G301,G302,G303,G304,G305,G306,G307,G401,G402,G403,G404,G501,G502,G503,G504,G505,G601 ./pkg/batcher/... ./pkg/auth/...

echo "✓ Build gate passed"
```

---

#### Test Gate (Quality Assurance)

**DoD:**
- [ ] Unit tests with race detector (`-race`)
- [ ] Integration tests with SQLite and Redis
- [ ] Coverage threshold: 80% for critical paths (batcher, auth)
- [ ] Benchmark tests for hot paths (3K RPS simulation)
- [ ] Test artifacts stored

**Test Gate Configuration:**
```bash
#!/bin/bash
# scripts/test-gate.sh
set -euo pipefail

echo "=== Test Gate ==="

# Unit tests with race detector
echo "→ Running unit tests..."
go test -v -race -coverprofile=coverage.out ./...

# Coverage check for critical paths
echo "→ Checking coverage..."
go tool cover -func=coverage.out | awk '
/^(pkg\/(batcher|auth)\/)/ { 
    if ($3 < 80) { 
        print "FAIL: Coverage below 80% in " $1 ": " $3 
        exit 1 
    }
}'

# Integration tests (requires running DB/Redis)
echo "→ Running integration tests..."
go test -v -tags=integration ./pkg/db/... ./pkg/redis/...

# Benchmark tests
echo "→ Running benchmarks..."
go test -bench=. -benchtime=10s ./pkg/batcher/... > benchmark.txt

echo "✓ Test gate passed"
```

**Test Coverage Requirements:**

| Package | Minimum Coverage |
|---------|------------------|
| `pkg/batcher` | 80% |
| `pkg/auth` | 80% |
| `pkg/db` | 70% |
| `pkg/redis` | 70% |
| Other | 60% |

---

#### Artifact Versioning

**DoD:**
- [ ] Semantic versioning (v1.2.3-build+commit)
- [ ] Git commit SHA embedded in binary
- [ ] Build timestamp embedded
- [ ] Version endpoint in each service (`/version`)
- [ ] Versioned artifacts stored in `/opt/app/releases/`

**Versioning Strategy:**
```bash
# Version format: v{MAJOR}.{MINOR}.{PATCH}-{build}+{git_sha}
# Example: v1.3.0-42+7d2f1a3b

VERSION=$(git describe --tags --always --dirty)
BUILD_TIME=$(date -u +%Y%m%d%H%M%S)
GIT_COMMIT=$(git rev-parse --short HEAD)
GO_VERSION=$(go version | awk '{print $3}')

# Build with version injection
go build -ldflags "\
    -X main.Version=${VERSION} \
    -X main.BuildTime=${BUILD_TIME} \
    -X main.GitCommit=${GIT_COMMIT} \
    -X main.GoVersion=${GO_VERSION} \
    -s -w \
" -o bin/api ./cmd/api
```

**Version Endpoint:**
```go
// In each service
func (s *Server) VersionHandler(c echo.Context) error {
    return c.JSON(http.StatusOK, map[string]string{
        "version":    Version,
        "build_time": BuildTime,
        "git_commit": GitCommit,
        "go_version": GoVersion,
    })
}
```

**Artifact Storage:**
```
/opt/app/releases/
├── v1.3.0/
│   ├── api
│   ├── ws
│   ├── price
│   ├── resolver
│   ├── archiver
│   ├── checksums.sha256
│   └── manifest.json
├── v1.3.1/
│   └── ...
└── current -> v1.3.0  # Symlink to current version
```

**Manifest Format:**
```json
{
  "version": "v1.3.0-42+7d2f1a3b",
  "build_time": "20260203093045",
  "git_commit": "7d2f1a3b",
  "artifacts": {
    "api": {
      "path": "api",
      "sha256": "a1b2c3d4..."
    },
    "ws": {
      "path": "ws", 
      "sha256": "e5f6g7h8..."
    }
  },
  "built_by": "ci-server-01"
}
```

---

#### Deployment Pipeline

**DoD:**
- [ ] Blue-green or rolling deployment strategy
- [ ] Pre-deployment health check
- [ ] Automated rollback on failure
- [ ] Post-deployment smoke tests

**Deployment Script (`scripts/deploy.sh`):**
```bash
#!/bin/bash
set -euo pipefail

VERSION=${1:-current}
RELEASE_DIR="/opt/app/releases/${VERSION}"

echo "=== Deploying ${VERSION} ==="

# Verify artifacts exist
if [ ! -d "${RELEASE_DIR}" ]; then
    echo "ERROR: Release ${VERSION} not found"
    exit 1
fi

# Pre-deployment health check
echo "→ Checking current health..."
if ! curl -sf http://localhost:8080/healthz > /dev/null; then
    echo "WARNING: Current deployment is unhealthy"
fi

# Backup current
BACKUP_DIR="/opt/app/releases/.backup-$(date +%Y%m%d%H%M%S)"
cp -r /opt/app/current "${BACKUP_DIR}"

# Update symlink
echo "→ Updating to ${VERSION}..."
ln -sfn "${RELEASE_DIR}" /opt/app/current

# Restart services
echo "→ Restarting services..."
systemctl restart tap-trading-api tap-trading-ws tap-trading-price

# Wait for services
echo "→ Waiting for services..."
sleep 5

# Post-deployment smoke tests
echo "→ Running smoke tests..."
for i in {1..10}; do
    if curl -sf http://localhost:8080/healthz > /dev/null; then
        echo "✓ API healthy"
        break
    fi
    if [ $i -eq 10 ]; then
        echo "ERROR: Deployment failed health check"
        echo "→ Rolling back..."
        ln -sfn "${BACKUP_DIR}" /opt/app/current
        systemctl restart tap-trading-api tap-trading-ws
        exit 1
    fi
    sleep 2
done

echo "✓ Deployment successful"
```

---

### Task 1.5.2: Systemd Service Definitions
**Owner:** Backend Engineer  
**Estimated:** 2 hours  
**Source ADR:** ADR-003  
**Source C4:** All Go containers

**Description:**
Create systemd service files for all Go binaries with proper dependency ordering.

**DoD:**
- [ ] Service files for api, ws, price, resolver, archiver
- [ ] Dependency ordering (api/ws after redis, price after network)
- [ ] Auto-restart with exponential backoff
- [ ] Structured logging to journald
- [ ] Resource limits (CPU: 50%, Memory: 1GB per service)

**Example Service:**
```ini
# /etc/systemd/system/tap-trading-api.service
[Unit]
Description=BTC Tap Trading API Server (v%i)
After=network.target redis.service sqlite.service
Requires=redis.service

[Service]
Type=simple
User=app
Group=app
WorkingDirectory=/opt/app/current
ExecStart=/opt/app/current/api
Restart=always
RestartSec=5
StartLimitInterval=60
StartLimitBurst=3

# Resource limits
CPUQuota=50%
MemoryLimit=1G

# Environment
Environment=LOG_LEVEL=info
Environment=DB_PATH=/data/bets.db
Environment=REDIS_ADDR=localhost:6379

# Logging
StandardOutput=journal
StandardError=journal
SyslogIdentifier=tap-trading-api

[Install]
WantedBy=multi-user.target
```

---

### Task 1.5.2: Configure Nginx
**Owner:** Backend Engineer  
**Estimated:** 1.5 hours  
**Source ADR:** ADR-003 (Monolithic Deployment)  
**Source C4:** Nginx container

**Description:**
Configure Nginx as reverse proxy with rate limiting and connection limits.

**DoD:**
- [ ] Nginx config at `/etc/nginx/sites-available/tap-trading`
- [ ] Static file serving for frontend
- [ ] Reverse proxy to API (:8080) and WebSocket (:8081)
- [ ] Rate limiting: 100 req/min per IP
- [ ] Connection limiting: 1000 concurrent per IP
- [ ] WebSocket upgrade headers configured

**Configuration:**
```nginx
limit_req_zone $binary_remote_addr zone=api:10m rate=100r/m;
limit_conn_zone $binary_remote_addr zone=addr:10m;

server {
    listen 80;
    server_name _;
    
    # Static files
    location / {
        root /opt/app/frontend/dist;
        try_files $uri $uri/ /index.html;
    }
    
    # API
    location /api/ {
        limit_req zone=api burst=20 nodelay;
        limit_conn addr 100;
        proxy_pass http://localhost:8080/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
    
    # WebSocket
    location /ws {
        limit_req zone=api burst=50 nodelay;
        proxy_pass http://localhost:8081;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
    }
}
```

---

## 1.6 Observability Baseline

### Task 1.6.1: Set Up Logging and Metrics
**Owner:** Backend Engineer  
**Estimated:** 2 hours  
**Source ADR:** All (Observability requirement in PRD)  
**Source C4:** All containers

**Description:**
Implement structured logging and basic metrics collection.

**DoD:**
- [ ] zerolog configured for JSON output
- [ ] Log rotation configured (logrotate)
- [ ] Prometheus metrics endpoint scaffolded
- [ ] Request ID middleware for tracing
- [ ] Health check endpoint pattern defined

**Log Format:**
```json
{
  "level": "info",
  "time": "2026-02-03T12:00:00Z",
  "request_id": "req_abc123",
  "component": "api",
  "message": "bet placed",
  "user_id": "0x...",
  "bet_id": "bet_xyz",
  "amount": 1000
}
```

---

## Phase 1 Deliverables Checklist

### Infrastructure
- [ ] VM provisioned and accessible
- [ ] Go 1.21+, SQLite, Redis, Nginx installed
- [ ] SQLite WAL mode configured
- [ ] Database schema templates created
- [ ] Redis configured with 512MB limit and AOF
- [ ] Nginx configured with rate limiting

### Project Structure
- [ ] Go project structure established
- [ ] Shared libraries (db, redis, config, logger) implemented
- [ ] Logging and metrics baseline ready

### CI/CD Pipeline
- [ ] **Build Gate**: golangci-lint, go vet, go fmt, security scan configured
- [ ] **Test Gate**: Unit tests with race detector, coverage thresholds defined
- [ ] **Artifact Versioning**: Semantic versioning, version injection, `/version` endpoint
- [ ] **Deployment Pipeline**: Automated deploy with smoke tests and rollback
- [ ] Build, test, and deploy scripts created
- [ ] systemd service files with resource limits

---

## Exit Criteria for Phase 1

Phase 1 is complete when:
1. All infrastructure is provisioned and configurable
2. A new team member can clone the repo and run `make build` successfully
3. **Build gate passes** (lint, vet, fmt, security scan)
4. **Test gate passes** (unit tests with coverage threshold)
5. **Version endpoint returns** correct version info (`GET /version`)
6. Database connections can be established from a test program
7. Redis is accepting connections with proper persistence
8. Nginx serves static files and proxies to upstream (even if upstream doesn't exist yet)
9. **CI/CD pipeline can deploy a versioned artifact end-to-end**
