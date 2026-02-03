# ADR 007: Replay Protection for Web3 Auth

## Status
Accepted (Hardening Response)

## Context
Hardening Audit #001 identified vulnerability: Without replay protection, attackers can capture valid bet payloads and replay them to drain user funds or grief the system.

## Decision
Implement **timestamp + nonce** validation with Redis-backed replay cache.

## Solution Design

### Signature Payload Schema

```typescript
// Client constructs message for signing
interface BetSignaturePayload {
    action: "place_bet";
    cell_id: string;           // Grid cell identifier
    amount: string;            // Bet amount (wei-like string)
    nonce: string;             // UUID v4, unique per request
    timestamp: number;         // Unix timestamp (seconds)
    chain_id: number;          // Prevent cross-chain replay
    contract_version: string;  // "1" for protocol versioning
}

// EIP-191 compatible message format
const message = `Tap Trading Bet
Action: place_bet
Cell: ${cell_id}
Amount: ${amount}
Nonce: ${nonce}
Timestamp: ${timestamp}
Chain: ${chain_id}
Version: ${contract_version}`;

// Sign with personal_sign (EIP-191)
const signature = await wallet.signMessage(message);
```

### Server Validation Flow

```go
type SignatureValidator struct {
    redis *redis.Client
    clock func() time.Time
}

func (v *SignatureValidator) ValidateBet(ctx context.Context, req BetRequest) error {
    // 1. Timestamp freshness (prevent old signature reuse)
    now := v.clock().Unix()
    if abs(now - req.Timestamp) > 30 {
        return ErrSignatureExpired
    }
    
    // 2. Nonce uniqueness check (Redis SET NX with TTL)
    nonceKey := fmt.Sprintf("nonce:%s", req.Nonce)
    ok, err := v.redis.SetNX(ctx, nonceKey, req.UserID, 24*time.Hour).Result()
    if err != nil || !ok {
        return ErrReplayDetected  // Nonce already used
    }
    
    // 3. Recover signer address from signature
    recoveredAddr, err := recoverAddress(req.Message, req.Signature)
    if err != nil {
        return ErrInvalidSignature
    }
    
    // 4. Verify signer matches claimed user
    if !strings.EqualFold(recoveredAddr, req.UserID) {
        return ErrUnauthorized
    }
    
    // 5. Verify payload integrity (hash matches)
    expectedMessage := constructMessage(req)
    if req.Message != expectedMessage {
        return ErrTamperedPayload
    }
    
    return nil
}
```

### Redis Key Strategy

| Key Pattern | Type | TTL | Purpose |
|-------------|------|-----|---------|
| `nonce:{uuid}` | String | 24h | Replay protection (SET NX) |
| `nonce_count:{date}` | Counter | 7d | Rate limiting abuse detection |

### Client Implementation

```typescript
class BetClient {
    async placeBet(cellId: string, amount: string): Promise<void> {
        const payload = {
            action: "place_bet",
            cell_id: cellId,
            amount: amount,
            nonce: crypto.randomUUID(),
            timestamp: Math.floor(Date.now() / 1000),
            chain_id: await wallet.getChainId(),
            contract_version: "1"
        };
        
        const message = this.formatMessage(payload);
        const signature = await wallet.signMessage(message);
        
        // Optimistic UI update
        this.grid.markPending(cellId);
        
        try {
            await api.post('/bets', {
                ...payload,
                message,
                signature
            });
        } catch (e) {
            this.grid.revertPending(cellId);
            throw e;
        }
    }
}
```

## Attack Scenarios Mitigated

| Attack | Mitigation |
|--------|------------|
| Replay old bet | Timestamp expires after 30s |
| Replay same nonce | Redis SET NX fails on duplicate |
| Cross-chain replay | Chain ID in signed payload |
| Signature malleability | EIP-191 prefix standardization |
| Clock skew | 30s window accommodates minor drift |

## Trade-off Analysis

| Alternative | Why Rejected |
|-------------|--------------|
| Timestamp only | Clock skew issues; no nonce = replay within window |
| Nonce only | Requires persistent nonce storage (infinite growth) |
| Session tokens | Violates "Web3 native" design; adds complexity |
| On-chain verification | Gas costs, latency, violates off-chain custody |

## NFR Compliance

| NFR | Compliance |
|-----|------------|
| Replay attack prevention | ✅ Timestamp + Nonce + Redis |
| ≤100ms latency | ✅ Redis check is sub-millisecond |
| Stateless validation | ✅ No session storage needed |

## Decision Date
2026-02-03

## Author
System Architect (Hardening Response)
