# Exit Tree Generator

> **⚠️ Deprecation Notice**
>
> ZKSync Lite is being deprecated. To withdraw your funds, please visit
> [lite.zksync.io](https://lite.zksync.io).
>
> This tool is provided for users who wish to generate their own withdrawal proofs independently, or to verify the
> exit tree against the on-chain state.

This tool helps users withdraw their tokens from ZKSync Lite using an exit Merkle tree. It restores the ZKSync Lite
state, generates the exit tree leaves, calculates Merkle roots, and produces proofs that can be submitted to the
withdrawal contract to claim funds.

## How `exit_tree_generator` Works

The withdrawal flow has three steps:

### 1. Download the data

Download the snapshot of the ZKSync Lite state at the last verified block from:

https://zksync-lite-sunset.matterlabs.dev/mainnet/data.zip

or IPFS CID:
QmSqzPQNRcz3grznTMCYDDunDFGZCXFFjew7fgYpFcGYoh

The archive contains:

- `accounts.csv` — account state (account ID, nonce, Ethereum address, public key hash)
- `balances.csv` — balances per account/token (account ID, token ID, balance)
- `tokens.csv` — token ID → Ethereum address mapping

Place these files in the working directory before running any of the commands below.

### 2. Create leaves for the Keccak tree

To generate the leaves from `accounts.csv`, `balances.csv`, and `tokens.csv`:

```bash
cargo run --release --bin zksync_exit_tree_generator -- create-new-leaves \
    --accounts accounts.csv \
    --balances balances.csv \
    --tokens tokens.csv \
    [--output new_leaves.csv]
```

### 3. Create a proof and claim

Generate a Merkle proof for your account and the token(s) you want to withdraw:

```bash
cargo run --release --bin zksync_exit_tree_generator -- create-proof \
    --account <YOUR_ACCOUNT_ADDRESS> \
    --tokens <TOKEN_ADDRESS_1> [<TOKEN_ADDRESS_2> ...] \
    [new_leaves.csv]
```

You can pass multiple token addresses in one invocation to produce a single combined proof. The proof is printed as a
hex-encoded string.

Submit the printed proof to the withdrawal contract.

**Mainnet:**: https://etherscan.io/address/0x0a14b696350546110a0d8acdb86226983af9d2a0

The contract exposes two methods for claiming:

- **`claim`** — withdraws the funds to the caller (`msg.sender`). Use this when the wallet submitting the
  transaction is the address that held the ZKSync Lite account.
- **`claimTo`** — withdraws the funds to a specified recipient address. Use this when you want the funds sent
  somewhere other than the caller (for example, a cold wallet or a different EOA).

Both methods take the account address, the token addresses, and the proof produced in step 3. You can
call them from Etherscan's "Write Contract" tab, or from any wallet/script that can encode a contract call.

## Advanced

These workflows are for users who want to independently verify the exit tree rather than trust the published data.

### Calculating the Keccak Merkle root

The Keccak tree is the exit tree used by the withdrawal contract. Its root is published on-chain. You can recompute
it locally from `new_leaves.csv` and compare:

```bash
cargo run --release --bin zksync_exit_tree_generator -- calculate-root-for-keccak-tree \
    [new_leaves.csv]
```

This is fast and runs in memory.

### Calculating the original ZKSync Merkle root

The original ZKSync Lite Merkle tree (Rescue-based) can be reconstructed from `accounts.csv` and `balances.csv`. Its
root matches the state root of the last verified block on the ZKSync Lite contract, so this is the strongest possible
verification that the snapshot data is authentic.

```bash
cargo run --release --bin zksync_exit_tree_generator -- restore-zksync-tree \
    --accounts accounts.csv \
    --balances balances.csv
```

> **Heads up:** This is a heavy computational task. It takes approximately **6 hours** on a typical machine and uses
> significant memory. Intermediate node hashes are written to `internals.txt` to speed up subsequent runs.

The command prints the restored root hash, which you can compare against the value reported by the ZKSync Lite
contract.


### Restoring the tree from a database

For users with direct access to the original database state:

```bash
cargo run --release --features postgres --bin zksync_exit_tree_generator -- restore-tree-from-db
```

Requires the `postgres` feature flag and a `DATABASE_URL` environment variable.

## Commands Reference

### `create-new-leaves`

Creates new leaves for the exit Merkle tree.

**Options:**

- `--accounts <PATH>`: Path to accounts CSV file (required)
- `--balances <PATH>`: Path to balances CSV file (required)
- `--tokens <PATH>`: Path to tokens CSV file (required) — must use the official snapshot's `tokens.csv`
- `--output <PATH>`: Optional output file path (default: `new_leaves.csv`)

### `calculate-root-for-keccak-tree`

Calculates the Keccak Merkle root from the leaves file.

**Arguments:**

- `<LEAVES_PATH>`: Path to leaves CSV file (optional, default: `new_leaves.csv`)

### `create-proof`

Generates a Merkle proof for a specific account and one or more tokens.

**Options:**

- `--account <ADDRESS>`: Ethereum address of the account (required)
- `--tokens <ADDRESS>...`: Ethereum address(es) of the token(s) (required, can specify multiple)
- `<LEAVES_PATH>`: Path to leaves CSV file (optional, default: `new_leaves.csv`)

### `restore-zksync-tree`

Restores the original ZKSync Lite Merkle tree and prints its root hash.

**Options:**

- `--accounts <PATH>`: Path to accounts CSV file (required)
- `--balances <PATH>`: Path to balances CSV file (required)


### `restore-tree-from-db` (PostgreSQL feature)

Restores the Merkle tree directly from the verified database state. Requires the `postgres` feature flag and the
`DATABASE_URL` environment variable.

## Building

```bash
cargo build --release --bin zksync_exit_tree_generator
```

With PostgreSQL support:

```bash
cargo build --release --features postgres --bin zksync_exit_tree_generator
```

## Output Files

- `new_leaves.csv`: Default output file containing Merkle tree leaves
- `restored_tokens.csv`: Output file from token ID restoration
- `internals.txt`: Internal node hashes used to speed up subsequent tree recalculations

## Notes

- All addresses should be provided in standard Ethereum hex format (`0x...`)
- `tokens.csv` from the official snapshot includes both fungible and non-fungible tokens; only fungible tokens are
  re-fetched by `restore-token-ids`
- `restore-zksync-tree` is computationally intensive (~6 hours)
