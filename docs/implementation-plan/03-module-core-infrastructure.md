# Module 1: Core Infrastructure (Vertical)

**Goal:** Implement the foundational data layer and shared components that all other modules depend on.

**Duration:** Days 3-4  
**Dependencies:** Phase 1 (Foundation), Phase 2 (Skeleton) complete  
**Priority:** CRITICAL (blocks all other modules)

---

## Module Scope

This module implements:
- Complete database layer with transaction support
- Redis caching and pub/sub with proper error handling
- Configuration management
- Structured logging with context propagation
- Domain models and shared types

**Source ADRs:** ADR-001 (SQLite), ADR-004 (Redis), ADR-002 (Go)  
**Source C4:** SQLite (RW/RO), Redis containers

---

## Task 1: Database Layer (ADR-001)

### Task 1.1: Implement SQLite Connection Pool
**Owner:** Backend Engineer  
**Estimated:** 4 hours  
**Source ADR:** ADR-001 (SQLite as Primary Database)  
**Source C4:** SQLite (Current Week)

**Description:**
Implement production-ready SQLite client with connection pooling, WAL support, and prepared statement caching.

**DoD:**
- [ ] Connection pool with configurable max open connections (10-20)
- [ ] WAL mode enforcement on connection open
- [ ] Prepared statement cache for hot queries
- [ ] Connection health check
- [ ] Retry logic with exponential backoff for SQLITE_BUSY
- [ ] Context cancellation support

**Implementation:**
```go
// pkg/db/sqlite.go
package db

import (
    "context"
    "database/sql"
    "fmt"
    "time"
    
    _ "github.com/mattn/go-sqlite3"
)

type Config struct {
    Path           string
    MaxOpenConns   int
    MaxIdleConns   int
    ConnMaxLifetime time.Duration
    BusyTimeout    time.Duration
}

type Client struct {
    db *sql.DB
}

func New(cfg Config) (*Client, error) {
    dsn := fmt.Sprintf("%s?_journal_mode=WAL&_synchronous=NORMAL&_busy_timeout=%d",
        cfg.Path, cfg.BusyTimeout.Milliseconds())
    
    db, err := sql.Open("sqlite3", dsn)
    if err != nil {
        return nil, fmt.Errorf("open database: %w", err)
    }
    
    db.SetMaxOpenConns(cfg.MaxOpenConns)
    db.SetMaxIdleConns(cfg.MaxIdleConns)
    db.SetConnMaxLifetime(cfg.ConnMaxLifetime)
    
    // Verify WAL mode
    var journalMode string
    if err := db.QueryRow("PRAGMA journal_mode").Scan(&journalMode); err != nil {
        return nil, fmt.Errorf("check journal mode: %w", err)
    }
    if journalMode != "wal" {
        return nil, fmt.Errorf("WAL mode not enabled, got: %s", journalMode)
    }
    
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    if err := db.PingContext(ctx); err != nil {
        return nil, fmt.Errorf("ping database: %w", err)
    }
    
    return &Client{db: db}, nil
}

func (c *Client) Close() error {
    return c.db.Close()
}

func (c *Client) ExecContext(ctx context.Context, query string, args ...interface{}) (sql.Result, error) {
    return c.db.ExecContext(ctx, query, args...)
}

func (c *Client) QueryContext(ctx context.Context, query string, args ...interface{}) (*sql.Rows, error) {
    return c.db.QueryContext(ctx, query, args...)
}

func (c *Client) QueryRowContext(ctx context.Context, query string, args ...interface{}) *sql.Row {
    return c.db.QueryRowContext(ctx, query, args...)
}

func (c *Client) BeginTx(ctx context.Context, opts *sql.TxOptions) (*sql.Tx, error) {
    return c.db.BeginTx(ctx, opts)
}
```

**Tests:**
- [ ] Connection pool limits enforced
- [ ] WAL mode verified
- [ ] SQLITE_BUSY triggers retry
- [ ] Context cancellation interrupts query

---

