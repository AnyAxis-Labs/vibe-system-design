# Module 2: Authentication & Security (Vertical)

**Goal:** Implement Web3 signature-based authentication with replay protection.

**Duration:** Days 4-5  
**Dependencies:** Module 1 (Core Infrastructure)  
**Priority:** HIGH (required for bet placement)

---

## Module Scope

This module implements:
- EIP-191 signature validation
- Timestamp + nonce replay protection
- Session management
- Rate limiting

**Source ADRs:** ADR-007 (Replay Protection)  
**Source C4:** Signature Validator, Session Manager containers

---

## Task 1: Signature Validation (ADR-007)

### Task 1.1: Implement EIP-191 Signature Recovery
**Owner:** Backend Engineer  
**Estimated:** 4 hours  
**Source ADR:** ADR-007 (Replay Protection for Web3 Auth)  
**Source C4:** Signature Validator

**Description:**
Implement Ethereum signature verification using EIP-191 personal_sign standard.

**DoD:**
- [ ] Recover address from signature
- [ ] Verify signature format (65 bytes: r, s, v)
- [ ] EIP-191 prefix validation
- [ ] Address normalization (checksum vs lowercase)
- [ ] Unit tests with known good/bad signatures

**Implementation:**
```go
// pkg/auth/signature.go
package auth

import (
    "crypto/ecdsa"
    "fmt"
    "strings"
    
    "github.com/ethereum/go-ethereum/common"
    "github.com/ethereum/go-ethereum/crypto"
)

type SignatureValidator struct {
    // dependencies
}

type BetSignaturePayload struct {
    Action          string `json:"action"`
    CellID          string `json:"cell_id"`
    Amount          string `json:"amount"`
    Nonce           string `json:"nonce"`
    Timestamp       int64  `json:"timestamp"`
    ChainID         int    `json:"chain_id"`
    ContractVersion string `json:"contract_version"`
}

func (v *SignatureValidator) FormatMessage(payload BetSignaturePayload) string {
    return fmt.Sprintf(`Tap Trading Bet
Action: %s
Cell: %s
Amount: %s
Nonce: %s
Timestamp: %d
Chain: %d
Version: %s`,
        payload.Action,
        payload.CellID,
        payload.Amount,
        payload.Nonce,
        payload.Timestamp,
        payload.ChainID,
        payload.ContractVersion,
    )
}

func (v *SignatureValidator) RecoverAddress(message string, signature []byte) (string, error) {
    // EIP-191 message prefix
    prefixed := fmt.Sprintf("\x19Ethereum Signed Message:\n%d%s", len(message), message)
    msgHash := crypto.Keccak256Hash([]byte(prefixed))
    
    // Handle signature malleability (v can be 27/28 or 0/1)
    if len(signature) != 65 {
        return "", fmt.Errorf("invalid signature length: %d", len(signature))
    }
    
    sig := make([]byte, 65)
    copy(sig, signature)
    
    // Adjust v value
    if sig[64] >= 27 {
        sig[64] -= 27
    }
    
    pubKey, err := crypto.SigToPub(msgHash.Bytes(), sig)
    if err != nil {
        return "", fmt.Errorf("recover public key: %w", err)
    }
    
    address := crypto.PubkeyToAddress(*pubKey)
    return address.Hex(), nil
}

func (v *SignatureValidator) ValidateSignature(payload BetSignaturePayload, signature string, expectedAddress string) error {
    // 1. Reconstruct message
    message := v.FormatMessage(payload)
    
    // 2. Decode signature (hex)
    sigBytes, err := hexutil.Decode(signature)
    if err != nil {
        return fmt.Errorf("decode signature: %w", err)
    }
    
    // 3. Recover address
    recovered, err := v.RecoverAddress(message, sigBytes)
    if err != nil {
        return fmt.Errorf("recover address: %w", err)
    }
    
    // 4. Compare addresses (case-insensitive)
    if !strings.EqualFold(recovered, expectedAddress) {
        return fmt.Errorf("signature mismatch: recovered %s, expected %s", recovered, expectedAddress)
    }
    
    return nil
}
```

**Tests:**
- [ ] Valid signature verification
- [ ] Invalid signature rejection
- [ ] Wrong address rejection
- [ ] Signature malleability handling

---

## Task 2: Replay Protection (ADR-007)

### Task 2.1: Implement Timestamp Validation
**Owner:** Backend Engineer  
**Estimated:** 2 hours  
**Source ADR:** ADR-007  
**Source C4:** Signature Validator

**Description:**
Validate request timestamp freshness to prevent replay of old signatures.

**DoD:**
- [ ] 30-second window (clock drift tolerance)
- [ ] Reject requests with old timestamps
- [ ] Reject requests with future timestamps
- [ ] Configurable clock skew tolerance

**Implementation:**
```go
// pkg/auth/replay.go
const (
    TimestampWindow = 30 * time.Second
)

func (v *SignatureValidator) ValidateTimestamp(timestamp int64) error {
    now := time.Now().Unix()
    diff := abs(now - timestamp)
    
    if diff > int64(TimestampWindow.Seconds()) {
        return fmt.Errorf("timestamp expired: diff=%ds", diff)
    }
    
    if timestamp > now+5 { // Small future tolerance
        return fmt.Errorf("timestamp in future")
    }
    
    return nil
}
```

