# Event Bus & Committee Certifier - Code Explanation

## ⚠️ Proof-Related Content Status

**Neither of these files contain transaction/event inclusion proof logic.**

However, they DO contain:

- ✅ **Certificate generation** (committee authorization proofs)
- ✅ **Vote aggregation** (Byzantine consensus voting)
- ✅ **Validator coordination** during epoch transitions

This is about **consensus-level cryptographic certificates**, not transaction Merkle proofs.

---

# PART 1: Committee Certifier (Proof-Related)

## Purpose

Creates **CommitteeAuthorization certificates** that prove validator committees are legitimate. This is consensus-level proof, not transaction-level.

---

## What It Does

### Core Responsibilities

1. **Aggregates votes** from validators on new committees
2. **Generates certificates** when threshold votes reached
3. **Validates proposals** from other validators
4. **Prevents Byzantine attacks** by rejecting invalid committees
5. **Helps slower nodes** catch up by forwarding existing certificates

### Certificate vs Transaction Proof

- **CommitteeAuthorization**: Proves a committee is authorized (voted by previous committee)
- **Transaction Inclusion Proof**: Proves a transaction exists in blockchain (from earlier files)

**Different types of proofs for different purposes.**

---

## Main Structures (Proof-Related)

### CommitteeCertifier<NetworkClient>

```json
{
  "CommitteeCertifier": {
    "certifier_holder": "OnceLock<AsynchronousCertifier>",
    "rx_committee_certificate": "Receiver<Certificate<CommitteeInfo>>",
    "tx_certifier_message": "Sender<CommitteeCertifierMessage>"
  }
}
```

**Fields Explained:**

- `certifier_holder` - The actual certifier that aggregates votes
- `rx_committee_certificate` - Channel receiving completed certificates
- `tx_certifier_message` - Send commands to the certifier

---

## Proof Generation Flow

```
1. EpochManager detects new epoch boundary
   ↓
2. Call certify(new_committee)
   ↓
3. AsynchronousCertifier creates Proposal
   ↓
4. Broadcast proposal to all validators
   ↓
5. Validators vote on proposal
   ↓
6. Collect votes until threshold reached
   ↓
7. Generate Certificate (the proof!)
   ↓
8. Broadcast certificate to network
   ↓
9. All validators enter new epoch
```

---

## Key Functions (Proof-Related)

### `new()`

Creates the certifier with voting infrastructure.

- **Input**: Current epoch state, validator identity, sync mode
- **Output**: Certifier + message channel
- **Setup**: Initializes vote aggregator with threshold type

### `certify()`

Requests certificate generation for new committee.

- **Input**: New committee to certify
- **Output**: None (async - certificate comes via channel)
- **Process**:
  1. Converts committee to `CommitteeInfo`
  2. Sends to `AsynchronousCertifier`
  3. Certifier broadcasts proposal
  4. Aggregates votes from validators
  5. Produces certificate when threshold met

### `await_certificate()`

Waits for completed certificate.

- **Input**: None
- **Output**: `Certificate<CommitteeInfo>` (the proof!)
- **Use**: EpochManager waits for this to transition epochs

### `enter_epoch()`

Updates certifier to new epoch.

- **Input**: New epoch state
- **Output**: None
- **Process**: Certifier updates its committee info for voting

---

## CommitteeCertifierPreprocessor (Byzantine Protection)

### Purpose

**Filters invalid proposals** before vote aggregation to prevent Byzantine attacks.

### `handle_proposal()`

Validates committee proposals from other validators.

**Validation Steps:**

1. **Check epoch** - Is proposal for current/future epoch?
2. **Check digest** - Does committee match expected?
3. **Check storage** - Do we already have certificate?
4. **Detect Byzantine** - Report validators proposing wrong committees

**Returns:**

- `Ok(Some(proposal))` → Valid, continue processing
- `Ok(None)` → Can't validate yet, ignore
- `Err(...)` → Invalid (Byzantine behavior)

### Byzantine Detection Example

```rust
// Honest validators all agree on committee for epoch N
// If validator proposes DIFFERENT committee for epoch N
// → Byzantine behavior detected!
// → Report error and reject
```

---

## Certificate Structure (The Proof)

```json
{
  "Certificate<CommitteeInfo>": {
    "data": "CommitteeInfo",
    "signatures": "Vec<ValidatorSignature>",
    "aggregated_signature": "AggregateSignature"
  }
}
```

**What It Proves:**

- This committee was voted on by ≥2/3 of previous committee
- Cryptographically unforgeable
- Can be verified by anyone with public keys

---

## Constants

```json
{
  "EPOCHS_TO_RETAIN": 1,
  "INPUT_CHANNEL_CAPACITY": 1000,
  "PROPOSAL_RETRY_DELAY_MS": 5000
}
```

---

## Type Aliases

```json
{
  "CommitteeCertifierMessage": "AsynchronousCertifierMessage<CommitteeInfo>"
}
```

---

# PART 2: Validator Event Bus (Coordination)

## Purpose

Coordinates validator components during epoch transitions using a **pub-sub event system**.

---

## What It Does

### Core Responsibilities

1. **Publishes** epoch transition events
2. **Schedules** component restart order
3. **Ensures** components receive updates in correct sequence
4. **Prevents** race conditions during transitions