### Task 1.2: Implement Database Migrations
**Owner:** Backend Engineer  
**Estimated:** 3 hours  
**Source ADR:** ADR-001, ADR-009, ADR-010  
**Source C4:** SQLite schema

**Description:**
Implement migration system for schema versioning and evolution.

**DoD:**
- [ ] Migration runner using `golang-migrate` or custom
- [ ] All schema files in `/opt/app/backend/schema/`
- [ ] Migration version table in database
- [ ] Up/down migrations for each version
- [ ] Migration command in Makefile

**Migration Files:**
```
schema/
├── 001_initial.up.sql
├── 001_initial.down.sql
├── 002_indexes.up.sql
├── 002_indexes.down.sql
└── ...
```

**Migration Runner:**
```go
// pkg/db/migrate.go
func Migrate(db *sql.DB, direction string) error {
    // Implementation using golang-migrate/migrate
    // or custom lightweight version
}
```

---

### Task 1.3: Implement Partition Manager (ADR-006)
**Owner:** Backend Engineer  
**Estimated:** 5 hours  
**Source ADR:** ADR-006 (Archival Strategy)  
**Source C4:** SQLite (Current Week + Historical)

**Description:**
Implement partition management for weekly database rotation.

**DoD:**
- [ ] `PartitionManager` struct tracking current week
- [ ] `ATTACH DATABASE` for historical partitions
- [ ] Query routing based on timestamp ranges
- [ ] Automatic partition discovery on startup
- [ ] Error handling for archived data queries

**Interface:**
```go
// pkg/db/partition.go
type PartitionManager struct {
    currentDB *sql.DB
    attached  map[string]*sql.DB // week -> read-only connection
    currentWeek string
}

func (pm *PartitionManager) QueryBets(ctx context.Context, userID string, from, to time.Time) ([]models.Bet, error)
func (pm *PartitionManager) AttachPartition(week string, dbPath string) error
func (pm *PartitionManager) DetachPartition(week string) error
func (pm *PartitionManager) RotateToNewWeek(newWeek string) error
```

---

## Task 2: Redis Layer (ADR-004)

### Task 2.1: Implement Redis Client
**Owner:** Backend Engineer  
**Estimated:** 3 hours  
**Source ADR:** ADR-004 (Redis for Cache, Sessions, Pub/Sub)  
**Source C4:** Redis container

**Description:**
Implement production-ready Redis client with connection pooling and pub/sub support.

**DoD:**
- [ ] Connection pool configuration
- [ ] Health check and connection retry
- [ ] Pub/Sub subscriber management
- [ ] Pipeline support for batch operations
- [ ] Context cancellation support

**Implementation:**
```go
// pkg/redis/client.go
package redis

import (
    "context"
    "fmt"
    "time"
    
    "github.com/redis/go-redis/v9"
)

type Config struct {
    Addr         string
    Password     string
    DB           int
    PoolSize     int
    MinIdleConns int
    MaxRetries   int
}

type Client struct {
    client *redis.Client
}

func New(cfg Config) (*Client, error) {
    rdb := redis.NewClient(&redis.Options{
        Addr:         cfg.Addr,
        Password:     cfg.Password,
        DB:           cfg.DB,
        PoolSize:     cfg.PoolSize,
        MinIdleConns: cfg.MinIdleConns,
        MaxRetries:   cfg.MaxRetries,
    })
    
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    
    if err := rdb.Ping(ctx).Err(); err != nil {
        return nil, fmt.Errorf("redis ping: %w", err)
    }
    
    return &Client{client: rdb}, nil
}

func (c *Client) Close() error {
    return c.client.Close()
}

func (c *Client) Get(ctx context.Context, key string) *redis.StringCmd {
    return c.client.Get(ctx, key)
}

func (c *Client) Set(ctx context.Context, key string, value interface{}, ttl time.Duration) *redis.StatusCmd {
    return c.client.Set(ctx, key, value, ttl)
}

func (c *Client) SetNX(ctx context.Context, key string, value interface{}, ttl time.Duration) *redis.BoolCmd {
    return c.client.SetNX(ctx, key, value, ttl)
}

func (c *Client) Publish(ctx context.Context, channel string, message interface{}) *redis.IntCmd {
    return c.client.Publish(ctx, channel, message)
}

func (c *Client) Subscribe(ctx context.Context, channels ...string) *redis.PubSub {
    return c.client.Subscribe(ctx, channels...)
}

func (c *Client) Pipeline() redis.Pipeliner {
    return c.client.Pipeline()
}
```

