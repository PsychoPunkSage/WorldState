# Storage Module - Code Explanation

## Purpose

This module handles **persistent storage** of the Merkle tree nodes in RocksDB. It manages reading, writing, and pruning of transaction inclusion data.

---

## Key Concepts

### What It Does

- **Stores** frozen tree nodes to disk (RocksDB)
- **Retrieves** nodes when needed for proofs
- **Prunes** old data by epoch to save disk space
- **Manages** separate storage for MoveVM and EVM transactions

### Storage Architecture

Two types of data are stored:

1. **Accumulator Data** - The actual tree nodes (PostOrderIndex → FrozenNode)
2. **Pruner Metadata** - Tracks last index per epoch (Epoch → PostOrderIndex)

---

## Column Families (Tables)

### MoveVM Storage

- `tx_inc_acc_mvm` - MoveVM transaction accumulator nodes
- `tx_inc_prn_mvm` - MoveVM pruner metadata

### EVM Storage

- `tx_inc_acc_evm` - EVM transaction accumulator nodes
- `tx_inc_prn_evm` - EVM pruner metadata

Each VM type gets its own isolated storage.

---

## Schemas Defined

### MovevmTransactionsInclusionAccumulatorSchema

Stores MoveVM tree nodes.

- **Key**: `PostOrderIndex` (u64) - encoded as big-endian bytes
- **Value**: `FrozenNode` - encoded as BCS (Binary Canonical Serialization)

### EvmTransactionsInclusionAccumulatorSchema

Stores EVM tree nodes.

- **Key**: `PostOrderIndex` (u64) - encoded as big-endian bytes
- **Value**: `FrozenNode` - encoded as BCS

### MovevmTransactionsInclusionPrunerSchema

Tracks MoveVM pruning progress.

- **Key**: `Epoch` - encoded as big-endian bytes
- **Value**: `PostOrderIndex` (u64) - encoded as BCS

### EvmTransactionsInclusionPrunerSchema

Tracks EVM pruning progress.

- **Key**: `Epoch` - encoded as big-endian bytes
- **Value**: `PostOrderIndex` (u64) - encoded as BCS

---

## Main Structure

### Storage<S, PS>

Generic storage handler with two schema type parameters.

**Type Parameters:**

- `S` - Accumulator schema (stores nodes)
- `PS` - Pruner schema (stores epoch metadata)

**Fields:**

- `_schema_type_holder` - PhantomData to track accumulator schema type
- `_pruner_schema_type_holder` - PhantomData to track pruner schema type

**Note:** Uses PhantomData because the actual schema types are encoded in the generic parameters, not as struct fields.

---

## Key Functions

### `new()`

Creates a new storage instance. No actual DB connection here - just initializes the type.

### `read()`

Reads all nodes from disk into memory.

- **Input**: Database reference
- **Output**: `BTreeMap<PostOrderIndex, FrozenNode>` - all stored nodes
- **Process**: Iterates through column family, decodes key-value pairs

### `insert()`

Writes multiple nodes to disk in a batch.

- **Input**: Key-value map, current epoch, database
- **Output**: Result
- **Process**:
  1. Creates write batch
  2. Adds all nodes to accumulator table
  3. Records last index in pruner table
  4. Commits batch atomically

### `prune()`

Deletes old nodes to free disk space, using **Mersenne number optimization**.

- **Input**: Epoch to prune, database
- **Output**: Result
- **Process**:
  1. Gets last index for the epoch
  2. Calculates safe pruning boundary using Mersenne numbers
  3. Deletes nodes below boundary
  4. Keeps nodes needed for future proofs

### `get_node_hash_value()`

Retrieves just the hash value for a specific position.

- **Input**: Position in tree, database
- **Output**: `Option<HashValue>`
- **Use Case**: Efficient when you only need the hash, not full node data

### `get()`

Retrieves full node data for a position.

- **Input**: Position in tree, database
- **Output**: Full `FrozenNode`
- **Use Case**: When you need complete node information

---

## Structs in JSON Format

```json
{
  "Storage": {
    "_schema_type_holder": "PhantomData<S>",
    "_pruner_schema_type_holder": "PhantomData<PS>"
  }
}
```

---

## Error Types

```json
{
  "Error": {
    "variants": [
      "ReadFailure(RocksDbError)",
      "WriteFailure(RocksDbError)",
      "KeyDecodeFailure(SchemaError)",
      "ValueDecodeFailure(SchemaError)",
      "KeyEncodeFailure(SchemaError)",
      "ValueEncodeFailure(SchemaError)",
      "SchemaFailure(SchemaError)",
      "PrunningFailure",
      "ValueMissing(Position)"
    ]
  }
}
```

