# Module 5: Resolution & Archival (Vertical)

**Goal:** Implement bet resolution worker and automated weekly archival.

**Duration:** Days 8-9  
**Dependencies:** Module 1 (Core), Module 4 (Price Feed)  
**Priority:** MEDIUM (background processes)

---

## Module Scope

This module implements:
- Bet resolution worker (scans expired bets, determines win/loss)
- Payout processing with ledger entries
- Weekly archival to S3 (ADR-006)
- Data integrity checking

**Source ADRs:** ADR-006 (Archival Strategy), ADR-010 (Ledger for payouts)  
**Source C4:** Bet Resolver, Weekly Archiver, SQLite, S3

---

## Task 1: Bet Resolution

### Task 1.1: Implement Resolution Worker
**Owner:** Backend Engineer  
**Estimated:** 5 hours  
**Source ADR:** PRD (Bet resolution logic)  
**Source C4:** Bet Resolver container

**Description:**
Implement background worker that scans and resolves expired bets.

**DoD:**
- [ ] Poll for pending bets past their target time
- [ ] Query price history for resolution window
- [ ] Determine win/loss based on price range
- [ ] Update bet status and payout
- [ ] Create ledger entry for payouts
- [ ] Increment user balance for wins

**Implementation:**
```go
// cmd/resolver/resolver.go
package main

import (
    "context"
    "database/sql"
    "fmt"
    "time"
    
    "github.com/oklog/ulid/v2"
    "github.com/tap-trading/backend/pkg/db"
    "github.com/tap-trading/backend/pkg/price"
    "github.com/tap-trading/backend/pkg/redis"
)

const (
    PollInterval = 10 * time.Second
    BatchSize    = 100
)

type Resolver struct {
    db      *db.Client
    redis   *redis.Client
    price   *price.HistoryService
    health  *price.HealthMonitor
}

func (r *Resolver) Start(ctx context.Context) {
    ticker := time.NewTicker(PollInterval)
    defer ticker.Stop()
    
    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            if err := r.resolveBatch(ctx); err != nil {
                log.Error().Err(err).Msg("resolution batch failed")
            }
        }
    }
}

func (r *Resolver) resolveBatch(ctx context.Context) error {
    // Query pending bets that have passed their target time
    // Note: This is simplified - actual query needs to consider cell time windows
    query := `
        SELECT id, user_id, cell_id, amount, reward_rate, placed_at
        FROM bets
        WHERE status = 'confirmed'
        AND placed_at < datetime('now', '-5 minutes')
        LIMIT ?
    `
    
    rows, err := r.db.QueryContext(ctx, query, BatchSize)
    if err != nil {
        return fmt.Errorf("query bets: %w", err)
    }
    defer rows.Close()
    
    var bets []models.Bet
    for rows.Next() {
        var bet models.Bet
        if err := rows.Scan(&bet.ID, &bet.UserID, &bet.CellID, &bet.Amount, &bet.RewardRate, &bet.PlacedAt); err != nil {
            continue
        }
        bets = append(bets, bet)
    }
    
    for _, bet := range bets {
        if err := r.resolveBet(ctx, bet); err != nil {
            log.Error().
                Err(err).
                Str("bet_id", bet.ID).
                Msg("resolve bet failed")
        }
    }
    
    return nil
}

func (r *Resolver) resolveBet(ctx context.Context, bet models.Bet) error {
    // Don't resolve if price feed is unhealthy
    if !r.health.IsHealthy() {
        return fmt.Errorf("price feed unhealthy, skipping resolution")
    }
    
    // Parse cell_id to get time window and price range
    // Cell format: "{time_col}_{min_price}_{max_price}"
    cellInfo, err := parseCellID(bet.CellID)
    if err != nil {
        return fmt.Errorf("parse cell: %w", err)
    }
    
    // Query price history for the cell's time window
    hit, err := r.price.DidPriceHitRange(
        ctx,
        cellInfo.TimeStart,
        cellInfo.TimeEnd,
        cellInfo.MinPrice,
        cellInfo.MaxPrice,
    )
    if err != nil {
        return fmt.Errorf("query price history: %w", err)
    }
    
    tx, err := r.db.BeginTx(ctx, &sql.TxOptions{
        Isolation: sql.LevelSerializable,
    })
    if err != nil {
        return fmt.Errorf("begin tx: %w", err)
    }
    defer tx.Rollback()
    
    now := time.Now()
    
    if hit {
        // WIN: Calculate payout and credit user
        payout := int64(float64(bet.Amount) * bet.RewardRate)
        
        // Update user balance
        _, err := tx.ExecContext(ctx, `
            UPDATE users 
            SET balance = balance + ?, updated_at = ?
            WHERE id = ?
        `, payout, now, bet.UserID)
        if err != nil {
            return fmt.Errorf("credit balance: %w", err)
        }
        
        // Get new balance for ledger
        var newBalance int64
        err = tx.QueryRowContext(ctx, `SELECT balance FROM users WHERE id = ?`, bet.UserID).Scan(&newBalance)
        if err != nil {
            return fmt.Errorf("get balance: %w", err)
        }
        
        // Create payout ledger entry
        ledgerID := ulid.Make().String()
        _, err = tx.ExecContext(ctx, `
            INSERT INTO transaction_ledger 
            (id, user_id, amount, balance_after, ref_type, ref_id, metadata, created_at)
            VALUES (?, ?, ?, ?, ?, ?, ?, ?)
        `, ledgerID, bet.UserID, payout, newBalance, "PAYOUT", bet.ID, nil, now)
        if err != nil {
            return fmt.Errorf("insert ledger: %w", err)
        }
        
        // Update bet status
        _, err = tx.ExecContext(ctx, `
            UPDATE bets 
            SET status = ?, resolved_at = ?, payout_amount = ?
            WHERE id = ?
        `, models.BetStatusWon, now, payout, bet.ID)
        if err != nil {
            return fmt.Errorf("update bet: %w", err)
        }
        
        log.Info().
            Str("bet_id", bet.ID).
            Str("user_id", bet.UserID).
            Int64("payout", payout).
            Msg("bet won")
    } else {
        // LOSS
        _, err := tx.ExecContext(ctx, `
            UPDATE bets 
            SET status = ?, resolved_at = ?
            WHERE id = ?
        `, models.BetStatusLost, now, bet.ID)
        if err != nil {
            return fmt.Errorf("update bet: %w", err)
        }
        
        log.Info().
            Str("bet_id", bet.ID).
            Str("user_id", bet.UserID).
            Msg("bet lost")
    }
    
    if err := tx.Commit(); err != nil {
        return fmt.Errorf("commit: %w", err)
    }
    
    // Update Redis cache (best effort)
    if hit {
        r.redis.IncrBy(ctx, fmt.Sprintf("balance:%s", bet.UserID), payout)
    }
    
    return nil
}

type CellInfo struct {
    TimeStart time.Time
    TimeEnd   time.Time
    MinPrice  float64
    MaxPrice  float64
}

func parseCellID(cellID string) (*CellInfo, error) {
    // Parse "{col}_{min}_{max}" format
    // Implementation depends on grid definition
    return nil, fmt.Errorf("not implemented")
}
```

