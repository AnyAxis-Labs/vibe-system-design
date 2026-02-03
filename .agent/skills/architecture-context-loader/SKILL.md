---
name: architecture-context-loader
description: Loads and interprets system architecture context from PRD, current_state.md, ADRs, and C4 diagrams. Ensures correct handling of ADR evolution, deprecation, and consistency.
---

# Architecture Context Loader

This skill ensures accurate loading and interpretation of system architecture documentation, preventing hallucination and maintaining consistency with evolved design decisions.

## When to Use

- Starting a new session on an existing architecture project
- Before making any architectural changes or additions
- When reviewing or auditing existing architecture
- When creating implementation plans from ADRs
- When resolving hardening challenges or critiques

## Context Loading Procedure

### Step 1: Load Core Documents (REQUIRED)

Always load in this exact order:

```
1. @prd.md                    → Product requirements and constraints
2. @current_state.md          → Current architecture state, version, active ADRs
3. @docs/adr/ (all ADRs)      → Architectural decisions (check for deprecation chains)
4. @docs/diagrams/            → C4 diagrams for visual reference
```

**CRITICAL:** Never assume you know the current state without reading `current_state.md`.

### Step 2: Identify Architecture Version

From `current_state.md`, extract:
- **Version:** (e.g., "Hardened v1.3")
- **Active ADRs:** Listed in "Key Architectural Decisions" table
- **Superseded ADRs:** Check ADR files for "Supersedes" or "Superseded By" headers

### Step 3: ADR Interpretation Rules

#### Rule 1: Check for Deprecation Chains

Read each ADR's **Status** and **Relationships** sections:

```markdown
## Status
Accepted / Deprecated / Superseded

## Relationships
- Supersedes: ADR-XXX (replaces this older decision)
- Superseded By: ADR-YYY (this decision is replaced by newer one)
```

**Status Definitions:**

| Status | Meaning | Action |
|--------|---------|--------|
| **Accepted** | Currently active decision | ✅ Include in design |
| **Superseded** | Replaced by newer ADR | ❌ DO NOT USE; follow "Superseded By" pointer |
| **Deprecated** | No longer recommended, but no clear replacement | ⚠️ Avoid using; check for better alternatives |

**Important:** `Deprecated` ≠ `Superseded`
- **Superseded**: Clear replacement exists (follow the link)
- **Deprecated**: Discouraged but no official replacement (use judgment)

**Action:** 
- If ADR is "Superseded" or has "Superseded By": DO NOT include; use the replacement ADR
- If ADR is "Deprecated": Check current_state.md for recommended alternatives
- If ADR is "Accepted": Include in active design (unless extended by another ADR)

#### Rule 2: Distinguish Extension from Replacement

| Relationship Type | Meaning | Action |
|-------------------|---------|--------|
| **Extends** | Adds to existing ADR without changing core decision | Include BOTH ADRs |
| **Replaces** | New ADR changes core decision of old ADR | Include NEW ADR only |
| **Refines** | Clarifies or narrows scope of existing ADR | Include BOTH (newer is authoritative) |

**Example from this project:**
- ADR-009 (Conditional Deduction) + ADR-010 (Hybrid Ledger) = **Extends**
  - ADR-009's UPDATE logic is still active
  - ADR-010 adds INSERT ledger in same transaction
  - **Both are active**, implement together

#### Rule 3: Configuration Changes ≠ ADR Changes

Some "changes" are implementation details, not architectural decisions:

| Type | Example | ADR Status |
|------|---------|------------|
| Config Tweak | Nonce TTL: 24h → 90s | ADR unchanged, config updated |
| Addition | Add hysteresis to CB | ADR updated or new ADR |
| Replacement | Switch from MySQL to PostgreSQL | New ADR supersedes old |

**Check:** Look at the ADR's "Last Updated" vs "Decision Date". If config changed but ADR date is old, it's a config change.

### Step 4: Verify Consistency

Cross-check between documents:

| Check | Source A | Source B | Must Match? |
|-------|----------|----------|-------------|
| Active ADRs | current_state.md | ADR files | YES |
| Tech Stack | current_state.md | C4 diagrams | YES |
| Data Retention | current_state.md | ADR-006 | YES |
| Performance | prd.md (NFRs) | ADR implementations | YES |

**Red Flag:** If current_state.md says "v1.3" but ADR files only go up to 008, you're missing context.

### Step 5: Extract Current Design Snapshot

Create a mental (or actual) summary:

```
Architecture: [Name] [Version]
Status: [Active/Deprecated/Experimental]

Active ADRs:
- ADR-001: [Brief decision] ✓
- ADR-002: [Brief decision] ✓
...
- ADR-NNN: [Brief decision] ✓

Superseded/Deprecated ADRs (DO NOT USE):
- ADR-XXX: Superseded by ADR-YYY

Key Constraints:
- [From PRD and current_state.md]

Critical Implementation Notes:
- [Any "gotchas" from hardening rounds]
```

## Common Pitfalls to Avoid

### Pitfall 1: Using Superseded or Deprecated ADRs

**Wrong (Superseded):** Implementing ADR-003's decision when ADR-007 supersedes it.

**Right:** Check "Superseded By" field. Implement ADR-007 instead.

**Wrong (Deprecated):** Using ADR-005's approach because it's still marked "Accepted" in your memory.

**Right:** Check Status - if "Deprecated", find the newer approach in recent ADRs or current_state.md.

### Pitfall 2: Double-Implementing Extended ADRs

**Wrong:** Implementing ADR-009's balance deduction, then ADR-010's ledger as separate features that conflict.

