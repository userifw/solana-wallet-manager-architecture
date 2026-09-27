# Security Model

## Key custody

Managed Solana private keys are generated locally, encrypted at rest, and decrypted only in backend memory immediately before signing.

Key material is excluded from:

- HTTP resources
- queue payloads
- outbox payloads
- audit metadata
- reconciliation records
- alerts
- application logs

## Single write path

There is one supported production outgoing transaction boundary:

```text
FinancialOperation
→ Outbox
→ Reservation
→ TransactionAttempt
→ Signing
→ Submission
→ Reconciliation
→ Settlement
```

Direct transfer helpers are intentionally excluded. This reduces the risk that a future route, command, or background job bypasses reservations, ledger accounting, reconciliation, or emergency-stop controls.

## Deny by default

Financial operations, blockchain submission, and Mainnet writes are independent safety gates and default to disabled.

Read-only synchronization and reconciliation can remain operational while outgoing writes are stopped.

## Treasury

Treasury operations receive stricter controls. Unexpected external outgoing activity is treated as a critical incident and is not silently reconciled into the ledger.

## Authentication

The private administrative implementation uses passwordless email OTP followed by RFC 6238 TOTP. OTPs are short-lived, single-use, rate-limited, and stored only as hashes. TOTP secrets are encrypted at rest.

## Backup

Wallet custody requires both database recovery and protection of the encryption key. Production Treasury backups are encrypted with a separate passphrase and verified off-host before Mainnet activation.

## Network isolation

Database and cache services are not exposed publicly. Administrative services are published only through the HTTPS reverse proxy. Distributed Guardian Nodes should use a private network plus cryptographic command authentication rather than trusting source IP alone.

## Public repository boundary

This repository intentionally contains no production secrets, wallet material, credential-bearing RPC URLs, production database data, or internal operational logs.
