# EIP-8141 "Frame Transaction" Review

**Reviewer:** Ben Adams
**Date:** 2026-04-12
**Status:** Draft Review

---

## 1. What I Like

### The Right Abstraction at the Right Layer

This EIP correctly identifies that account abstraction belongs at the transaction level, not bolted on as a contract-level shim (ERC-4337). By introducing frame transactions as a first-class EIP-2718 type, the protocol can reason about validation, execution, and payment as distinct phases with proper isolation guarantees. This is the kind of clean protocol-level design that avoids the accumulated overhead of userspace workarounds.

### Separation of VERIFY and SENDER Modes

The mode system is well-designed. Separating validation (VERIFY running as STATICCALL semantics) from execution (SENDER running with caller authority) creates a clean security boundary. The fact that VERIFY frames cannot modify state means nodes can safely re-execute validation without worrying about side effects. This is critical for mempool safety and is something ERC-4337 always struggled with conceptually.

### Signature Hash Elision

Eliding VERIFY frame data from the signature hash is elegant. It solves the chicken-and-egg problem (signature cannot be part of the signed data) and simultaneously enables future signature aggregation. This forward-thinking design choice costs nothing today but opens significant data-efficiency gains tomorrow when aggregation schemes mature.

### ORIGIN Redefinition per Frame

Returning the frame caller from ORIGIN rather than the transaction origin is the correct choice. The traditional ORIGIN has been an anti-pattern magnet for years. Scoping it per-frame prevents confused-deputy attacks where contracts assume ORIGIN is a trusted EOA.

### Warm/Cold Journal Sharing Across Frames

Sharing the warm/cold access journal across frames is a pragmatic performance optimization. Without this, a multi-frame transaction that touches the same storage slots or addresses in VERIFY and then SENDER would pay cold-access costs twice. Given that the validation frame will almost certainly touch the sender's storage and the execution frame will too, this saves 2100 gas (cold storage read) per overlapping slot. For a typical smart account validation that reads 2-3 storage slots, this is 4200-6300 gas saved, which is meaningful.

### Atomic Batching

The atomic batch flag on SENDER frames is a well-scoped feature. The approve-then-swap pattern is the most common source of dangling approvals and stuck intermediate states in DeFi today. Making this atomic at the transaction level rather than requiring multicall contracts removes a class of bugs entirely. The flag-based design is minimal and composes well with the existing frame structure.

### Default Code for EOAs

Providing default code behavior for EOAs is critical for adoption. Users should not be forced to deploy a smart account before they can use frame transactions. The inclusion of both secp256k1 and P256 signature types in the default code is forward-looking (P256 for passkey/secure-enclave wallets). The RLP-encoded multicall in SENDER mode default code gives EOAs basic batching for free.

### Data Efficiency

134 bytes for a basic smart account ETH transfer is competitive. The analysis in the EIP is honest about the overhead versus EIP-1559 transactions and correctly notes it is dramatically more efficient than ERC-4337's ABI-encoded UserOperation.

---

## 2. What I Don't Like / Concerns

### Intrinsic Cost of 15,000 Gas Feels Arbitrary

The FRAME_TX_INTRINSIC_COST of 15,000 gas is a 71% increase over the standard 21,000 for legacy transactions when you consider the validation frame gas is on top of this. For simple self-paying EOA transactions (the most common case), a frame transaction costs at minimum 15,000 + calldata_cost + validation_gas + execution_gas. If the goal is to eventually replace legacy transaction types, this premium discourages migration. The intrinsic cost should be justified with concrete measurements of the additional node processing overhead, not set as a round number.

More importantly: what is the intrinsic cost actually paying for? The per-frame gas limits already cover execution. The calldata cost already covers data availability. The intrinsic cost should reflect the fixed overhead of transaction processing (signature verification, state lookups, receipt creation). For frame transactions, some of this work (signature verification) is done inside frames and already paid for by frame gas limits. There is a risk of double-charging.

