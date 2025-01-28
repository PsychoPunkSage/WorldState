# **Understanding Bonsai Trie**

## **Introduction**
Bonsai Trie is a `state management mechanism` introduced as an `efficient` alternative to the Merkle Patricia Trie (MPT). While MPT has been the standard for blockchain systems, it comes with significant performance and memory challenges. Bonsai Trie addresses these issues with a flat, layered structure optimized for modern blockchain needs, such as Ethereum.

---

## **How Does Bonsai Work Under the Hood?**

Bonsai Trie introduces a **flat database** approach, replacing MPT's recursive structure. This makes state reads, writes, and updates faster while reducing memory usage. Key highlights:
- **Snapshots**: Bonsai uses snapshots for consistency during transaction simulations and block imports.
- **Layered World States**: Keeps transient states isolated to avoid affecting the chain's persisted state.
- **Efficient Caching**: Uses caching to store recently accessed states, minimizing redundant operations.

> **Before Bonsai:** Each node of the trie was saved as a separate entry in the database
```js
Database Schema:
Key: Node hash (e.g., H(branch_node))
Value: Node data (e.g., branch details)
```

> **In Bonsai:** All nodes are stored in a single column, using unique keys for each trie node

```js
Database Schema:
Key: (Account Trie Location) or (Account Hash + Storage Location)
Value: Node data
```

### Diagram:
```js
// Account trie
Root (Hash: H1)
  ├── Branch (Hash: H2)
  │      └── Leaf (Key: 0x1234...abcd, Value: Balance=10 ETH)
  └── Branch (Hash: H3)
         └── Leaf (Key: 0x5678...abcd, Value: Balance=5 ETH)

// Storage Trie
Root (Hash: H4)
  ├── Branch (Hash: H5)
  │      └── Leaf (Key: Hash(0x9876...efgh):Slot0, Value: 42) // Slot0 - Hash(0xab)
  └── Branch (Hash: H6)
         └── Leaf (Key: Hash(0x9876...efgh):Slot1, Value: 100) // Slot1 - Hash(0xcd)
```

## **Flat Database**
Bonsai employs a **flat database** to store key-value pairs. This eliminates the overhead of MPT’s recursive structure and hashing at each node, leading to:
- Faster lookups and updates.
- Reduced storage requirements.
- Simpler state management.

| traditional                                                                | Bonsai                                                                                                                    |
| -------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| To fetch an account’s balance, the EVM must traverse multiple trie levels. | The `account hash` directly maps to the balance, requiring only **`one disk read`**.                                      |
| Involves several disk reads and hash computations.                         | Saves time and computational effort, especially for operations like **SLOAD**, which are heavily used in smart contracts. |

### **DataStorage**
| Column   | Key                 | Value                |
| -------- | ------------------- | -------------------- |
| Accounts | Account hash        | Balance, Nonce, etc. |
| Storage  | Account hash + Slot | Data at that slot    |
| Code     | Account hash        | Bytecode             |


### **Analogy**:  
Think of MPT as a **library with multiple floors** (complex navigation) and Bonsai as an **e-library** (compact and accessible).

### **Diagram**:  
Two side-by-side representations:  
- **MPT**: A hierarchical tree with nodes and hashes.  
- **Bonsai**: A flat table with direct key-value pairs.

```
MPT (Merkle Patricia Tree)                Bonsai (Flat Table)
--------------------------                -------------------
         Root                                  +------------+
         /   \                                 | Key | Value|
     Hash1   Hash2                             +------------+
     /  \     |                                | K1  | V1   |
  Key1 Value1 Key2                             | K2  | V2   |
                                               | K3  | V3   |
                                               +------------+
================================================================
================================================================
================================================================

    Before Bonsai:             With Bonsai:
    -----------------          -----------------
    [Hash -> Node Data]        [Location -> Node Data]
    [Hash -> Node Data]        [Location -> Node Data]
    Old nodes kept forever.    Old nodes replaced.

```

<details>
<summary>
More detailed
</summary>

```js
// WO FlatDB
Root
 ├── Branch (Hash: H1)
 │      ├── Branch (Hash: H2)
 │      │      └── Leaf (Key: 0x1234...abcd, Value: Balance=100 ETH)
 │      └── Leaf (Key: 0x5678...efgh, Value: Balance=50 ETH)
 └── Branch (Hash: H3)
        └── Leaf (Key: 0x9abc...def0, Value: Balance=25 ETH)


// With FlatDB

Accounts Column:
Key: Hash(0x1234...abcd)
Value: { balance: 100 ETH, nonce: 5 }

Storage Column:
Key: Hash(0x5678...efgh) + Hash(0xA)
Value: 42

Code Column:
Key: Hash(0x9876...ijkl)
Value: 0x6001600055
```

