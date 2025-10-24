# Transactions Inclusion Tree - Code Explanation

## Purpose

This code implements a **Merkle Accumulator Tree** for proving that transactions were included in a blockchain and that events were emitted. SIMILAR to `cryptographic receipt system`.

---

## Key Concepts

### What It Does

- **Accumulates transactions** into a tree structure
- **Generates proofs** that a transaction was included
- **Proves events** were emitted by transactions
- **Persists data** to disk using RocksDB

### How It Works

1. Transactions get hashed and added as leaves to the tree
2. Each transaction's events are also hashed into a mini-tree
3. The tree root acts as a fingerprint of all transactions
4. Proofs allow anyone to verify a transaction existed without seeing all data

---

## Main Structure

### TransactionsInclusionTree

The core data structure that maintains the Merkle tree.

**Fields:**

- `root` - Current tree root hash (None if empty)
- `transaction_hash_value_to_position` - Quick lookup: transaction hash → tree position
- `num_existing_leaves` - Count of transactions in tree
- `on_disk_storage` - Persistent storage layer
- `db` - RocksDB database reference

---

## Key Functions

### `load_tree()`

Loads existing tree from disk storage. Rebuilds the in-memory state from persisted nodes.

### `insert()`

Adds new transactions to the tree.

- Takes: transaction hashes + their event hashes, epoch number, database
- Returns: New root hash
- Process: Creates leaf nodes, updates tree, saves to disk, prunes old data

### `prove_transaction_inclusion()`

Generates proof that a transaction exists in the tree.

- Takes: transaction hash
- Returns: `TransactionInclusionProof`

### `prove_event_emission()`

Generates proof that an event was emitted.

- Takes: transaction hash, event hash
- Returns: `EventEmissionProof`

---

## Structs in JSON Format

```json
{
  "TransactionInclusionProof": {
    "proof": "AccumulatorProof<TransactionAccumulatorHasher>",
    "leaf_hash_value": "HashValue",
    "current_root": "HashValue",
    "index": "u64"
  },

  "EventEmissionProof": {
    "proof": "AccumulatorProof<EventAccumulatorHasher>",
    "merkle_root_hash_value": "HashValue",
    "leaf_hash_value": "HashValue"
  },

  "FrozenNode": {
    "0": "HashValue",
    "1": "Option<HashValue>",
    "2": "Option<EventsLeaves>"
  },

  "EventsLeaves": {
    "0": "Vec<HashValue>"
  },

  "PruneInfo": {
    "epoch": "Epoch",
    "snapshot_first_post_order_index": "u64"
  }
}
```

---

## Internal Helper Structures

### `BTreeMapHashReader`

Reads frozen node hashes from a BTreeMap. Implements the `HashReader` trait needed by the Merkle accumulator.

### `FrozenNode`

Represents an immutable node in the tree.

- Hash of the node itself
- Optional transaction hash (only for leaf nodes)
- Optional events leaves (only for leaf nodes)

### `EventsLeaves`

Wrapper around event hashes for a transaction. Can compute Merkle root of all events.

---

## Storage Layer

The `Storage<S, PS>` handles:

- **Reading** frozen nodes from disk
- **Writing** new nodes in batches
- **Pruning** old data by epoch

Uses RocksDB schemas:

- `S` - Maps node index → FrozenNode
- `PS` - Maps epoch → last pruned index

---

## Key Algorithms

### Hashing

- **Transaction leaf hash** = Hash(transaction_hash + events_merkle_root)
- **Events merkle root** = Merkle tree root of all event hashes
- Uses cryptographic hash functions from `aptos_crypto`

### Proof Verification

Proofs contain the minimum hashes needed to recompute the root. If recomputed root matches, proof is valid.

---

## Summary Flow

```
1. Transactions executed → Hashes generated
2. insert() called → New leaves added to tree
3. Tree updated → New root computed
4. Nodes saved → Written to RocksDB
5. Old epochs pruned → Disk space freed

Later:
6. User requests proof → prove_transaction_inclusion() called
7. Proof generated → Minimal hash path returned
8. Proof verified → Root recomputed and checked
```
