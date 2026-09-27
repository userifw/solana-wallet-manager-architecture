# Financial Integrity

## Ledger semantics

The ledger is append-only. Financially meaningful entries are never edited or deleted.

Representative entry types:

| Entry | Confirmed | Reserved |
|---|---:|---:|
| credit | +amount | — |
| debit | -amount | — |
| reserve | — | +amount |
| release | — | -amount |
| settlement | -amount | -amount |
| reversal / adjustment | compensating | — |

```text
available = confirmed - reserved
```

A successful outgoing operation uses `reserve + settlement`. A failed operation that is proven not to have settled uses `reserve + release`.

## Operation lifecycle

```text
created
  → reserved
  → signing
  → submitted
  → confirmed
```

Ambiguous paths introduce `unknown` and `reconciling`. Terminal outcomes are explicit. Arbitrary state jumps are rejected.

## Submission classification

| Classification | Reservation | Next action |
|---|---|---|
| Pre-submission failure | release | fail operation |
| Definite rejection | release | fail operation |
| Submitted | keep | reconcile |
| Unknown submission result | keep | reconcile |

A timeout after the submission boundary is never sufficient evidence to release funds or create a new transaction.

## Double-spend protection

Correctness relies on database transactions, row locks, unique constraints, and state-machine guards. Redis and queue uniqueness are optimizations, not the financial correctness boundary.

## Double-credit protection

A blockchain movement has a stable instruction-level identity. Repeated scanning, concurrent import workers, and reconciliation may observe the same movement multiple times, but the database permits only one economic credit.

## Reconciliation

Settlement is created only after chain evidence matches the immutable economic intent. Missing or contradictory evidence remains unresolved.

Blockhash expiry alone is not enough to prove failure. The known signature is checked again before a reservation can be released.

## Restart recovery

Crash recovery distinguishes operations that provably never crossed the submission boundary from operations that may have reached Solana.

Only the former may safely create a new technical attempt. The latter are reconciliation-only.

## Manual review

Manual review acknowledges operational investigation. It does not provide controls for force-confirm, force-fail, manual settlement, reservation release, or arbitrary ledger edits.