---

### Task 1.2: Resolution Read Replica
**Owner:** Backend Engineer  
**Estimated:** 2 hours  
**Source ADR:** ADR-006 (Historical queries)  
**Source C4:** SQLite (Historical)

**Description:**
Use read-only replica for historical bet queries during resolution.

**DoD:**
- [ ] Query historical bets from attached read-only DB
- [ ] Covering index on bets table
- [ ] Eliminate read-write contention

**Index:**
```sql
CREATE INDEX idx_bets_resolution ON bets(target_time, status) WHERE status = 'confirmed';
```

---

## Task 2: Weekly Archival (ADR-006)

### Task 2.1: Implement Archiver
**Owner:** Backend Engineer  
**Estimated:** 5 hours  
**Source ADR:** ADR-006 (Archival Strategy)  
**Source C4:** Weekly Archiver, S3

**Description:**
Implement automated weekly archival to S3 with compression.

**DoD:**
- [ ] Identify week to archive
- [ ] VACUUM INTO optimized copy
- [ ] zstd compress with -19 level
- [ ] Upload to S3 STANDARD_IA
- [ ] Verify upload
- [ ] Update metadata
- [ ] Attach as read-only
- [ ] Purge old local files (>4 weeks)

**Implementation:**
```go
// cmd/archiver/archiver.go
package main

import (
    "context"
    "fmt"
    "os"
    "os/exec"
    "path/filepath"
    "time"
    
    "github.com/aws/aws-sdk-go-v2/aws"
    "github.com/aws/aws-sdk-go-v2/config"
    "github.com/aws/aws-sdk-go-v2/service/s3"
)

const (
    DataDir     = "/data"
    ArchiveDir  = "/data/archives"
    S3Bucket    = "tap-trading-archives"
    S3Prefix    = "bets/"
    RetainWeeks = 4
)

type Archiver struct {
    s3Client *s3.Client
}

func (a *Archiver) Run(ctx context.Context, week string) error {
    // 1. Freeze: VACUUM INTO
    sourceDB := filepath.Join(DataDir, "bets_current.db")
    weekDB := filepath.Join(DataDir, fmt.Sprintf("bets_%s.db", week))
    
    log.Info().Str("week", week).Msg("starting archival")
    
    cmd := exec.CommandContext(ctx, "sqlite3", sourceDB, fmt.Sprintf("VACUUM INTO '%s'", weekDB))
    if err := cmd.Run(); err != nil {
        return fmt.Errorf("vacuum: %w", err)
    }
    
    // 2. Compress
    archiveFile := filepath.Join(ArchiveDir, fmt.Sprintf("bets_%s.db.zst", week))
    cmd = exec.CommandContext(ctx, "zstd", "-19", "-T4", "-o", archiveFile, weekDB)
    if err := cmd.Run(); err != nil {
        return fmt.Errorf("compress: %w", err)
    }
    
    // 3. Upload to S3
    file, err := os.Open(archiveFile)
    if err != nil {
        return fmt.Errorf("open archive: %w", err)
    }
    defer file.Close()
    
    stat, _ := file.Stat()
    key := fmt.Sprintf("%sbets_%s.db.zst", S3Prefix, week)
    
    _, err = a.s3Client.PutObject(ctx, &s3.PutObjectInput{
        Bucket:       aws.String(S3Bucket),
        Key:          aws.String(key),
        Body:         file,
        ContentLength: aws.Int64(stat.Size()),
        StorageClass: types.StorageClassStandardIa,
    })
    if err != nil {
        return fmt.Errorf("s3 upload: %w", err)
    }
    
    // 4. Verify
    head, err := a.s3Client.HeadObject(ctx, &s3.HeadObjectInput{
        Bucket: aws.String(S3Bucket),
        Key:    aws.String(key),
    })
    if err != nil {
        return fmt.Errorf("s3 verify: %w", err)
    }
    
    if *head.ContentLength != stat.Size() {
        return fmt.Errorf("upload size mismatch")
    }
    
    // 5. Update metadata
    if err := a.recordArchive(ctx, week, key, stat.Size()); err != nil {
        return fmt.Errorf("record metadata: %w", err)
    }
    
    // 6. Attach as read-only (via partition manager)
    // This would be done through the app's partition manager
    
    // 7. Cleanup old files
    if err := a.cleanupOldFiles(ctx); err != nil {
        log.Error().Err(err).Msg("cleanup failed")
    }
    
    log.Info().
        Str("week", week).
        Int64("size", stat.Size()).
        Msg("archival complete")
    
    return nil
}

func (a *Archiver) recordArchive(ctx context.Context, week, s3Path string, size int64) error {
    // Insert into metadata DB
    query := `
        INSERT INTO archives (week, s3_path, size_bytes, archived_at, status)
        VALUES (?, ?, ?, ?, ?)
    `
    _, err := a.db.ExecContext(ctx, query, week, s3Path, size, time.Now(), "complete")
    return err
}

func (a *Archiver) cleanupOldFiles(ctx context.Context) error {
    cutoff := time.Now().Add(-RetainWeeks * 7 * 24 * time.Hour)
    
    entries, err := os.ReadDir(ArchiveDir)
    if err != nil {
        return err
    }
    
    for _, entry := range entries {
        info, _ := entry.Info()
        if info.ModTime().Before(cutoff) {
            path := filepath.Join(ArchiveDir, entry.Name())
            if err := os.Remove(path); err != nil {
                log.Error().Err(err).Str("file", path).Msg("remove failed")
            } else {
                log.Info().Str("file", entry.Name()).Msg("purged old archive")
            }
        }
    }
    
    return nil
}
```