---

### Task 2.2: Implement Price Cache
**Owner:** Backend Engineer  
**Estimated:** 2 hours  
**Source ADR:** ADR-004  
**Source C4:** Redis (price storage)

**Description:**
Implement price caching with time-series storage in Redis sorted sets.

**DoD:**
- [ ] Store latest price with TTL
- [ ] Append price to sorted set with timestamp
- [ ] Query price history by time range
- [ ] Automatic pruning (Redis TTL handles expiration)

**Interface:**
```go
// pkg/redis/price.go
func (c *Client) SetPrice(ctx context.Context, symbol string, price float64) error
func (c *Client) GetLatestPrice(ctx context.Context, symbol string) (float64, error)
func (c *Client) GetPriceHistory(ctx context.Context, symbol string, from, to time.Time) ([]PricePoint, error)
```

---

## Task 3: Shared Components

### Task 3.1: Configuration Management
**Owner:** Backend Engineer  
**Estimated:** 2 hours  
**Source ADR:** All  
**Source C4:** All containers

**Description:**
Implement environment-based configuration with validation.

**DoD:**
- [ ] Struct-based config with env tags
- [ ] Validation for required fields
- [ ] Sensible defaults
- [ ] Documentation of all config options

**Config Struct:**
```go
// pkg/config/config.go
type Config struct {
    LogLevel string `env:"LOG_LEVEL" envDefault:"info"`
    
    Database struct {
        Path           string        `env:"DB_PATH" envDefault:"/data/bets.db"`
        MaxOpenConns   int           `env:"DB_MAX_OPEN_CONNS" envDefault:"20"`
        MaxIdleConns   int           `env:"DB_MAX_IDLE_CONNS" envDefault:"5"`
        ConnMaxLifetime time.Duration `env:"DB_CONN_MAX_LIFETIME" envDefault:"1h"`
        BusyTimeout    time.Duration `env:"DB_BUSY_TIMEOUT" envDefault:"10s"`
    }
    
    Redis struct {
        Addr         string `env:"REDIS_ADDR" envDefault:"localhost:6379"`
        Password     string `env:"REDIS_PASSWORD"`
        DB           int    `env:"REDIS_DB" envDefault:"0"`
        PoolSize     int    `env:"REDIS_POOL_SIZE" envDefault:"10"`
        MinIdleConns int    `env:"REDIS_MIN_IDLE_CONNS" envDefault:"2"`
    }
    
    // ... other config sections
}

func Load() (*Config, error)
```

---

### Task 3.2: Structured Logging
**Owner:** Backend Engineer  
**Estimated:** 2 hours  
**Source ADR:** All (Observability)  
**Source C4:** All containers

**Description:**
Implement structured JSON logging with request context propagation.

**DoD:**
- [ ] zerolog setup with configurable level
- [ ] Request ID injection middleware
- [ ] Component field for all logs
- [ ] Error stack trace capture

**Implementation:**
```go
// pkg/logger/logger.go
package logger

import (
    "os"
    "github.com/rs/zerolog"
)

type Logger struct {
    logger zerolog.Logger
}

func New(level string) Logger {
    l := zerolog.New(os.Stdout).With().Timestamp().Logger()
    // Set level...
    return Logger{logger: l}
}

func (l Logger) WithComponent(name string) Logger {
    return Logger{logger: l.logger.With().Str("component", name).Logger()}
}

func (l Logger) WithRequestID(id string) Logger {
    return Logger{logger: l.logger.With().Str("request_id", id).Logger()}
}

// Middleware for Echo
func Middleware(log Logger) echo.MiddlewareFunc
```