</details>

---

## **Trielogs**
In Bonsai, **trielogs** are lightweight logs used for tracking changes in the state. Unlike the MPT's reliance on recalculating hashes, trielogs:
- Record incremental state changes.
- Enable efficient rollbacks and roll-forwards without recomputing the entire state.

### **Diagram**:  
A flowchart showing trielogs capturing changes (state deltas) and applying them during rollback/roll-forward.

```javascript
Bonsai TrieLogs Mechanism
--------------------------

State Changes --> [ Delta1 ] --> [ Delta2 ] --> [ Delta3 ] 
                     |              |              |
                     V              V              V
                [ TrieLog1 ]   [ TrieLog2 ]   [ TrieLog3 ]
                     |              |              |
Rollback <---------- [ Apply Backward ] <--------- [ Apply Backward ]
                     |              |              |
Roll-Forward ------> [ Apply Forward ] ---------> [ Apply Forward ]
```

<details>
<summary>
Actual TrieLogs
</summary>

```
Block Hash: 0xabcdef1234567890...

Trielog:

  - Account: 
    - Account address: 0x1234567890abcdef...
    - Previous Value: { balance: 100, nonce: 0 , ...}
    - Next Value: { balance: 150, nonce: 1 , ...}

  - Storage Slot:

    - Account address: 0x1234567890abcdef...
    - Slot key: 0x0111...
    - Previous Value: 0x01
    - Next Value: 0x02

  - Code:

    - Account address: 0x1234567890abcdef...
    - Previous Value: 0xaabbccddeeff...
    - Next Value: 0x112233445566...
```

</details>

---

## **Accumulator**
Bonsai uses an **accumulator** to group multiple state changes for efficient batch processing. This approach:
- Reduces write operations by combining updates.
- Maintains consistency across transactions.

### **Diagram**:  
Visualize an accumulator as a funnel collecting updates before writing them to the state.

```js
Bonsai Accumulator Mechanism
----------------------------

State Changes --> [ Update1 ] --> [ Update2 ] --> [ Update3 ]
                         |               |               |
                         V               V               V
                    +---------------------------------------+
                    |           Accumulator                |
                    +---------------------------------------+
                                  |
                                  V
                            [ Batched Write ]
                                  |
                                  V
                              [ Final State ]
```

---

## **Rollback/Rollforward**
Bonsai Trie supports **rollback and roll-forward** mechanisms for robust state management:
- **Rollback**: Reverts to a previous snapshot during failed transactions or forks.
- **Roll-forward**: Advances to the next valid state during chain reorganization.