### Per-Frame Gas Isolation Is Wasteful

Each frame has its own gas_limit and unused gas from one frame cannot flow to the next. This means transaction authors must over-provision each frame to handle worst-case execution, and the sum of these over-provisions is dead gas that cannot be reclaimed by later frames. In practice, a 3-frame transaction (verify, approve+swap atomic batch) requires the user to estimate gas for each frame independently. Over-estimation across 3 frames compounds.

Consider: a VERIFY frame needs ~30,000 gas but the user provisions 50,000 to be safe. A SENDER frame for ERC-20 approve needs ~50,000 but provisions 70,000. A SENDER frame for a DEX swap needs ~200,000 but provisions 250,000. The user pays for 370,000 gas but uses 280,000. The 90,000 gas of over-provision is refunded, but the user still had to have the ETH to cover it, and the block gas accounting reserved that capacity.

An alternative would be a shared gas pool with per-frame caps, or allowing unused gas to cascade forward. The current design prioritizes simplicity and frame isolation at the cost of capital efficiency.

### MAX_FRAMES = 1000 Is Excessive

A limit of 1000 frames per transaction is far too generous. The atomic batch example uses 3 frames. The most complex sponsored deployment example uses 5. Even an elaborate multi-operation batch with verification and post-processing would struggle to meaningfully use more than 20-30 frames. A limit of 1000 creates unnecessary surface area for abuse:

- The receipt structure contains a frame_receipt per frame. 1000 frame receipts bloat the receipt trie.
- The signature hash computation iterates all frames to elide VERIFY data. 1000 frames means 1000 iterations.
- Static validation must check mode constraints, atomic batch flag consistency, and target length for each frame. 1000 frames means 1000 checks before any execution begins.
- The TXPARAM opcode exposes frame index parameters, meaning contracts can iterate over frame metadata. 1000 frames means potentially 1000 TXPARAM calls at 2 gas each (trivially cheap to enumerate).

A limit of 64 or even 32 would be more than sufficient for any realistic use case and would allow implementations to use fixed-size stack-allocated arrays for frame metadata.

### compute_sig_hash Is Computed Per TXPARAM(0x08, 0) Call

The specification says TXPARAM with param 0x08 returns `compute_sig_hash(tx)`. This involves iterating all frames, eliding VERIFY data, RLP-encoding the modified transaction, and computing a keccak256 hash. If this is computed on every TXPARAM(0x08, 0) call rather than cached, it is an O(n) operation where n is the total size of non-VERIFY frame data. Implementations must cache this value, but the spec does not mention caching. It should either explicitly state that the signature hash is computed once and cached, or acknowledge the gas cost implications if it is recomputed.

At 2 gas per TXPARAM call, a contract could call TXPARAM(0x08, 0) in a loop and force the node to repeatedly compute keccak over potentially large transaction data for almost no gas cost. This is a potential DoS vector if implementations do not cache.

### Transient Storage Reset Between Frames Is Surprising

Discarding TSTORE/TLOAD transient storage between frames breaks the mental model that frames within a single transaction share transient context. Transient storage (EIP-1153) was designed to be transaction-scoped. Resetting it between frames means that a VERIFY frame cannot pass hints to a SENDER frame via transient storage, and two SENDER frames in an atomic batch cannot use transient storage to coordinate. This forces all cross-frame communication through the TXPARAM/FRAMEDATALOAD opcodes or through persistent storage (which costs 20,000+ gas for a write).

I understand the rationale is probably isolation and preventing validation frames from leaking state to execution frames, but VERIFY frames already run as STATICCALL (no writes). The reset penalizes legitimate SENDER-to-SENDER communication within atomic batches.

### Nonce Increment Timing Is Fragile

The nonce is incremented inside APPROVE(0x2) or APPROVE(0x3), which happens during a VERIFY frame. If the VERIFY frame runs but the transaction ultimately fails (e.g., payer_approved never becomes true because a later payment frame reverts), is the nonce incremented? The spec says VERIFY frame state changes are discarded on revert, but APPROVE modifies transaction-scoped state (sender_approved, payer_approved) and also increments the nonce. The nonce increment is a persistent state change happening inside a STATICCALL-like context. This interaction needs to be clearer.

