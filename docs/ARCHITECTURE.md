# Architecture

## Separation of concerns

The system intentionally separates six concepts that are often collapsed into a single transaction table.

### FinancialOperation

Immutable business intent: move a specific raw amount of a specific asset from a managed wallet to a destination.

### Reservation

Temporary claim on available funds before signing. It prevents concurrent operations from spending the same balance.

### TransactionAttempt

One concrete Solana transaction preparation/submission attempt. It stores public technical identity such as blockhash, last valid block height, local signature, and a hash of the signed payload. Private key material and serialized signed transactions are not persisted.

### BlockchainMovement

Observed on-chain economic movement identified by network, signature, and instruction identity.

### LedgerEntry

Append-only internal accounting effect.

### Reconciliation

Evidence-driven comparison between the intended operation and the blockchain result.

## Outgoing flow

```text
HTTP intent
  ↓
FinancialOperation + Outbox (same DB transaction)
  ↓
Atomic reservation
  ↓
TransactionAttempt
  ↓
Persist signature/blockhash identity
  ↓
COMMIT
  ↓
RPC submission
  ↓
submitted | unknown | definite rejection
  ↓
Reconciliation
  ↓
Settlement / release / manual review
```

No database lock is held across network I/O.

## Idempotency boundaries

Idempotency exists independently at multiple boundaries:

- HTTP request
- business operation
- transactional outbox
- queue delivery
- reservation
- transaction attempt
- settlement
- incoming blockchain credit

Queue delivery is treated as at-least-once. Consumers are idempotent rather than assuming exactly-once infrastructure.

## Concurrency

A financial balance scope provides a stable MySQL row for `SELECT ... FOR UPDATE`.

Identity:

```text
SOL: wallet + SOL
SPL: wallet + mint
```

The balance check and reservation insertion happen inside the same database transaction. Current/locking reads are used to avoid stale REPEATABLE READ snapshots after waiting for a competing transaction.

## Submission boundary

The critical boundary is whether bytes may have reached the RPC provider.

Before network submission, the system durably stores:

- operation state
- attempt state
- locally derived Solana signature
- blockhash
- last valid block height
- signed payload hash

Only then is the RPC request made.

A transport timeout after this boundary is ambiguous and becomes `unknown`, not `failed`.

## Reconciliation

Known signatures are checked first. Reconciliation validates not merely that a transaction exists, but that the economic effect matches:

- source
- destination
- asset
- mint
- raw amount

Mismatch does not trigger automatic settlement or release.

## Recovery

Recovery is operational, not financial. It detects stuck states and chooses a state-specific safe action. It never implements a generic “retry transfer” button.

## Observability

Metrics, health, alerts, and scheduler heartbeats are projections. They never become a source of truth for financial state.

Critical signals include:

- unknown/stuck operations
- reconciliation mismatch
- unexpected Treasury outgoing
- negative available-balance invariant
- RPC outage
- queue/outbox backlog
- stale scheduler heartbeat