---

### Task 2.2: Implement Nonce Tracking
**Owner:** Backend Engineer  
**Estimated:** 3 hours  
**Source ADR:** ADR-007  
**Source C4:** Signature Validator → Redis

**Description:**
Implement nonce uniqueness check using Redis SET NX.

**DoD:**
- [ ] Redis SET NX for atomic check-and-set
- [ ] 90-second TTL (ADR-007 hardening fix, not 24h)
- [ ] Nonce key format: `nonce:{uuid}`
- [ ] Duplicate nonce rejection

**Implementation:**
```go
// pkg/auth/nonce.go
package auth

import (
    "context"
    "fmt"
    "time"
    
    "github.com/tap-trading/backend/pkg/redis"
)

const (
    NonceTTL = 90 * time.Second // Hardened: was 24h, now 90s
)

type NonceStore struct {
    redis *redis.Client
}

func NewNonceStore(redis *redis.Client) *NonceStore {
    return &NonceStore{redis: redis}
}

func (s *NonceStore) CheckAndStore(ctx context.Context, nonce string, userID string) error {
    key := fmt.Sprintf("nonce:%s", nonce)
    
    // SET key value NX EX ttl
    ok, err := s.redis.SetNX(ctx, key, userID, NonceTTL).Result()
    if err != nil {
        return fmt.Errorf("redis error: %w", err)
    }
    
    if !ok {
        return fmt.Errorf("nonce already used: %s", nonce)
    }
    
    return nil
}
```

**⚠️ Critical:**
- TTL is 90 seconds (30s window + 60s buffer)
- This reduces memory from ~13GB to ~13MB at 3K RPS

---

## Task 3: Session Management

### Task 3.1: Implement Session Store
**Owner:** Backend Engineer  
**Estimated:** 3 hours  
**Source ADR:** ADR-004 (Redis for Sessions)  
**Source C4:** Session Manager

**Description:**
Implement session management for authenticated users.

**DoD:**
- [ ] Session creation on successful auth
- [ ] 24-hour session TTL
- [ ] Session validation middleware
- [ ] Session revocation (logout)

**Implementation:**
```go
// pkg/auth/session.go
package auth

import (
    "context"
    "crypto/rand"
    "encoding/hex"
    "fmt"
    "time"
    
    "github.com/tap-trading/backend/pkg/redis"
)

const (
    SessionTTL = 24 * time.Hour
)

type Session struct {
    ID        string    `json:"id"`
    UserID    string    `json:"user_id"`
    Wallet    string    `json:"wallet"`
    CreatedAt time.Time `json:"created_at"`
}

type SessionStore struct {
    redis *redis.Client
}

func NewSessionStore(redis *redis.Client) *SessionStore {
    return &SessionStore{redis: redis}
}

func (s *SessionStore) Create(ctx context.Context, userID, wallet string) (*Session, error) {
    sessionID := generateSessionID()
    
    session := &Session{
        ID:        sessionID,
        UserID:    userID,
        Wallet:    wallet,
        CreatedAt: time.Now(),
    }
    
    key := fmt.Sprintf("session:%s", sessionID)
    // Store session in Redis
    err := s.redis.Set(ctx, key, session, SessionTTL).Err()
    if err != nil {
        return nil, fmt.Errorf("store session: %w", err)
    }
    
    // Also index by wallet for lookup
    walletKey := fmt.Sprintf("session:wallet:%s", wallet)
    s.redis.Set(ctx, walletKey, sessionID, SessionTTL)
    
    return session, nil
}

func (s *SessionStore) Get(ctx context.Context, sessionID string) (*Session, error) {
    key := fmt.Sprintf("session:%s", sessionID)
    
    var session Session
    err := s.redis.Get(ctx, key).Scan(&session)
    if err != nil {
        return nil, fmt.Errorf("get session: %w", err)
    }
    
    return &session, nil
}

func (s *SessionStore) Delete(ctx context.Context, sessionID string) error {
    key := fmt.Sprintf("session:%s", sessionID)
    
    // Get wallet first for index cleanup
    var session Session
    err := s.redis.Get(ctx, key).Scan(&session)
    if err == nil {
        walletKey := fmt.Sprintf("session:wallet:%s", session.Wallet)
        s.redis.Del(ctx, walletKey)
    }
    
    return s.redis.Del(ctx, key).Err()
}

func generateSessionID() string {
    b := make([]byte, 32)
    rand.Read(b)
    return hex.EncodeToString(b)
}
```

---

### Task 3.2: Auth Middleware
**Owner:** Backend Engineer  
**Estimated:** 2 hours  
**Source ADR:** ADR-007  
**Source C4:** API Server

**Description:**
Implement Echo middleware for session validation.

**DoD:**
- [ ] Extract session from header/cookie
- [ ] Validate session exists and not expired
- [ ] Inject user context into request
- [ ] Return 401 for invalid sessions