**Right:** Recognize ADR-010 EXTENDS ADR-009. Implement both in SAME transaction:
```sql
BEGIN;
  -- From ADR-009
  UPDATE users SET balance = balance - ? WHERE ...;
  
  -- From ADR-010 (extends)
  INSERT INTO transaction_ledger ...;
COMMIT;
```

### Pitfall 3: Missing Hardening Rounds

**Wrong:** Implementing v1.0 design without reading hardening critiques.

**Right:** Check `docs/hardening/` for resolution documents. Ensure v1.3 (or latest) hardening is applied.

### Pitfall 4: Config vs Architecture Confusion

**Wrong:** Creating new ADR for "Change nonce TTL from 24h to 90s".

**Right:** This is a config change to existing ADR-007. Update implementation, not architecture.

### Pitfall 5: Stale Diagrams

**Wrong:** Using C4 Context diagram that shows v1.0 components when design is v1.3.

**Right:** Check diagram timestamps or version headers. Regenerate if needed.

## Decision Tree: How to Handle ADR Changes

```
Reading an ADR file
        ↓
Check Status header
        ↓
    ┌───┴───┬─────────────┐
    ↓       ↓             ↓
Accepted  Deprecated    Superseded
    ↓       ↓             ↓
    ↓    Use with      Check "Superseded By"
    ↓    caution;      ↓
    ↓    look for      Note replacement ADR
    ↓    alternatives  ↓
    ↓                    Exclude this ADR
    ↓                    Use replacement instead
    ↓
Check "Supersedes"
    ↓
┌───┴───┐
↓       ↓
None   ADR-YYY
↓       ↓
Check "Extends"    Update ADR-YYY:
    ↓              Status = Superseded
┌───┴───┐          Superseded By = this ADR
↓       ↓
None   ADR-ZZZ
↓       ↓
This ADR Implement BOTH
standalone together
```

## Implementation Plan Guidelines

When creating implementation plans from architecture:

1. **Reference Source:** Every task must cite source ADR and C4 Container
2. **Check Active Status:** Verify ADR is not superseded before including
3. **Version Alignment:** Ensure tasks match current_state.md version
4. **Dependency Chain:** If ADR-B extends ADR-A, tasks for both must be coordinated

### Implementation Task Template

```markdown
### Task Name
**Owner:** [BE/FE/Both]  
**Estimated:** [Hours]  
**Source ADR:** [ADR-XXX] (Status: Accepted)  
**Source C4:** [Container Name]  
**Extends:** [ADR-YYY if applicable]  

**Description:**
[What to implement]

**DoD:**
- [ ] Criterion 1
- [ ] Criterion 2

**⚠️ Notes:**
- [Any conflicts, deprecated alternatives, or coordination needs]
```

## Example: Loading BTC Tap Trading v1.3

### Correct Loading

1. **Read PRD:** 3K RPS, 100ms latency, Web3 auth, 2-week timeline
2. **Read current_state.md:** 
   - Version: Hardened v1.3
   - Active ADRs: 001-010 (all accepted)
   - No superseded ADRs
   - Constraints: Single VM, 256GB SSD
3. **Read ADRs 001-010:**
   - Check each Status: All "Accepted"
   - Check Relationships:
     - ADR-010 "Extends" ADR-009 (implement together)
     - No "Superseded By" found
4. **Read C4 diagrams:** Verify they match v1.3 (ledger component present)
5. **Check hardening:** All 3 rounds resolved

### Correct Interpretation

```
Active Design: Hardened v1.3
ADRs to Implement: 001, 002, 003, 004, 005, 006, 007, 008, 009, 010

Coordination Required:
- ADR-009 + ADR-010: Same transaction (balance update + ledger)

Config Values (not ADR changes):
- Nonce TTL: 90s (from 24h in ADR-007)
- CB Recovery: 30s hysteresis (added to ADR-008)

No Deprecated Decisions: ✓
```

### Incorrect Interpretation (AVOID)

```
❌ Implementing ADR-009 WITHOUT ADR-010
   → Missing audit trail, financial integrity gap

❌ Implementing ledger as separate feature from balance update
   → Race condition, data inconsistency

❌ Using original 24h nonce TTL from ADR-007 text
   → Redis OOM risk (hardening fix: 90s)

❌ Implementing v1.0 design without batching
   → SQLite contention, 3K RPS failure
```

## Self-Check Before Acting

Before making any architectural changes or implementation plans:

- [ ] Loaded PRD, current_state.md, all ADRs, diagrams
- [ ] Identified current version from current_state.md
- [ ] Checked all ADRs for "Superseded By" relationships
- [ ] Identified which ADRs extend others (must implement together)
- [ ] Verified C4 diagrams match current version
- [ ] Cross-checked constraints between PRD and current_state.md
- [ ] Noted any config changes that override ADR defaults

## Quick Reference: File Reading Order

```bash
# 1. Requirements
read prd.md

# 2. Current State (ALWAYS)
read current_state.md

# 3. All ADRs (check for deprecation)
for adr in docs/adr/*.md; do
    read $adr  # Check Status and Relationships headers
done

# 4. Diagrams
read docs/diagrams/c4-context.md
read docs/diagrams/c4-container.md

# 5. Hardening (if exists)
for h in docs/hardening/*_resolution.md; do
    read $h
done
```

## Maintenance Note

When creating new ADRs:
- Always include "Status" and "Relationships" sections
- If extending: `Extends: ADR-XXX`
- If superseding: `Supersedes: ADR-XXX` in new ADR, update old ADR to `Superseded By: ADR-YYY`
- Update current_state.md "Active ADRs" table immediately

When updating current_state.md:
- Bump version number for significant changes
- Update "Hardening Summary" with new rounds
- Ensure ADR table matches actual ADR files