---

### Task 2.2: Cron Scheduling
**Owner:** Backend Engineer  
**Estimated:** 1 hour  
**Source ADR:** ADR-006  
**Source C4:** Weekly Archiver

**Description:**
Schedule weekly archival via cron.

**DoD:**
- [ ] Cron job at Sunday 00:00 UTC
- [ ] Script wrapper for archiver binary
- [ ] Dead man's switch metric

**Cron:**
```bash
# /etc/cron.d/tap-trading-archiver
0 0 * * 0 root /opt/app/scripts/archive-weekly.sh >> /var/log/archiver.log 2>&1
```

**Script:**
```bash
#!/bin/bash
# /opt/app/scripts/archive-weekly.sh

WEEK=$(date -d 'last week' +%Y_w%U)
/opt/app/bin/archiver -week="$WEEK"
```

---

## Task 3: Data Integrity

### Task 3.1: Implement Daily Integrity Check
**Owner:** Backend Engineer  
**Estimated:** 3 hours  
**Source ADR:** ADR-010 (Financial auditability)  
**Source C4:** SQLite

**Description:**
Daily check that users.balance matches SUM(ledger.amount).

**DoD:**
- [ ] Query comparing balance to ledger sum
- [ ] Report any drift
- [ ] Auto-correct option (emergency only)
- [ ] Metric export for alerting

