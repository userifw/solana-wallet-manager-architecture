# Solana Wallet Manager Architecture

Production-oriented architecture for secure Solana wallet management, SPL transfers, financial integrity, reconciliation, failure recovery, and distributed signing.

This repository is a **sanitized architecture case study**. Production source code, private operational data, wallet secrets, infrastructure credentials, and internal logs are intentionally not published.

## What this project solves

Blockchain transaction submission is easy. Reliable financial processing is not.

The architecture focuses on failure modes that appear in real payment and wallet systems:

- append-only financial ledger
- atomic balance reservations
- protection against double-spend and double-credit
- idempotency across HTTP, queue, outbox, and blockchain boundaries
- explicit operation and transaction-attempt lifecycles
- ambiguous RPC outcomes and lost responses
- signature-first reconciliation
- blockhash expiration handling
- crash/restart recovery
- external blockchain movement detection
- operational health, alerts, and manual review
- encrypted wallet key storage and a single controlled signing boundary

## Core transaction pipeline

```text
FinancialOperation
    ↓
Transactional Outbox
    ↓
Atomic Reservation
    ↓
TransactionAttempt
    ↓
Local Signing
    ↓
One-shot RPC Submission
    ↓
Reconciliation
    ↓
BlockchainMovement
    ↓
Settlement / Incoming Credit
    ↓
Append-only Ledger
```

A successful RPC response is **not** treated as settlement. Blockchain state is verified independently before the financial ledger is finalized.

## Key invariants

1. No double spend.
2. No double credit.
3. No duplicate settlement or release.
4. Available balance cannot become negative.
5. Financial history is append-only.
6. Economic intent is immutable.
7. Only one active blockchain attempt may exist for an operation.
8. Timeout after submission is not equivalent to failure.
9. Unknown submission state retains the reservation.
10. Confirmation comes from chain reconciliation, not from `sendTransaction`.
11. Ambiguous failures never trigger blind resend.
12. There is exactly one supported production outgoing transaction boundary.

## Failure model

The system explicitly distinguishes:

```text
PRE_SUBMISSION_FAILURE
DEFINITE_REJECTION
SUBMITTED
UNKNOWN_SUBMISSION_RESULT
```

If the RPC accepts a transaction but the response is lost, the operation becomes `unknown`. Its reservation remains active. The locally known Solana signature is then used by reconciliation to determine what actually happened on-chain.

## Concurrency model

Reservations are serialized by a database-backed lock scope for:

```text
wallet + asset
```

For SPL tokens, the mint is part of the asset identity.

A real MySQL contention scenario such as:

```text
confirmed = 100
request A = 70
request B = 70
```

must result in exactly one reservation. The second request fails for insufficient available balance. Correctness does not depend on frontend controls, Redis locks, or queue uniqueness.

## Reconciliation

Reconciliation is a separate financial subsystem, not a synonym for wallet synchronization.

It compares:

- immutable business intent
- transaction attempt
- known Solana signature
- parsed on-chain movement
- internal ledger state

Possible outcomes include confirmed-on-chain, pending, missing, expired-without-transaction, amount/destination/asset mismatch, duplicate detection, and manual review.

## Recovery

Restart recovery is state-specific. It never means “retry the transfer.”

Examples:

- reserved + lost dispatch → redispatch the idempotent processing workflow
- signed but provably never submitted → safe technical retry may be allowed
- submission boundary crossed → never resend blindly
- submitted/unknown → reconciliation only
- contradictory evidence → manual review

## Security boundary

Wallet private keys are encrypted at rest and decrypted only in backend memory immediately before signing.

The supported outgoing path is:

```text
FinancialOperation
→ outbox
→ reservation
→ TransactionAttempt
→ signing
→ submission
→ reconciliation
→ settlement
```

Direct SOL/SPL transfer helpers are deliberately excluded from the application API.

## Guardian Nodes

A planned extension introduces distributed **Guardian Nodes**.

A central Wallet Manager acts as the Mentor. A Guardian is installed on a predefined VPS, binds to exactly one Mentor, generates and stores its Solana private keys locally, and accepts only authenticated commands from that Mentor.

The Mentor decides **what** should happen.  
The Guardian owns keys and performs **signing/submission**.  
Solana determines **what actually happened**.  
The Mentor independently reconciles the result.

See [Guardian Nodes](docs/GUARDIAN_NODES.md).

## Documentation

- [Architecture](docs/ARCHITECTURE.md)
- [Financial integrity](docs/FINANCIAL_INTEGRITY.md)
- [Security model](docs/SECURITY.md)
- [Guardian Nodes](docs/GUARDIAN_NODES.md)

## Verification status

The private implementation has been exercised with automated, concurrency, recovery, reconciliation, and controlled Solana Devnet tests, including real SOL and SPL transfers. This public repository intentionally documents the architecture rather than publishing production code or operational data.

## Technology context

Reference implementation:

- Laravel / PHP
- MySQL
- Redis queues
- Solana JSON-RPC
- Node.js bridge for Solana transaction preparation/signing
- Docker
- Caddy / Nginx

The architectural patterns are not Laravel-specific and can be applied to other wallet/payment stacks.

## Public vs private boundary

Published here:

- architecture
- state-machine concepts
- failure semantics
- sanitized examples
- security boundaries

Not published:

- production source code
- wallet private keys or encrypted key material
- RPC/API credentials
- production database contents
- internal logs
- operational infrastructure configuration
- sensitive incident data

## Author

Andrew — Senior Full-Stack / Backend Engineer  
Portfolio: https://ifreework.com/en/
