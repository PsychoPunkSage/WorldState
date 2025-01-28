# QMDB Documentation: Indexer, Twigs, and Architecture

## **Overview**
QMDB (Quick Merkle Database) is a high-performance database architecture designed for **efficient state management** in blockchain systems. It seamlessly ***integrates key-value storage with a binary Merkle tree structure***, enabling fast and scalable CRUD operations while ensuring cryptographic integrity. This document explains two key components of QMDB in detail: the **Indexer** and **Twigs**. The interplay between these components ensures that QMDB achieves its performance, scalability, and cryptographic guarantees.

### **Summary of Components**
- **Twigs**: 
  - Fixed-size subtrees of the QMDB Merkle tree.
  - Compress data for efficient in-memory operations and minimize I/O overhead.
  - Enable in-memory Merkleization with lifecycle states (Fresh, Full, Inactive, Pruned).
- **Indexer**:
  - Maps application-level keys to file offsets in Twigs for fast lookups.
  - Based on a B-Tree structure for efficient retrieval, insertion, and deletion.
  - Scales to billions of entries while minimizing DRAM usage.

---

## **Detailed Technical Documentation**

### **1. Twigs**

Twigs are the core data structure within QMDB, serving as fixed-size subtrees of the Merkle tree. Each Twig holds a fixed number of entries (2048) and plays a vital role in achieving in-memory Merkleization and efficient compression.

#### **Structure of a Twig**
Each Twig is structured as a binary Merkle tree:

```
              [Twig Root Hash]
                    |
         +----------+----------+
         |                     |
    [Internal Node 1]    [Internal Node 2]
         |                     |
    +----+----+          +-----+-----+
    |         |          |           |
[Leaf 1]   [Leaf 2]  [Leaf 3]    [Leaf 4]
```

- **Twig Root Hash**: Represents the cryptographic hash of the entire Twig, summarizing its state.
- **Internal Nodes**: Aggregate hashes of their child nodes.
- **Leaves**: Contain the actual data entries (key-value pairs).

Each entry includes:
- `Id`: Unique identifier for the entry.
- `Key`: Application key (e.g., "user123").
- `Value`: The associated state value.
- `NextKey`: The lexicographic successor of the key.
- `OldId` and `OldNextKeyId`: Metadata for historical state tracking.

#### **ActiveBits Bitmap**
Each Twig maintains an `ActiveBits Bitmap` to track the active state of its entries. Each bit in the bitmap corresponds to an entry in the Twig:

- **1**: Entry is active (part of the current world state).
- **0**: Entry is inactive (overwritten or deleted).

Example:
```
ActiveBits Bitmap: [1, 0, 1, 0]

Entries:
  Entry 1: {Key: "A", Value: 100} (Active)
  Entry 2: {Key: "B", Value: 200} (Inactive)
  Entry 3: {Key: "C", Value: 300} (Active)
  Entry 4: {Key: "D", Value: 400} (Inactive)
```

#### **Twig Lifecycle**
Twigs transition through four lifecycle states to optimize memory and storage usage:

1. **Fresh**:
   - Entire Twig resides in DRAM.
   - New entries are appended sequentially.

2. **Full**:
   - Contains 2048 entries and is flushed to SSD.
   - Only the Twig Root Hash and ActiveBits Bitmap remain in DRAM.

3. **Inactive**:
   - All entries are marked inactive.

4. **Pruned**:
   - The Twig is replaced by its Merkle Root Hash, discarding all intermediate and leaf nodes.

Lifecycle Transitions:
```
[Fresh] ---> [Full] ---> [Inactive] ---> [Pruned]
   ^                                      |
   |                                      |
   +--------------------------------------+
        Garbage collection prunes old data
```

---

### **2. Indexer**

The Indexer is a key component of QMDB, designed to map application-level keys to their physical or logical locations within Twigs.

#### **Purpose of the Indexer**
- **Efficient Lookups**: Avoids scanning the Merkle tree by directly retrieving the location of an entry.
- **CRUD Operations**: Facilitates Create, Read, Update, and Delete operations by pinpointing the exact location of entries.
- **State Proofs**: Assists in generating Merkle proofs for inclusion/exclusion.
- **Key Iteration**: Enables ordered traversal of keys.

#### **How the Indexer Works**
The Indexer uses a **B-Tree-based structure** for efficient key-to-offset mapping:
- **Keys**: Application-level keys (or their hashes).
- **Values**: File offsets pointing to the location of entries in Twigs.

<details>

<summary>
File Offset
</summary>