---

### Task 3.3: Domain Models
**Owner:** Backend Engineer  
**Estimated:** 2 hours  
**Source ADR:** ADR-009, ADR-010  
**Source C4:** Data models

**Description:**
Define domain models for users, bets, and ledger entries.

**DoD:**
- [ ] User model with balance
- [ ] Bet model with status enum
- [ ] LedgerEntry model
- [ ] Validation methods on models
- [ ] JSON marshaling/unmarshaling

**Models:**
```go
// pkg/models/models.go
package models

import "time"

type User struct {
    ID            string    `json:"id" db:"id"`
    WalletAddress string    `json:"wallet_address" db:"wallet_address"`
    Balance       int64     `json:"balance" db:"balance"`
    Version       int       `json:"version" db:"version"`
    CreatedAt     time.Time `json:"created_at" db:"created_at"`
    UpdatedAt     time.Time `json:"updated_at" db:"updated_at"`
}

type BetStatus string

const (
    BetStatusPending           BetStatus = "pending"
    BetStatusConfirmed         BetStatus = "confirmed"
    BetStatusWon               BetStatus = "won"
    BetStatusLost              BetStatus = "lost"
    BetStatusInsufficientFunds BetStatus = "insufficient_funds"
)

type Bet struct {
    ID           string    `json:"id" db:"id"`
    UserID       string    `json:"user_id" db:"user_id"`
    CellID       string    `json:"cell_id" db:"cell_id"`
    Amount       int64     `json:"amount" db:"amount"`
    RewardRate   float64   `json:"reward_rate" db:"reward_rate"`
    Status       BetStatus `json:"status" db:"status"`
    PlacedAt     time.Time `json:"placed_at" db:"placed_at"`
    ResolvedAt   *time.Time `json:"resolved_at,omitempty" db:"resolved_at"`
    PayoutAmount *int64     `json:"payout_amount,omitempty" db:"payout_amount"`
}

type LedgerEntry struct {
    ID           string                 `json:"id" db:"id"`
    UserID       string                 `json:"user_id" db:"user_id"`
    Amount       int64                  `json:"amount" db:"amount"` // Negative for debit
    BalanceAfter int64                  `json:"balance_after" db:"balance_after"`
    RefType      string                 `json:"ref_type" db:"ref_type"`
    RefID        string                 `json:"ref_id" db:"ref_id"`
    Metadata     map[string]interface{} `json:"metadata,omitempty" db:"metadata"`
    CreatedAt    time.Time              `json:"created_at" db:"created_at"`
}
```

---

## Task 4: Integration Testing

### Task 4.1: Module Integration Tests
**Owner:** Backend Engineer  
**Estimated:** 3 hours  
**Source ADR:** All  
**Source C4:** All containers

**Description:**
Write integration tests for the complete module.

**DoD:**
- [ ] SQLite client integration tests
- [ ] Redis client integration tests
- [ ] Partition manager tests
- [ ] Config loading tests
- [ ] All tests pass in CI

**Test Commands:**
```bash
cd /opt/app/backend
go test -v ./pkg/db/... -tags=integration
go test -v ./pkg/redis/... -tags=integration
go test -v ./pkg/config/...
```

---

## Module Deliverables

- [ ] SQLite client with WAL, pooling, retry logic
- [ ] Migration system with all schema files
- [ ] Partition manager for weekly rotation
- [ ] Redis client with pub/sub support
- [ ] Configuration management
- [ ] Structured logging
- [ ] Domain models (User, Bet, LedgerEntry)
- [ ] Integration tests passing

---

## Exit Criteria

This module is complete when:
1. Database layer handles concurrent access without SQLITE_BUSY errors
2. Redis pub/sub can broadcast messages across services
3. All models serialize/deserialize correctly
4. Integration tests pass reliably
5. Other modules can import and use these packages