If the nonce increments only when the VERIFY frame succeeds and the transaction completes, that is fine. But the spec says the nonce increments inside APPROVE, which runs during VERIFY. Does VERIFY frame revert roll back the nonce? What about the gas cost collection? These are persistent state mutations triggered from a mode described as "behaves the same as STATICCALL."

### APPROVE Scope Bit Encoding Is Overloaded

Bits 9-10 of the mode field encode the approval scope, and the scope operand on the APPROVE opcode must match. But the frame mode also determines VERIFY/SENDER/DEFAULT via the lower 8 bits. And bit 11 is the atomic batch flag. This bit-packing is dense and error-prone for implementers. A single off-by-one in the bit shift and a node accepts invalid transactions or rejects valid ones. The spec should include explicit test vectors for all valid combinations of mode bits.

---

## 3. Issues

### Issue 1: TXPARAM Parameter Numbering Gap

The TXPARAM table jumps from 0x09 to 0x10, skipping 0x0A through 0x0F. This appears intentional (hex alignment for frame-specific params starting at 0x10), but 0x10 ("currently executing frame index") requires `in2` to be 0, while 0x11-0x17 use frame indices. The gap creates wasted opcode parameter space and the asymmetry between 0x10 (current frame, in2=0) and 0x11+ (arbitrary frame, in2=frame_index) is inconsistent. Why not return the current frame index as a simple stack push and reserve TXPARAM for parameterized queries?

### Issue 2: Exceptional Halts Are Overly Aggressive

The specification uses exceptional halt (consuming all remaining gas) for several conditions that could reasonably be soft errors:

- Invalid TXPARAM `param` values
- Out-of-bounds frame index in TXPARAM, FRAMEDATALOAD, FRAMEDATACOPY
- Accessing status of current/future frames

Exceptional halts in validation frames are particularly dangerous because they consume MAX_VERIFY_GAS and provide no error information. A revert with a reason would be more debuggable and would still fail the transaction. This is the EVM's historical mistake repeated. Consider returning zero or a sentinel value for out-of-bounds access (matching CALLDATALOAD behavior, which zero-pads) rather than halting.

### Issue 3: Default Code P256 Address Derivation Is Permanently Binding

The default code derives the P256 account address as `keccak(qx|qy)[12:]`. This means the address is permanently bound to a specific P256 key pair. If the key is compromised, the user cannot rotate to a new key without migrating to a new address. For secp256k1, this matches existing EOA behavior (address derived from public key). But P256 is being introduced specifically as a new signature type. It would be better to allow P256 keys to be associated with addresses via some registration mechanism rather than hard-coding the derivation in the default code. Otherwise, P256-based EOAs inherit the same key-rotation limitations as secp256k1 EOAs.

### Issue 4: Atomic Batch Revert Semantics and Gas Accounting

When an atomic batch reverts (frame N in the batch fails, frames 0..N-1 are rolled back, frames N+1..end are skipped), what happens to the gas consumed by frames 0..N? The spec says "restore the state to the snapshot," but gas is not state. If the gas is consumed (not refunded), the user pays for work that was entirely discarded. If the gas is refunded, the block builder did real work (executing those frames) for free, creating a DoS vector.

The spec should explicitly state: gas consumed by reverted atomic batch frames is charged and not refunded. This matches the behavior of reverted internal calls in the current EVM, but it should be stated, not implied.

### Issue 5: Sender Field in Transaction Payload

The frame transaction includes `sender` as an explicit field in the RLP payload. This is unusual. In all other transaction types, the sender is recovered from the signature. Including sender explicitly means:

1. The sender is trusted input until the VERIFY frame confirms it. A node must treat the sender as unverified until APPROVE(0x1) or APPROVE(0x3) succeeds.
2. The nonce check (`tx.nonce == state[tx.sender].nonce`) happens before any frame executes, using the unverified sender. A malicious actor could submit transactions with arbitrary sender addresses and force nodes to perform nonce lookups against those addresses before discovering the transaction is invalid.
3. This creates a mempool DoS vector: submit 1000 transactions with random sender addresses. Each requires a state lookup for the nonce check before the (expensive) validation frame reveals the sender is fake.

Mitigation: Nodes should rate-limit frame transaction acceptance and potentially require a bond or proof-of-work for initial submission. The spec should acknowledge this vector more explicitly.

### Issue 6: APPROVE Has Return Data Semantics but Unclear Behavior

APPROVE takes offset and length from the stack (like RETURN) but the spec does not describe what happens with this return data. Is it available to subsequent frames via FRAMEDATALOAD? Is it discarded? Can it be used to pass data from a VERIFY frame to later frames? The offset/length parameters suggest return data is intended, but the spec is silent on where it goes.

### Issue 7: Missing gas_limit Per-Blob Specification

The transaction references `max_fee_per_blob_gas` and `blob_versioned_hashes` but does not specify how blob gas interacts with frame gas limits. Blob gas is a separate dimension (EIP-4844). Does each frame's gas_limit apply only to execution gas? Can a single frame trigger blob-related costs? This should be clarified, even if the answer is "blob gas is transaction-level and not allocated per-frame."

---

## 4. Usage Scenarios and Possibilities I Like

### Post-Quantum Migration Path

This is the headline feature and it delivers. Any account can define its own signature verification in EVM code. When NIST finalizes post-quantum standards and precompiles are added for ML-DSA or SPHINCS+, accounts can adopt them without a protocol upgrade to the transaction format. The frame transaction is the last transaction type Ethereum needs to add for authentication purposes.

### Trustless ERC-20 Gas Payment

Example 3 in the EIP (sponsored transaction with ERC-20 payment) is the killer use case for mainstream adoption. A user holding only USDC can transact on Ethereum without ever acquiring ETH. The sponsor verifies the user has tokens, pays ETH for gas, receives tokens as compensation, and can even swap them back to ETH in a post-op frame. This is end-to-end trustless and does not require the user to trust a centralized relayer.

### Atomic Approve-and-Swap

This eliminates the most common footgun in DeFi. No more dangling ERC-20 approvals from failed swaps. No more front-running between the approve and swap transactions. A single frame transaction atomically approves and swaps, and if the swap fails, the approval is rolled back.

### First-Transaction Account Deployment

Deploying a smart account in the first frame of its first transaction means users do not need a separate funding/deployment step. Combined with counterfactual addresses (deterministic deployers), a user can receive funds at their future smart account address, and the first transaction they send deploys the account and executes the intended operation in one shot.

### Passkey and Secure Enclave Wallets

P256 support in the default code enables browser-native passkey authentication and mobile secure enclave signing without deploying a smart account. This dramatically lowers the barrier to non-ECDSA authentication for the ~100M EOA addresses that exist today.

### Multi-Operation Batching

Even without the atomic flag, multi-frame SENDER execution lets wallets batch multiple operations into a single transaction. Claim rewards, swap tokens, provide liquidity -- all in one transaction. This reduces the number of transactions hitting the mempool and saves users 21,000 gas per operation that would otherwise be a separate transaction.

---

## 5. Additional Possibilities Enabled That Aren't Being Considered

### Session Keys and Delegated Execution

The VERIFY/SENDER separation naturally supports session key patterns. A smart account's VERIFY frame could validate a signature from a delegated key (not the account owner) for a limited scope of operations. Combined with the scope bits on APPROVE, this enables:

- Gaming sessions where a temporary key can execute game moves but not transfer assets
- DApp-specific keys that can interact with one contract but not others
- Time-limited delegation using on-chain state checked during VERIFY

This is not explicitly discussed in the EIP but falls out naturally from the architecture.