### **File Offset**

#### **What is File Offset?**
A **file offset** is a numerical value that indicates the position of a specific piece of data within a file stored on disk. In QMDB, it is used to locate an entry within a Twig efficiently.

#### **Purpose of File Offset**
- **Efficient Data Retrieval**:
  - Instead of traversing the entire Merkle tree or scanning the Twig, the Indexer uses the file offset to jump directly to the entry location.
- **Integration with Indexer**:
  - The Indexer maps keys to file offsets, enabling fast lookup of entries in Twigs.

#### **How File Offset is Used?**
1. **Indexer Lookup**:
   - The Indexer stores mappings from keys to file offsets (e.g., `Key: "user123" → Offset: 202`).
2. **Twig Access**:
   - The file offset identifies the entry’s location within a Twig stored on disk.
3. **Entry Retrieval**:
   - Using the offset, QMDB directly fetches the entry, avoiding unnecessary traversal.

#### **Example**
Consider a Twig file containing 4 entries stored sequentially:
```
File Structure (Twig 2):
Offset 0: Entry 1 (Key: "A")
Offset 1: Entry 2 (Key: "B")
Offset 2: Entry 3 (Key: "C")
Offset 3: Entry 4 (Key: "D")
```

Query:
- Key: "C"
- File Offset: 2

Result:
- Directly retrieve:
  ```
  Entry 3: {Key: "C", Value: 300}
  ```

#### **File Offset in CRUD Operations**
- **Create**: Append a new entry and update the Indexer with its offset.
- **Read**: Use the Indexer to locate the file offset, then retrieve the entry.
- **Update**: Mark the old entry as inactive and append a new entry at a new offset.
- **Delete**: Mark the entry at the offset as inactive.


</details><br>

**B-Tree Example**:
```
[Indexer's B-Tree]
       [Key: A | C]
      /   |     \
  [A]   [B]    [C]
 Offset Offset Offset
```

#### **Query Flow Example**
Let’s assume the user wants to fetch the balance of account "user123".

1. **Hash the Key**:
   - Key "user123" → Hash `0xA1B2C3D4`.

2. **Shard Determination**:
   - The first 2 bytes of the hash (`0xA1B2`) map the key to **Shard 3**.

```
[Global Root]
      |
+-------------+-------------+-------------+
|  Shard 1    |  Shard 2    |  Shard 3    | ...
+-------------+-------------+-------------+
```

3. **Indexer Lookup**:
   - The Indexer in Shard 3 maps the hash to a file offset in Twig 2:
     ```
     Key: 0xA1B2C3D4
     File Offset: 202
     ```

4. **Twig and Leaf Lookup**:
   - Navigate to Twig 2 and retrieve the entry from the leaf node.

```
[Twig 2 Root Hash]
       |
   +---+---+
   |       |
[Node A] [Node B]
   |         |
[Leaf X]  [Leaf Y]

(Entry: {Key: "user123", Value: "Balance=1000"})
```

5. **Result**: 
   - The user receives the balance: **1000**.

---

#### **Hybrid Indexer**
- For resource-constrained setups, QMDB offers a **Hybrid Indexer**:
  - Reduces DRAM usage to **2.3 bytes per key** by storing mappings on SSD.
  - Adds an additional SSD read per lookup.

---

### **3. Twigs vs. Indexer**

| **Aspect**    | **Twigs**                                    | **Indexer**                                  |
| ------------- | -------------------------------------------- | -------------------------------------------- |
| **Role**      | Stores actual entries in Merkle tree format. | Maps keys to entry locations.                |
| **Structure** | Binary Merkle tree.                          | B-Tree-based mapping structure.              |
| **Location**  | DRAM (root), SSD (entries).                  | DRAM (default), SSD (hybrid).                |
| **Usage**     | Ensures cryptographic integrity and proofs.  | Facilitates fast lookups and efficient CRUD. |

---

### **4. Advantages of QMDB Design**

1. **Performance**:
   - In-memory operations for active Twigs.
   - Minimal SSD I/O due to append-only updates and batching.

2. **Scalability**:
   - Supports billions of entries with low memory overhead.
   - Modular components like the Hybrid Indexer adapt to resource constraints.

3. **Cryptographic Integrity**:
   - Merkle proofs for inclusion, exclusion, and historical state.

4. **Efficiency**:
   - Compression reduces memory usage by 99.9% without data loss.

---

This document provides an in-depth technical view of QMDB’s components and processes. For further information, feel free to refer to QMDB’s implementation details or reach out for clarification.
