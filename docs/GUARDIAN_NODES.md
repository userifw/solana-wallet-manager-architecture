# Guardian Nodes

## Concept

Guardian Nodes extend the Wallet Manager into a distributed signing architecture.

The central **Mentor** owns business logic, ledger, reconciliation, and operational policy. Each **Guardian** runs on a predefined VPS and owns a local set of Solana wallets.

A Guardian binds to one Mentor and does not accept management commands from other sources.

## Responsibilities

### Mentor

- decides what operation should be executed
- tracks Guardian identities and public wallet addresses
- creates financial operations
- performs global accounting
- independently verifies Solana results
- decides confirmed / failed / unknown through reconciliation

### Guardian

- generates Solana keypairs locally
- stores private keys locally
- never exports private keys to the Mentor
- validates authenticated commands
- signs and submits permitted transactions
- returns technical execution data such as a signature
- maintains a local append-only command audit

### Solana

Solana is the source of truth for what actually happened on-chain.

## Command examples

```text
CREATE_WALLETS count=1000
LIST_WALLETS
GET_BALANCES
TRANSFER_SOL from_wallet=1 to_address=... amount_raw=...
TRANSFER_SPL from_wallet=499 to_address=... mint=... amount_raw=...
```

Each command has a unique command ID. Replaying the same command must not create a second transfer.

## Trust model

“Obey the Mentor” does not mean blindly execute malformed or unsafe input.

A Guardian rejects commands when:

- Mentor authentication fails
- command is a replay
- network is wrong
- wallet is unknown or disabled
- destination is invalid
- amount is invalid
- local balance is insufficient
- configured safety limits are exceeded

## Result semantics

A Guardian may report:

```text
accepted
submitted
signature=<public Solana signature>
```

It must not be treated as the authority for “funds delivered.”

The Mentor independently verifies the signature and economic movement through Solana RPC and the existing reconciliation subsystem.

## Integration

The intended pipeline becomes:

```text
FinancialOperation
→ reservation
→ TransactionAttempt
→ Guardian command
→ local Guardian signing
→ Solana submission
→ signature
→ Mentor reconciliation
→ settlement
```

The Guardian replaces the local signer/submission boundary. Ledger, idempotency, unknown-state handling, recovery, and reconciliation remain centralized.

## Network security

Preferred deployment:

- predefined VPS
- private WireGuard network
- cryptographically authenticated commands
- replay protection
- optional mTLS
- strict firewall rules
- no public management API
- no default credentials

## MVP

Initial Devnet scope:

1. bootstrap one Guardian
2. pair it with one Mentor
3. create 1,000 wallets
4. return public addresses only
5. transfer SOL/SPL between selected Guardian wallets
6. verify duplicate-command protection
7. verify lost-response handling
8. let the Mentor independently reconcile results

Mainnet activation is explicitly outside the initial Guardian MVP.