### Not Proof-Related

This is pure coordination/orchestration. No cryptographic proofs generated here.

---

## Main Structure

### ValidatorEventBus

```json
{
  "ValidatorEventBus": {
    "core": "ValidatorEventBusCore"
  }
}
```

Simple wrapper around generated event bus core.

---

## Notification Types

```json
{
  "Notification": {
    "variants": ["Epoch(EpochNotification)", "Evm(EvmNotification)"]
  },

  "EpochNotification": {
    "variants": ["EndEpoch", "NewEpoch(EpochState)"]
  }
}
```

---

## Subscribers (Components)

```json
{
  "ValidatorEpochManagerNotificationSubscribers": {
    "variants": [
      "ConsensusHelper",
      "MempoolHelper",
      "ConsensusSynchronizer",
      "MempoolSynchronizer",
      "TransactionsInclusionMerkleRootCertification",
      "BlockProposer",
      "ConsensusCore",
      "BatchProcessor",
      "BatchProposer",
      "Pruner",
      "TransactionsIncusionTreePruner"
    ]
  }
}
```

---

## Component Restart Ordering

### NewEpoch (Resume Order)

```
1. ConsensusHelper           → Answer remote requests
2. MempoolHelper              → Answer remote requests
3. ConsensusSynchronizer      → Sync from new validators
4. MempoolSynchronizer        → Sync transactions
5. TransactionsInclusionMerkleRootCertification → Proof generation
6. BlockProposer              → Create blocks with new keys
7. ConsensusCore              → Resume consensus
8. BatchProcessor             → Process batches
9. BatchProposer              → Propose new batches
10. Pruner (optional)         → Clean old data
11. TransactionsIncusionTreePruner (optional) → Clean proof data
```

### EndEpoch (Pause Order)

**Reverse of above** - pauses components in opposite order for safety.

---

## Key Functions

### `schedule()`

Determines component restart order based on event type.

- **Input**: Notification event
- **Output**: Ordered list of subscribers
- **Logic**:
  - `EndEpoch` → Reverse order (pause safely)
  - `NewEpoch` → Forward order (resume correctly)

### `filter()`

Determines which components receive which events.

- **Logic**:
  - `ConsensusCore`, `BatchProposer`, `BatchProcessor` → Get ALL events
  - All others → Only get `Epoch` events (not EVM events)

### `publish_acked()`

Sends event to all subscribers and waits for acknowledgment.

- **Ensures**: All components processed event before proceeding
- **Prevents**: Race conditions during transitions

---

## Why This Ordering Matters

### Example Problem Scenario

```
❌ WRONG ORDER:
1. ConsensusCore resumes first
2. Starts proposing blocks
3. BlockProposer still has OLD epoch keys
4. Signs block with wrong key
5. Block rejected!

✅ CORRECT ORDER:
1. BlockProposer updates first (gets new keys)
2. ConsensusCore resumes
3. Requests block from proposer
4. Block signed with CORRECT key
5. Success!
```

---

## Category Types

```json
{
  "Category": {
    "variants": ["Mandatory(Subscriber)", "Optional(Subscriber)"]
  }
}
```

- **Mandatory** - Must receive event (blocks if not ready)
- **Optional** - Nice to have (doesn't block transitions)

---

## Usage Pattern

```rust
// Create event bus
let mut event_bus = ValidatorEventBus::new();

// Subscribe components
for component in all_components {
    event_bus.subscribe_acked(component)?;
}

// Publish epoch transition
event_bus.publish_acked(
    Notification::Epoch(EpochNotification::NewEpoch(state))
).await?;

// All components now in new epoch!
```

---

## Summary Comparison

### Committee Certifier (Proof System)

- **Type**: Consensus-level cryptographic proof
- **Proves**: Committee is authorized by previous committee
- **Method**: Vote aggregation + Byzantine fault tolerance
- **Output**: `Certificate<CommitteeInfo>`

### Event Bus (Coordination)

- **Type**: Pub-sub orchestration
- **Purpose**: Coordinate component updates
- **Method**: Ordered message delivery
- **Output**: Synchronized state transitions

---

## Relationship to Transaction Proofs

**These systems are complementary:**

```
Transaction Inclusion Proofs (from earlier files)
    ↓
Prove transactions exist in blocks
    ↓
Blocks are proposed by validators
    ↓
Validators are authorized by certificates (THIS FILE)
    ↓
Certificate creation coordinated by event bus (THIS FILE)
```

**Complete trust chain:**

1. Event bus coordinates epoch transition
2. Committee certifier proves new validators legitimate
3. New validators create blocks
4. Transactions in blocks get inclusion proofs

---

## Key Design Decisions

### 1. Ordered Acknowledgment

- Events delivered in sequence
- Components acknowledge processing
- Prevents race conditions

### 2. Byzantine Protection

- Preprocessor validates proposals
- Rejects conflicting committees
- Reports malicious validators

### 3. Helping Slow Nodes

- Forwards existing certificates
- Allows catchup without re-voting
- Ensures network progress

### 4. Separation of Concerns

- Event bus = coordination
- Certifier = proof generation
- Preprocessor = validation
- Each has single responsibility