**Implementation:**
```go
// pkg/auth/middleware.go
package auth

import (
    "net/http"
    "strings"
    
    "github.com/labstack/echo/v4"
)

func (s *SessionStore) Middleware() echo.MiddlewareFunc {
    return func(next echo.HandlerFunc) echo.HandlerFunc {
        return func(c echo.Context) error {
            // Extract session from Authorization header
            auth := c.Request().Header.Get("Authorization")
            if auth == "" {
                return c.JSON(http.StatusUnauthorized, map[string]string{
                    "error": "missing authorization header",
                })
            }
            
            parts := strings.SplitN(auth, " ", 2)
            if len(parts) != 2 || strings.ToLower(parts[0]) != "bearer" {
                return c.JSON(http.StatusUnauthorized, map[string]string{
                    "error": "invalid authorization format",
                })
            }
            
            sessionID := parts[1]
            session, err := s.Get(c.Request().Context(), sessionID)
            if err != nil {
                return c.JSON(http.StatusUnauthorized, map[string]string{
                    "error": "invalid or expired session",
                })
            }
            
            // Inject user into context
            c.Set("user_id", session.UserID)
            c.Set("wallet", session.Wallet)
            
            return next(c)
        }
    }
}
```

---

## Task 4: Rate Limiting

### Task 4.1: Implement Rate Limiter
**Owner:** Backend Engineer  
**Estimated:** 2 hours  
**Source ADR:** ADR-004 (Redis for rate limiting)  
**Source C4:** API Server

**Description:**
Implement per-wallet rate limiting for bet placement.

**DoD:**
- [ ] Redis-based sliding window or token bucket
- [ ] Configurable rate limit (e.g., 10 bets/minute)
- [ ] Return 429 with Retry-After header when limited

**Implementation:**
```go
// pkg/auth/ratelimit.go
package auth

import (
    "context"
    "fmt"
    "time"
    
    "github.com/tap-trading/backend/pkg/redis"
)

const (
    RateLimitRequests = 10
    RateLimitWindow   = 60 * time.Second
)

type RateLimiter struct {
    redis *redis.Client
}

func NewRateLimiter(redis *redis.Client) *RateLimiter {
    return &RateLimiter{redis: redis}
}

func (r *RateLimiter) Allow(ctx context.Context, wallet string) (bool, error) {
    key := fmt.Sprintf("rate_limit:%s", wallet)
    
    pipe := r.redis.Pipeline()
    incr := pipe.Incr(ctx, key)
    pipe.Expire(ctx, key, RateLimitWindow)
    
    _, err := pipe.Exec(ctx)
    if err != nil {
        return false, err
    }
    
    count := incr.Val()
    return count <= RateLimitRequests, nil
}
```

---

## Task 5: Complete Auth Flow

### Task 5.1: Implement Login Endpoint
**Owner:** Backend Engineer  
**Estimated:** 3 hours  
**Source ADR:** ADR-007  
**Source C4:** API Server

**Description:**
Create login endpoint that validates Web3 signature and creates session.

**DoD:**
- [ ] `POST /api/auth/login` endpoint
- [ ] Validate signature
- [ ] Validate timestamp + nonce
- [ ] Create or get user record
- [ ] Create session
- [ ] Return session token

**Request/Response:**
```json
// POST /api/auth/login
{
  "wallet": "0x...",
  "payload": { /* BetSignaturePayload */ },
  "signature": "0x..."
}

// Response
{
  "session_id": "sess_abc123",
  "user": {
    "id": "...",
    "wallet": "0x...",
    "balance": 10000
  },
  "expires_at": "2026-02-04T12:00:00Z"
}
```

---

### Task 5.2: Auth Integration Tests
**Owner:** Backend Engineer  
**Estimated:** 3 hours  
**Source ADR:** ADR-007  
**Source C4:** All auth components

**Description:**
Write comprehensive tests for auth flow.

**DoD:**
- [ ] Valid login test
- [ ] Invalid signature rejection
- [ ] Expired timestamp rejection
- [ ] Reused nonce rejection
- [ ] Rate limit enforcement
- [ ] Session validation
- [ ] Logout test

---

## Module Deliverables

- [ ] EIP-191 signature validation
- [ ] Timestamp freshness validation (30s window)
- [ ] Nonce tracking with 90s TTL
- [ ] Session management (24h TTL)
- [ ] Rate limiting per wallet
- [ ] Login endpoint
- [ ] Auth middleware
- [ ] Integration tests

---

## Exit Criteria

This module is complete when:
1. Valid Web3 signatures are accepted
2. Replayed signatures (old timestamp or used nonce) are rejected
3. Sessions are created and can be validated
4. Rate limits are enforced
5. All auth integration tests pass
6. Bet placement can use auth middleware

---

## Security Checklist

- [ ] Signature uses EIP-191 standard
- [ ] Nonce TTL is 90s (not 24h)
- [ ] Timestamp window is 30s
- [ ] Rate limits are enforced
- [ ] Sessions use cryptographically random IDs
- [ ] No sensitive data in logs