### **Diagram**:  
Illustrate snapshots at different blocks (e.g., Block #4 and Block #9) with arrows showing rollback and roll-forward paths.
```js
Rollback/Roll-forward in Bonsai Trie
------------------------------------

Block #1 ---> Block #2 ---> Block #3 ---> Block #4 ---> Block #5 ---> Block #6 ---> Block #9
                          ^                                    |                     ^
                          |                                    |                     |
                          |--------- Rollback -----------------|---- Roll-forward ---|
                          
Snapshots: [ Block #3 Snapshot ]                   [ Block #5 Snapshot ]     [ Block #9 Snapshot ]

```
---

## **Layered State vs. Trielog**
- **Layered State**: Temporary states that stack over the persisted state, allowing isolated modifications.
- **Trielog**: Incremental logs that track state changes for efficient updates without recalculating hashes.

### **Key Difference**:
Layered state is used during simulations, while trielogs manage updates efficiently without state recomputation.

### **Diagram**:  
Show layered states as multiple transparent sheets stacked over a base and trielogs as a linear change tracker.
```js
Layered State vs. Trielog
-------------------------

Layered State:                                 Trielog:
---------------
|             |                                 -------------------------
| Layer #1    |                                 | Delta1 (State Change) |
|             |                                 -------------------------
---------------
|             |                                 -------------------------
| Layer #2    |                                 | Delta2 (State Change) |
|             |                                 -------------------------
--------------
|             |                                 -------------------------
| Layer #3    |                                 | Delta3 (State Change) |
|             |                                 -------------------------
---------------                                        |
          |                                            |
          |----------> Base Persisted State <----------|

```

<details>
<summary>
Differences in depth
</summary>


| **Aspect**          | **Layered State**                                                                                                                                                                                                                                                                                     | **Trielog**                                                                                                                                                                                                                                                                       |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Definition**      | A representation of the world state where incremental changes (diffs) to the state are applied on top of a snapshot or persisted state.                                                                                                                                                               | A data structure that records changes (deltas) between two states, including prior and next values for each modification.                                                                                                                                                         |
| **Purpose**         | To provide a temporary and efficient way to manage and query state changes without modifying the underlying persisted state directly.                                                                                                                                                                 | To provide a history of changes that enables efficient rollbacks (undo) and rollforwards (redo) in response to chain reorganizations or other state adjustments.                                                                                                                  |
| **Persistence**     | Temporary (in-memory). Changes are not persisted to the database until explicitly written.                                                                                                                                                                                                            | Persistent. Stored in the database keyed by block hash.                                                                                                                                                                                                                           |
| **Structure**       | Combines the base state (snapshot or persisted) with an in-memory diff for the current state.                                                                                                                                                                                                         | Contains prior and next values for every modified element to track state changes between two blocks.                                                                                                                                                                              |
| **Granularity**     | Represents the whole state with applied diffs, allowing access to the current state.                                                                                                                                                                                                                  | Tracks specific changes at a granular level (e.g., accounts, storage slots, code modifications).                                                                                                                                                                                  |
| **Characteristics** | - Represents the state as base + in-memory diff. <br> - Efficient for incremental updates. <br> - Used for temporary state changes during execution. <br> - Supports parallel processing by isolating modifications in memory. <br> - Does not affect the persisted state until explicitly committed. | - Records deltas between blocks. <br> - Stores both prior and next values for modifications. <br> - Persistent structure stored in the database. <br> - Does not represent the current state but tracks transitions between states. <br> - Used during rollbacks or rollforwards. |
| **Usage**           | - Efficiently manage and access the current state during block processing. <br> - Facilitate parallel execution by isolating state changes in memory.                                                                                                                                                 | - Enable precise rollback to a previous state during a chain reorganization. <br> - Facilitate rollforward to a future state using recorded modifications.                                                                                                                        |
| **Scope**           | Temporary, dynamic, and for representing the state at a specific moment with applied diffs.                                                                                                                                                                                                           | Historical record of state transitions between blocks, enabling restoration or advancement of state.                                                                                                                                                                              |
| **Key Difference**  | Represents incremental changes on top of a base state, useful for current state access and updates.                                                                                                                                                                                                   | Records state changes over time to manage transitions efficiently for rollbacks or rollforwards.                                                                                                                                                                                  |

</details>

---

## **Life Cycle of the World State**
The world state in Bonsai Trie goes through a specific life cycle to ensure consistency and avoid memory leaks:
1. **Snapshot Creation**: A snapshot of the persisted state is created after importing a block or during system startup.
2. **State Access**: Requests for world state are handled via cached snapshots.
3. **State Closure**: Unused states are evicted from the cache and closed to prevent memory leaks.

### **Diagram**:  
A flowchart of the Bonsai Trie life cycle:
- **Start** → **Import Block** → **Snapshot Creation** → **Simulation/Validation** → **State Closure**.

```js
Bonsai Trie World State Life Cycle
---------------------------------

   Start
     |
     V
Import Block --> Snapshot Creation --> Simulation/Validation --> State Access (Cached Snapshot)
                                                 |
                                                 V
                                        State Closure (Eviction)

```
* **Start:** The beginning of the process, either during system startup or upon block import.
* **Import Block:** New data is imported into the system.
* **Snapshot Creation:** A snapshot of the persisted state is created after importing the block.
* **Simulation/Validation:** The state is simulated or validated for consistency.
* **State Access:** World state requests are handled through cached snapshots for efficiency.
* **State Closure:** Unused snapshots are evicted from the cache to prevent memory leaks.
---

## **Why Bonsai is Better than MPT**

| **Feature**            | **Merkle Patricia Trie (MPT)**       | **Bonsai Trie**                    |
| ---------------------- | ------------------------------------ | ---------------------------------- |
| **Structure**          | Recursive tree                       | Flat key-value database            |
| **Performance**        | Slower due to hashing at every node  | Faster due to direct lookups       |
| **Memory Usage**       | High due to redundant nodes          | Low with efficient caching         |
| **Rollback Mechanism** | Requires full recomputation of state | Snapshot-based, fast and efficient |
| **Use Case**           | Legacy systems                       | Modern blockchain requirements     |

---

## **Conclusion**
Bonsai Trie is a significant improvement over MPT, offering better performance, memory efficiency, and state management capabilities. It is designed to handle the challenges of modern blockchain systems, making it a preferred choice for platforms like Ethereum.

### **Closing Analogy**:  
**MPT**: A bulky traditional map—useful but cumbersome.  
**Bonsai Trie**: A GPS navigation system—efficient and precise.

---