### Cross-Frame Data Passing for MEV Protection

VERIFY frames with scope-restricted APPROVE could be used to implement MEV-protection schemes. A VERIFY frame could commit to a specific execution ordering or price bound, and the SENDER frame would enforce it. The frame structure makes this commitment verifiable by block builders without them needing to understand the smart account's internal logic.

### Programmable Fee Markets

The separation of sender approval (0x1) and payer approval (0x2) enables fee market experimentation. A payer could dynamically price gas based on the operations being performed, the current gas price, or any on-chain condition. This opens the door to:

- Subscription-based gas payment (paymaster checks subscription status)
- Gas futures (lock in a gas price for future transactions)
- Cross-chain gas payment (verify a proof from another chain in the VERIFY frame)

### Signature Aggregation

The EIP mentions this in passing (rationale for eliding VERIFY data from the signature hash), but does not fully explore it. With VERIFY frame data being opaque to the signature hash, a block builder could aggregate BLS signatures from multiple frame transactions into a single verification. This requires additional protocol support but the frame transaction format is designed to be compatible. This could reduce per-transaction verification cost to near-zero for batched L2 submissions.

### Intent-Based Execution

The frame structure is a natural fit for intent protocols. A VERIFY frame validates the user's intent (signed statement of desired outcome). The SENDER frames execute the operations to fulfill that intent. A post-op DEFAULT frame verifies the outcome matches the intent. If it does not, the atomic batch reverts everything. This is strictly more powerful than existing intent protocols because the verification happens at the protocol level.

### Programmable Replay Protection

The nonce field combined with custom VERIFY logic enables programmable replay protection beyond sequential nonces. A smart account could implement:

- 2D nonces (channel + sequence) for parallel transaction submission
- Bitmap-based nonces for out-of-order execution
- Time-based expiry in addition to nonce ordering

The TXPARAM notes hint at multidimensional nonces as a future extension, but smart accounts can implement this in their VERIFY logic today.

---

## 6. Suggestions for Changes to Increase Optionality

### 6.1. Add a Frame Return Data Channel

Allow APPROVE's offset/length parameters to populate a per-frame return data buffer accessible to subsequent frames via a new opcode (e.g., FRAMERETURNDATA). This enables:

- VERIFY frames passing validated parameters to SENDER frames (e.g., "this signature authorizes spending up to X tokens")
- Post-op frames reading execution results from SENDER frames
- Cross-frame coordination without persistent storage writes

The VERIFY frame data elision already prevents information leakage from signatures. Return data flowing forward (never backward) maintains the security model while enabling composition.

### 6.2. Reduce MAX_FRAMES to 64

This is generous enough for any realistic use case (deploy + verify + 60 batched operations + post-op) while being small enough that implementations can use fixed-size arrays and avoid heap allocations for frame metadata. It also limits the receipt trie bloat from frame_receipts. If 64 proves insufficient, it can be increased in a future fork. Reducing it later would be a breaking change.

### 6.3. Allow Optional Gas Cascading

Add an optional flag (another mode bit, or a transaction-level flag) that enables unused gas from completed frames to be added to the gas pool of subsequent frames. This would be opt-in per transaction to maintain backward compatibility with the isolated gas model. When enabled, the gas accounting becomes:

```
available_gas[i] = frame[i].gas_limit + sum(unused_gas[j] for j < i if cascade enabled)
```

This dramatically improves capital efficiency for multi-frame transactions where gas estimation is uncertain.

### 6.4. Define a Frame Return Status Encoding

The receipt includes `status` per frame, but only as 0 (failure) or 1 (success). Consider expanding this to include:

- 2: skipped (atomic batch, subsequent frame after revert)
- 3: skipped (sender not approved, frame could not execute)

This provides better observability without changing execution semantics.

### 6.5. Add FRAMERETURNDATACOPY / FRAMERETURNDATASIZE

