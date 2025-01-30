# QMDB Storage Architecture Documentation

## Table of Contents
- [Overview](#overview)
- [Key Components](#key-components)
- [Data Flow](#data-flow)
- [Storage Architecture](#storage-architecture)
- [Implementation Details](#implementation-details)
- [Comparison with Traditional Systems](#comparison-with-traditional-systems)
- [Performance Considerations](#performance-considerations)

## Overview

QMDB is a high-performance key-value storage system that combines efficient indexing with optimized storage management. It uses a hybrid approach that leverages BTree indexing for fast lookups while maintaining data in optimized leaf structures.

### Key Features
- O(1) data access after initial BTree lookup
- Efficient space utilization through leaf-based storage
- Automatic storage optimization via compaction
- Concurrent access support through sharding

## Key Components

### 1. InMemory Indexer
The main coordinator for data storage operations, managing 65,536 independent units.

```
InMemory Indexer Structure:
┌─────────────────────────────────────────────────┐
│                  InMemIndexer                   │
├─────────────────────────────────────────────────┤
│    ┌─────────┐ ┌─────────┐      ┌─────────┐     │
│    │ Unit 0  │ │ Unit 1  │ ...  │Unit 65K │     │
│    └─────────┘ └─────────┘      └─────────┘     │
└─────────────────────────────────────────────────┘
```

### 2. Unit Structure
Each unit contains:
- BTreeMap for key-to-position mapping
- BufList for physical data storage
- Management counters

```rust
pub struct Unit {
    bt: BTreeMap<u64, u64>,     // Maps keys to positions
    buf_list: BufList,          // Buffer storage
    last_size: u32,             // Size tracking
    change_count: u32,          // For compaction
    temp_vec: Vec<u8>,          // Temporary storage
}
```

### 3. Key Structure
80-bit keys are structured as follows:

```
80-bit Key Structure:
┌───────┬─────────────────┬───────┐
│ AABB  │ CCDDEEFF112233  │  33   │
│(16bit)│    (56bit)      │(8bit) │
└───────┴─────────────────┴───────┘
   ↓           ↓              ↓
Unit Index  Lookup Value    Version
```

## Data Flow

### 1. Initial Request Processing

When adding data (e.g., user balance):

```rust
// Example: Adding user balance
indexer.add_kv("user123", 100, sequence_number)
```

### 2. Key Generation Process

```
Input Key Processing:
"user123" → Hash Function → 80-bit key

Example:
"user123" → 0xAABBCCDDEEFF112233

Split into:
[AABB]          [CCDDEEFF112233]    [33]
Unit Index      Lookup Value       Extra Data
```

### 3. Physical Storage Structure

#### Buffer Organization
```
Buffer (10KB - 16 bytes each):
┌────────────────────────────────────┐
│ Buffer Structure                   │
├──────┬──────┬──────┬──────────────-┤
│Leaf 1│Leaf 2│Leaf 3│   Empty...    │
└──────┴──────┴──────┴──────────────-┘
```

#### Leaf Structure
```
┌─────────┬─────────┬───────────────┐
│Capacity │ Count   │   KV Pairs    │
│ (2B)    │ (2B)    │   (12B each)  │
└─────────┴─────────┴───────────────┘
     │         │            │
     v         v            v
    32        5      ┌────────────┐
                     │ Key (6B)   │
                     │ Value (6B) │
                     └────────────┘
```

## Implementation Details

### 1. Position Encoding
64-bit position values encode storage location:

```
64-bit Position: 0x0001000000000200
┌────────────────┬────────────┬────────────┐
│ Buffer Index   │  Reserved  │   Offset   │
│   (32 bits)    │  (16 bits) │  (16 bits) │
└────────────────┴────────────┴────────────┘
```

### 2. Leaf Operations

#### New Leaf Creation
```
┌──────────────────────────────┐
│ Capacity: 8   Count: 1       │
├──────────────────────────────┤
│ Key: CCDDEEFF112233          │
│ Value: 100                   │
└──────────────────────────────┘
```

#### Leaf Split Process
```
Before Split (Full):
┌────────────────────────┐
│ Count: 8/8             │
└────────────────────────┘

After Split:
┌────────────────────┐    ┌────────────────────┐
│ Leaf 1 (4 entries) │    │ Leaf 2 (5 entries) │
└────────────────────┘    └────────────────────┘
```

### 3. Important Constants

```rust
const LEAF_INIT_CAP: usize = 8;
const DEFAULT_LEAF_CAP: usize = 32;
const EXTRA_KV_PAIRS: usize = 2;
const DEFAULT_LEAF_LEN: usize = DEFAULT_LEAF_CAP - EXTRA_KV_PAIRS;
const BUF_SIZE: usize = 1024 * 10 - 16;
const K_SIZE: usize = 6;
const V_SIZE: usize = 6;
const KV_SIZE: usize = K_SIZE + V_SIZE;
```

## Comparison with Traditional Systems

### QMDB vs Traditional MPT (Ethereum)

```
Traditional MPT (Ethereum):
┌────────────────────────────────────┐
│ - O(log n) operations              │
│ - Multiple node traversals         │
│ - Path-dependent lookups           │
│ - Storage overhead for paths       │
└────────────────────────────────────┘

QMDB:
┌────────────────────────────────────┐
│ - O(1) after BTree lookup          │
│ - Direct leaf access               │
│ - Position-based jumping           │
│ - Compact storage format           │
└────────────────────────────────────┘
```

## Performance Considerations

### 1. Compaction Process
Triggered when:
```rust
change_count * COMPACT_RATIO > max(last_size, COMPACT_THRES)
```

Where:
```rust
const COMPACT_RATIO: u32 = 20;
const COMPACT_THRES: u32 = 500;
```

### 2. Concurrency
- Each Unit is protected by RwLock
- Independent unit operations
- Lock-free reads

### 3. Memory Management
- Buffer size: 10KB - 16 bytes
- Automatic buffer allocation
- Efficient space utilization

## Data Operations Comparison

### Locating Data
1. Generate 80-bit key
2. Extract unit index
3. BTreeMap lookup
4. Find leaf
5. Scan for exact key
6. Return value

### Adding Data
1. Generate 80-bit key
2. Extract unit index
3. BTreeMap lookup
4. If leaf exists:
   - Check capacity
   - Split if needed
   - Insert data
5. If no leaf:
   - Create new leaf
   - Add to BTreeMap
6. Update metadata

## Sequence Numbers
- Monotonically increasing
- Used for transaction ordering
- Ensures consistency
- Assists in:
  - Concurrency control
  - Version tracking
  - Conflict resolution

## Complete System Overview
```
                    QMDB Storage System
┌─────────────────────────────────────────────────┐
│                  InMemIndexer                   │
├─────────────────────────────────────────────────┤
│                                                 │
│    ┌─────────┐ ┌─────────┐      ┌─────────┐     │
│    │ Unit 0  │ │ Unit 1  │ ...  │Unit 65K │     │
│    └────┬────┘ └─────────┘      └─────────┘     │
│         │                                       │
│    ┌────┴────┐                                  │
│    │ BTreeMap│                                  │
│    └────┬────┘                                  │
│         │                                       │
│    ┌────┴────┐                                  │
│    │ BufList │                                  │
│    └────┬────┘                                  │
│         │                                       │
│    ┌────┴────┐ ┌─────────┐     ┌─────────┐      │
│    │Buffer 0 │ │Buffer 1 │ ... │Buffer N │      │
│    └─────────┘ └─────────┘     └─────────┘      │
└─────────────────────────────────────────────────┘
```