---

## Table Setup Parameters

Both accumulator and pruner tables use these settings:

```json
{
  "TableSetupParameters": {
    "l1_cache_capacity": "DEFAULT_L1_CACHE_CAPACITY",
    "l2_cache_capacity": "DEFAULT_L2_CACHE_CAPACITY",
    "prunable": "false",
    "enable_callbacks": "false",
    "upgradable": "false"
  }
}
```

**Note:** Accumulator tables are marked as NOT prunable at the table level because pruning is handled manually via the `prune()` function.

---

## Encoding Schemes

### Keys

- **Big-Endian encoding** (`define_key_codec_be!`)
- Ensures proper sorting in RocksDB
- Used for: PostOrderIndex, Epoch

### Values

- **BCS encoding** (`define_value_codec_bcs!`)
- Binary Canonical Serialization
- Deterministic, compact format
- Used for: FrozenNode, PostOrderIndex (when stored as value)

---

## Pruning Algorithm Explained

The pruning uses **Mersenne numbers** for optimization:

```
Mersenne Number = 2^n - 1
Examples: 3, 7, 15, 31, 63, 127, 255...
```

### Why Mersenne Numbers?

In a binary tree, these positions represent **complete subtrees**. Pruning up to a Mersenne boundary ensures:

- All proofs for unpruned transactions still work
- No broken references in the tree structure
- Can continue adding new transactions

### Pruning Steps

1. Get last leaf index from epoch
2. If index > 3, find next power of 2
3. Calculate safe boundary: `next_power_of_2 - 1 - 2`
4. Delete all nodes with index < boundary
5. Keep 2 extra nodes for left sibling proofs

### Example

```
Last leaf index: 100
Next power of 2: 128
Mersenne: 127
Minus shift: 125
Delete nodes: 0 to 125
```

---

## Usage Pattern

```rust
// Initialize storage
let storage = Storage::<
    MovevmTransactionsInclusionAccumulatorSchema,
    MovevmTransactionsInclusionPrunerSchema
>::new();

// Read all nodes
let nodes = storage.read(&db)?;

// Insert new nodes
let mut batch = BTreeMap::new();
batch.insert(42, frozen_node);
storage.insert(&batch, current_epoch, &db)?;

// Prune old data
storage.prune(old_epoch, &db)?;

// Get specific node
let node = storage.get(&position, &db)?;
```

---

## Type Aliases

```json
{
  "Result<T>": "StdResult<T, Error>"
}
```

---

## Constants Reference

```rust
DEFAULT_MOVEVM_TRANSACTIONS_INCLUSION_ACCUMULATOR_COLUMN_FAMILY = "tx_inc_acc_mvm"
DEFAULT_EVM_TRANSACTIONS_INCLUSION_ACCUMULATOR_COLUMN_FAMILY = "tx_inc_acc_evm"
DEFAULT_MOVEVM_TRANSACTIONS_INCLUSION_PRUNER_COLUMN_FAMILY = "tx_inc_prn_mvm"
DEFAULT_EVM_TRANSACTIONS_INCLUSION_PRUNER_COLUMN_FAMILY = "tx_inc_prn_evm"

SECOND_MERSENN_NUMBER = 3
MERSENNE_SHIFT = 1
SHIFT_TO_SAVE_LEFT_SIBLING_USED_IN_PROOFS = 2
```

---

## Key Design Decisions

### Separation of MoveVM and EVM

Each VM type gets its own storage to:

- Isolate different transaction types
- Allow independent pruning strategies
- Prevent cross-contamination

### Batch Writes

All writes are batched for:

- **Atomicity** - all succeed or all fail
- **Performance** - single disk write
- **Consistency** - epoch metadata always matches data

### Manual Pruning

Pruning is explicit, not automatic, to:

- Give control over when pruning happens
- Avoid mid-operation data loss
- Allow verification before deletion

---

## Summary Flow

```
Write Flow:
1. Accumulate nodes in BTreeMap
2. Call insert() with all nodes + epoch
3. Nodes encoded and batched
4. Pruner metadata updated
5. Atomic commit to disk

Read Flow:
1. Call read() on storage
2. Iterator scans column family
3. Keys/values decoded
4. Returns in-memory BTreeMap

Prune Flow:
1. Identify epoch to prune
2. Get last index for epoch
3. Calculate Mersenne boundary
4. Delete range of old nodes
5. Keep necessary nodes for proofs
```