Complement FRAMEDATALOAD/FRAMEDATACOPY with opcodes to access the return data of previously executed frames. This is different from frame input data (FRAMEDATALOAD reads the frame's calldata). Return data from executed frames is often needed for composition:

- A SENDER frame calls a DEX and returns the amount received
- The next SENDER frame needs to know that amount to provide liquidity

Currently, the only way to pass this data is through persistent storage (expensive) or by hardcoding it (fragile). Frame return data opcodes would make multi-frame composition practical.

### 6.6. Consider a VERIFY Frame Gas Sublimit

Instead of relying solely on the mempool-enforced MAX_VERIFY_GAS of 100,000, consider making this a consensus-enforced limit on VERIFY frames. This would:

- Prevent validation frames from consuming excessive gas even in private mempools
- Give block builders a hard upper bound on the cost of including a frame transaction
- Simplify mempool validation (no need to simulate and measure, just check the gas_limit field)

If the VERIFY frame's gas_limit exceeds the consensus sublimit, the transaction is statically invalid.

### 6.7. Explicit Caching Requirement for compute_sig_hash

Add a note that implementations MUST cache the result of compute_sig_hash for the duration of the transaction's processing. This prevents the DoS vector of repeated TXPARAM(0x08, 0) calls forcing recomputation.

### 6.8. Consider Preserving Transient Storage Within Atomic Batches

If transient storage must be reset between frames generally (for isolation), consider preserving it within atomic batches. Frames in an atomic batch are semantically a single unit of work. Allowing them to share transient storage via TSTORE/TLOAD would be cheaper than persistent storage for cross-frame coordination within a batch, and the isolation property still holds between batches.

### 6.9. Add Frame-Level Access Lists (Future Consideration)

The EIP notes that access lists are omitted because block-level access lists will cover this use case. However, per-frame access list hints would allow:

- Parallel frame execution by non-overlapping state access (future optimization)
- Better gas estimation by pre-declaring touched slots
- Reduced warm/cold gas variance across execution paths

This does not need to be in the initial spec, but the frame structure should be designed to accommodate it. Reserving a field position in the frame tuple (e.g., `[mode, target, gas_limit, data, access_list?]`) would be sufficient.

### 6.10. Post-Quantum Default Code Should Include ML-DSA

The default code supports secp256k1 (signature_type 0x0) and P256 (signature_type 0x1). Given that the primary motivation of the EIP is post-quantum security, the default code should also support ML-DSA-44 (FIPS 204) as signature_type 0x2 once a precompile exists. Reserving the signature type now and specifying the behavior (even if the precompile is not yet deployed) ensures the default code design is forward-compatible. This is especially important because the default code is "virtual" -- it is implemented by clients, not deployed as a contract. Adding a new signature type to the default code requires all clients to update, which has the same coordination cost as a hard fork.

---

## Summary Assessment

EIP-8141 is a well-designed, ambitious proposal that correctly places account abstraction at the protocol level. The frame model is the right abstraction -- it cleanly separates validation, authorization, and execution while remaining flexible enough to support post-quantum cryptography, gas sponsorship, atomic batching, and future extensions.

The primary concerns are:

1. **Gas efficiency** -- Per-frame gas isolation wastes capital; the intrinsic cost needs justification.
2. **MAX_FRAMES** -- 1000 is gratuitously large and creates unnecessary implementation complexity.
3. **TXPARAM(0x08) DoS** -- compute_sig_hash caching must be mandated.
4. **Transient storage reset** -- Overly aggressive isolation that penalizes atomic batch composition.
5. **Nonce/APPROVE state mutation semantics** -- The interaction between STATICCALL-like VERIFY mode and the persistent state changes in APPROVE needs clearer specification.
6. **Sender field DoS** -- Explicit sender in the payload enables cheap nonce-lookup spam.

None of these are fatal. The architecture is sound. The suggestions above are aimed at increasing optionality and reducing the likelihood of needing breaking changes in future forks. Ship it, but tighten the constants and clarify the edge cases.