**Implementation:**
```go
// cmd/archiver/integrity.go
package main

import (
    "context"
    "database/sql"
)

func (a *Archiver) RunIntegrityCheck(ctx context.Context) (*IntegrityReport, error) {
    query := `
        WITH ledger_balances AS (
            SELECT 
                user_id,
                COALESCE(SUM(amount), 0) as ledger_sum
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
           OR (lb.ledger_sum IS NULL AND u.balance != 0)
    `
    
    rows, err := a.db.QueryContext(ctx, query)
    if err != nil {
        return nil, err
    }
    defer rows.Close()
    
    var drifts []BalanceDrift
    for rows.Next() {
        var d BalanceDrift
        if err := rows.Scan(&d.UserID, &d.UserBalance, &d.LedgerSum, &d.Drift); err != nil {
            continue
        }
        drifts = append(drifts, d)
    }
    
    report := &IntegrityReport{
        CheckedAt: time.Now(),
        Drifts:    drifts,
        Valid:     len(drifts) == 0,
    }
    
    // Export metric
    metrics.RecordDriftCount(len(drifts))
    
    return report, nil
}

type BalanceDrift struct {
    UserID      string
    UserBalance int64
    LedgerSum   int64
    Drift       int64
}

type IntegrityReport struct {
    CheckedAt time.Time
    Drifts    []BalanceDrift
    Valid     bool
}
```

**Cron:**
```bash
# Daily at 02:00 UTC
0 2 * * * root /opt/app/bin/archiver -integrity-check >> /var/log/integrity.log 2>&1
```

---

## Task 4: Dead Man's Switch

### Task 4.1: Archive Health Metric
**Owner:** Backend Engineer  
**Estimated:** 1 hour  
**Source ADR:** ADR-006 (Operational monitoring)  
**Source C4:** Archiver

**Description:**
Export metric for last successful archive timestamp.

**DoD:**
- [ ] Gauge: `last_successful_archive_timestamp`
- [ ] Alert if >8 days old
- [ ] Prometheus metric export

**Implementation:**
```go
var (
    lastArchiveTimestamp = prometheus.NewGauge(prometheus.GaugeOpts{
        Name: "last_successful_archive_timestamp",
        Help: "Unix timestamp of last successful archive",
    })
)

func init() {
    prometheus.MustRegister(lastArchiveTimestamp)
}

func (a *Archiver) updateArchiveMetric() {
    lastArchiveTimestamp.SetToCurrentTime()
}
```

**Alert Rule:**
```yaml
- alert: ArchiveStale
  expr: time() - last_successful_archive_timestamp > 8 * 24 * 3600
  for: 1h
  labels:
    severity: critical
  annotations:
    summary: "Weekly archive has not run in 8+ days"
```

---

## Module Deliverables

- [ ] Bet resolution worker
- [ ] Win/loss determination with price history
- [ ] Payout processing with ledger entries
- [ ] Weekly archival to S3 (VACUUM, zstd, upload)
- [ ] Read-only attachment for historical data
- [ ] Daily integrity check (balance vs ledger)
- [ ] Dead man's switch metric for archival
- [ ] Cron scheduling for all jobs

---

## Exit Criteria

This module is complete when:
1. Bets are automatically resolved based on price history
2. Winners receive payouts with ledger entries
3. Weekly archival runs successfully end-to-end
4. Integrity check passes daily (zero drift)
5. Dead man's switch metric is exported
6. Historical queries work via read replica
7. Disk usage stays within 256GB (with archival)
