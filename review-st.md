# EIP-8141 "Frame Transaction" -- Technical Review

**Reviewer:** Stephen Toub (style)
**Date:** April 12, 2026
**Status of EIP:** Draft
**Authors:** Vitalik Buterin, lightclient, Felix Lange, Yoav Weiss, Alex Forshtat, Dror Tirosh, Shahaf Nacson, Derek Chiang

---

## 1. What I Like About It

### The Frame Abstraction Is Genuinely Good API Design

The central insight of this EIP -- decomposing a transaction into an ordered sequence of typed frames with distinct modes (VERIFY, SENDER, DEFAULT) -- is the kind of composable abstraction that ages well. It reminds me of how `System.IO.Pipelines` decomposed I/O into a producer/consumer pipeline with backpressure: you take something monolithic (a transaction), identify the orthogonal concerns (authentication, authorization, payment, execution, post-processing), and give each one a first-class representation.

The mode system is a good application of the "make illegal states unrepresentable" principle. VERIFY frames run as STATICCALL, which means you cannot accidentally mutate state during validation. SENDER frames require prior approval, which means you cannot impersonate the sender without an explicit authorization gate. This is the kind of structural invariant that prevents entire classes of bugs rather than relying on developers to remember a convention.

### Signature-Agnostic Validation

Moving cryptographic validation into EVM-executed code, with the canonical signature hash provided via `TXPARAM(0x08, 0)`, is the right long-term architecture. Hardcoding ECDSA into the protocol was always a tech-debt time bomb given the post-quantum transition ahead. The default code path for EOAs (supporting both secp256k1 and P256 via a type byte) provides a clean migration story.

The decision to elide VERIFY frame data from the signature hash is subtle and correct. The signature cannot sign over itself, and leaving the VERIFY data malleable enables future aggregation schemes and sponsor-appended data. This is the kind of forward-thinking design choice that pays dividends years later.

### Gas Isolation Between Frames

Each frame having its own `gas_limit` with no cross-frame gas borrowing is a strong design choice. It mirrors the principle that resource budgets should be explicit and local rather than shared and implicit. In .NET terms, this is like giving each middleware component its own `CancellationToken` timeout rather than sharing a single ambient deadline -- it makes reasoning about resource consumption compositional.

### Atomic Batching via a Flag Bit

The atomic batch design -- consecutive SENDER frames with bit 11 set, terminated by a SENDER frame without it -- is elegantly minimal. It avoids introducing a new frame mode or nesting construct. The snapshot-and-rollback semantics are clean and match what users expect from database transactions. The approve-then-swap example (Example 2) is a compelling demonstration of why this matters: without atomicity, a reverted swap leaves a dangling ERC-20 approval, which is a real security footgun that has caused losses in the wild.

### Warm/Cold State Sharing Across Frames

Sharing the warm/cold access journal across frames is a good amortization decision. If frame 0 touches a storage slot, frame 1 should not pay the cold access cost again. This is the same principle as connection pooling -- the expensive setup cost should be paid once per logical unit of work, not once per sub-operation.

### Transient Storage Isolation

Discarding TSTORE/TLOAD transient storage between frames is the correct boundary. Transient storage was designed for intra-transaction communication within a single execution context. Leaking it across frames would create implicit coupling between frames that are supposed to be independently authored (e.g., a user's execution frame and a sponsor's post-op frame).

---

## 2. What I Do Not Like About It

### The Mode Bit-Packing Is Too Clever

The `mode` field encodes the execution mode in bits 0-7, the approval scope in bits 9-10, and the atomic batch flag in bit 11. But bit 8 is... skipped? The spec says "lower bits (<= 8)" for the mode and "upper bits (> 8)" for flags. This means bit 8 is part of the mode field (which uses only values 0-2 out of a 9-bit space), while bits 9-10 are the approval scope.

This is the kind of bit-packing that looks compact in a spec table but creates implementation bugs. Every client team will write their own bit-manipulation code, and the off-by-one between "bits <= 8" and "bits > 8" will trip someone up. The TXPARAM opcode then re-exposes these as separate fields (0x13 for mode, 0x16 for scope, 0x17 for atomic_batch), acknowledging that the packed representation is not how consumers actually want to see the data.

I would prefer three separate fields in the frame tuple: `[mode, scope, flags, target, gas_limit, data]`. The RLP overhead of two extra small integers is negligible compared to the clarity gained. Alternatively, if compactness is truly essential, document the bit layout with a diagram and use consistent 0-indexed bit numbering throughout.

### TXPARAM Parameter Numbering Has a Gap

The `param` values jump from `0x09` to `0x10`. This is not hexadecimal confusion (0x0A through 0x0F are simply absent). It appears intentional -- grouping transaction-level params in 0x00-0x09 and frame-level params in 0x10-0x17 -- but it is not documented as such. An implementer reading the spec will wonder whether 0x0A-0x0F are reserved, accidentally omitted, or a typo. The spec should explicitly state the grouping rationale and mark the gap as reserved.

### The `in2` Parameter Name Is Unhelpful

In the TXPARAM opcode, the second stack input is called `in2`. This name communicates nothing about its semantics. For some param values it must be zero; for others it is a frame index. A name like `frame_index_or_zero` or even just `index` would be more self-documenting. In API design, parameter names are documentation -- `Task.WhenAll(params Task[] tasks)` tells you what it expects; `Task.WhenAll(params Task[] in2)` does not.

### Gas Isolation Creates Estimation Complexity

While I praised gas isolation above, it has a significant usability cost. The transaction sender must estimate gas for each frame independently and cannot over-allocate to one frame to compensate for under-allocation in another. This is like having separate thread pool quotas for each stage of a pipeline -- correct in theory, but operationally painful.

For simple transactions this is manageable, but for sponsored transactions with 4-5 frames, the user (or their wallet) must make 4-5 independent gas estimates that must all be correct. If any single frame runs out of gas, the entire sequence may fail (or in the atomic batch case, roll back). The spec does not discuss gas estimation tooling or provide guidance for wallets on how to handle this.

Unused gas from earlier frames being "wasted" (not available to later frames) also creates a perverse incentive to estimate as tightly as possible, which makes transactions more fragile. Consider whether a small inter-frame gas donation mechanism (perhaps opt-in via a flag bit) would improve robustness without undermining the isolation benefits.

### The APPROVE Opcode Does Too Many Things

APPROVE is simultaneously:
1. A control flow instruction (exits the current context like RETURN)
2. A state mutation (sets sender_approved / payer_approved)
3. A nonce increment (for scope 0x2 and 0x3)
4. A balance deduction (collects gas cost for scope 0x2 and 0x3)
5. A return data mechanism (offset/length on the stack)

This violates the single-responsibility principle. In .NET terms, this is like having a `Dispose()` method that also commits a transaction, increments a sequence number, and writes to a log. Each of those is a reasonable operation, but combining them into one call makes the behavior hard to reason about, hard to test in isolation, and fragile to extend.

The nonce increment happening inside APPROVE is particularly surprising. The nonce is a transaction-level concern, but it is incremented by an opcode that runs inside a frame's EVM execution. If a future EIP needs to change nonce semantics (e.g., multidimensional nonces, as hinted in the TXPARAM notes), it will need to modify the APPROVE opcode's behavior, coupling two concerns that should be independent.

### ORIGIN Semantics Change Is a Compatibility Risk

The spec acknowledges that `ORIGIN` now returns the frame's caller rather than the traditional transaction origin, and waves this away by citing EIP-7702 precedent. But EIP-7702 modified ORIGIN in a much narrower context (delegated code execution). Here, ORIGIN's semantics change fundamentally for an entire new transaction type. Contracts that use `tx.origin == msg.sender` as a (discouraged but widespread) EOA check will behave differently under frame transactions. The "discouraged pattern" caveat does not help the contracts already deployed on mainnet that use it.

More importantly, banning ORIGIN in the validation prefix means that contracts which currently use ORIGIN for any purpose (even logging) cannot participate in validation. This is fine for new code but creates friction for existing contracts that might otherwise be reusable as validation helpers.

---

## 3. Issues

### Issue 1: Signature Hash Mutation of the Transaction Object

The `compute_sig_hash` function in the spec modifies the transaction object in place:

```python
def compute_sig_hash(tx: FrameTx) -> Hash:
    for i, frame in enumerate(tx.frames):
        if (frame.mode & 0xFF) == VERIFY:
            tx.frames[i].data = Bytes()
    return keccak(rlp(tx))
```

This is a mutation side effect. If a client caches the transaction object and calls `compute_sig_hash`, it silently destroys the VERIFY frame data. The spec should either (a) explicitly note that this must operate on a copy, or (b) rewrite it as a pure function that constructs a new frame list. This is exactly the kind of bug that shows up in one client implementation and takes weeks to diagnose.

### Issue 2: Atomic Batch Boundary Ambiguity

The atomic batch constraint says:

```python
if (frame.mode >> 10) & 1 == 1:
```

But bit 11 is the atomic batch flag, and `(frame.mode >> 10) & 1` extracts bit 10, not bit 11. To extract bit 11 you need `(frame.mode >> 10) & 1` only if you are 0-indexed from bit 0 and the shift is correct -- let me re-examine. If `mode` has bit 11 set (value 0x800 = 2048), then `mode >> 10` gives `0b10`, and `& 1` gives `0`. You would need `(frame.mode >> 10) & 2` or `(frame.mode >> 11) & 1`. **This appears to be a bug in the constraint pseudocode.** The text says "bit 11" but the code checks bit 10.

Update: Actually, this depends on whether bits are numbered starting from 0 or 1. The mode flag table says "9-10" for approval scope and "11" for atomic batch. If we number from bit 0 (the LSB), then bit 9 is `(mode >> 9) & 1` and bit 11 is `(mode >> 11) & 1`. The APPROVE spec says `(frame.mode >> 8) & 3` for the approval scope, which extracts bits 8-9 (0-indexed). But the table says bits "9-10" for approval scope. This is internally inconsistent -- either the table uses 1-indexed bit numbering and the code uses 0-indexed, or there is a genuine off-by-one error.

**This inconsistency between bit numbering in the prose and bit manipulation in the pseudocode is a critical specification bug.** Every client implementation will need to make a choice, and if they choose differently, consensus will break.

### Issue 3: Nonce Check Timing vs. APPROVE Nonce Increment

The spec says:
- Stateful validation: "Ensure `tx.nonce == state[tx.sender].nonce`"
- APPROVE scope 0x2: "Increment the sender's nonce"

The nonce is checked at the start of transaction processing against the state, but incremented only when APPROVE is called during a VERIFY frame. If the VERIFY frame executes significant code before calling APPROVE, there is a window where the nonce has been validated but not incremented. During this window, if the same sender has another transaction in the mempool with the same nonce, there is no on-chain protection against replay.

This is mitigated by the mempool rule limiting one pending transaction per sender, but the spec should explicitly address what happens if two frame transactions from the same sender with the same nonce are included in the same block by a block builder. The nonce check passes for both at the start of execution, but only the first one to call APPROVE will successfully increment it -- what happens to the second? Does it become invalid? Does the block become invalid?

### Issue 4: Payment Approval Requires Sender Approval First

APPROVE scope 0x2 requires `sender_approved == true`. This means a third-party payer (sponsor) cannot approve payment until the sender has approved execution. But in the sponsored transaction flow (Example 3), the sender's VERIFY frame (frame 0) calls `APPROVE(0x1)` and the sponsor's VERIFY frame (frame 1) calls `APPROVE(0x2)`.

The ordering constraint is fine for the happy path, but what happens if the sender wants to delegate both execution approval and payment selection to a smart contract? The rigid ordering (sender first, then payer) may be unnecessarily restrictive. Consider whether `sender_approved` should be required for `APPROVE(0x2)` or whether the payment approval should be independent, with the constraint enforced structurally (both must be true before SENDER frames execute).

### Issue 5: Default Code SENDER Mode Reverts on Any Sub-call Failure

The default code for SENDER mode with EOAs executes RLP-decoded calls and reverts the entire frame if any single call reverts:

```python
for call_target, call_value, call_data in calls:
    result = evm_call(caller=tx.sender, to=call_target, value=call_value, data=call_data)
    if result.reverted:
        revert()
```

This is all-or-nothing within a single frame. If a user batches 10 calls in one SENDER frame's default code, a revert on call 9 discards calls 1-8. But the user has no way to express "these 3 calls are atomic, those 7 are independent" within a single frame. They must split into separate frames, each with its own gas allocation.

This is a usability problem because the gas estimation becomes harder (more frames = more independent estimates) and the RLP encoding becomes less efficient (repeating the frame header overhead for each group). The default code should consider a per-call "must succeed" flag, or the spec should provide guidance on the trade-offs of few-large-frames versus many-small-frames.

### Issue 6: TXPARAM(0x09, 0) Says "can be zero" But Constraints Say len(frames) > 0

The TXPARAM table says `len(frames)` "can be zero" for param 0x09. But the static constraints say `len(tx.frames) > 0`. These are contradictory. If a transaction must have at least one frame, then `TXPARAM(0x09, 0)` will always return >= 1, and the "can be zero" note is misleading.

### Issue 7: No Explicit Limit on Frame Data Size

`MAX_FRAMES` is defined as 10^3 (1000), but there is no explicit limit on the size of each frame's data field. A single frame could carry megabytes of data, with the only constraint being the gas cost of the calldata (4/16 gas per byte) and the block gas limit. This is consistent with existing transaction calldata limits, but the frame structure adds overhead for deserialization and validation that could amplify the impact of large payloads. The spec should either explicitly state that frame data size is bounded only by the block gas limit, or introduce a per-frame data size cap.

### Issue 8: FRAMEDATACOPY Gas Cost Specification

The spec says "The gas cost matches CALLDATACOPY" but CALLDATACOPY has both a static cost (3 gas) and a dynamic cost based on memory expansion and data length. The spec partially describes this ("a fixed cost of 3 and a variable cost that accounts for the memory expansion and copying") but does not specify the per-word copy cost. CALLDATACOPY charges 3 gas per 32-byte word of data copied (the `COPY` gas schedule). The spec should reference this explicitly to avoid ambiguity.

### Issue 9: Interaction with EIP-7702 Delegations

The validation trace rules ban `CALL*` or `EXTCODE*` to addresses that use EIP-7702 delegations, "except for tx.sender default-code behavior." But the spec does not clearly define what happens when `tx.sender` itself has an EIP-7702 delegation. If the sender has delegated to a smart account via 7702, does the "default code" still apply? Or does the delegated code take over? The interaction between frame transactions and EIP-7702 delegated accounts needs explicit specification.

### Issue 10: Receipt Does Not Include Skipped Frame Status

When an atomic batch partially executes and then reverts (rolling back all frames in the batch), the receipt's `frame_receipt` array needs to represent the skipped frames. The spec defines `status` as "the return code of the top-level call" but does not specify what status value a skipped frame receives. Is it 0 (failure)? A new status code? Omitted from the array? This affects every block explorer and indexing service.

---

## 4. Usage Scenarios I Like

### Post-Quantum Migration Path

The most compelling use case is the one stated in the motivation: providing a native path from ECDSA to post-quantum cryptography. The default code already supports P256 as a second signature type, and the architecture allows any cryptographic scheme to be implemented in EVM code. When NIST finalizes ML-DSA (FIPS 204) or similar PQ schemes, smart accounts can adopt them without any further protocol changes. This is existentially important for Ethereum's long-term security.

### Gas Sponsorship Without Trusted Relayers

Example 3 (sponsored transactions) eliminates the need for a trusted relayer network like the current ERC-4337 bundler ecosystem. The sponsor is just another frame in the transaction, with its own VERIFY step and explicit payment approval. The trust boundary is enforced by the protocol (the sponsor must call APPROVE(0x2) and have sufficient balance) rather than by off-chain reputation systems. This dramatically simplifies the gas abstraction stack.

### Atomic Multi-Step DeFi Operations

The atomic batch feature directly addresses a real pain point in DeFi: multi-step operations where partial execution leaves the user in a worse state than not executing at all. Approve-then-swap (Example 2) is the canonical case, but this extends to any sequence: approve-deposit-stake, borrow-swap-repay, or complex rebalancing operations across multiple protocols.

### Account Deployment in the Same Transaction

Example 1b shows deploying a smart account and using it in the same transaction. This is a significant UX improvement over the current flow where account deployment and first use are separate transactions. For onboarding new users, this means a single interaction instead of two, with the deployment cost amortized into the first operation.

---

## 5. Additional Possibilities Not Being Discussed

### Programmable Transaction-Level Invariants

Because frames can introspect earlier frames' results via `TXPARAM(0x15, frame_index)` (status of prior frames), a DEFAULT-mode "invariant checker" frame at the end of a transaction could verify post-conditions across the entire transaction. For example: "after all SENDER frames execute, assert that my portfolio value has not decreased by more than 1%." This is a form of transaction-level assertion that does not exist in Ethereum today and would be valuable for MEV protection.

### Conditional Execution Chains

By checking prior frame statuses, later frames can implement conditional logic: "execute frame 3 only if frame 2 succeeded; otherwise execute frame 4." The spec does not discuss this pattern explicitly, but it falls out naturally from the TXPARAM(0x15) introspection capability. This enables try/catch semantics at the transaction level -- a significant expressiveness improvement over today's all-or-nothing transactions.

### Multi-Party Transactions Without Intermediaries

Frame transactions can represent multi-party operations where different frames are authorized by different parties. Consider an OTC trade: Alice's VERIFY frame approves sending token A, Bob's VERIFY frame approves sending token B, and two SENDER frames execute the swap atomically. Today this requires either a trusted escrow contract or an AMM. Frame transactions could enable direct peer-to-peer atomic swaps with no intermediary.

However, the spec currently requires a single `sender` per transaction, which limits this pattern. See Section 6 for how to extend it.

### Progressive Account Migration

A user could deploy a new smart account (frame 0), verify with their old ECDSA key (frame 1), and execute a migration operation that transfers assets and sets up the new account's validation logic (frames 2-N), all in a single atomic transaction. This makes account upgrades a single-block operation rather than a multi-transaction process with intermediate states.

### Block Builder Specialization for Frame Transactions

Block builders could offer specialized services for frame transactions: bundling multiple users' frame transactions together, sharing deployment costs for common account implementations, or batching sponsor payments. The frame structure makes these optimizations visible and verifiable at the protocol level, unlike current private mempool arrangements.

### Generalized Intents via Frame Composition

Frame transactions are essentially a protocol-level intent system. The VERIFY frame expresses the authorization intent ("I approve this class of operations"), the SENDER frames express the execution intent ("do these things"), and the gas payment mechanism expresses the payment intent ("pay for it this way"). A solver/filler could construct frame transactions that satisfy user-specified intents by composing appropriate frames, with the atomic batch guaranteeing all-or-nothing execution.

---

## 6. Changes to Increase Optionality

### 6.1 Explicit Bit Layout Specification

As discussed in Issue 2, the bit numbering is inconsistent between prose and pseudocode. The spec should include a definitive bit layout diagram:

```
mode field (uint16 or larger):

Bit:  15  14  13  12  11  10   9   8   7   6   5   4   3   2   1   0
     [-- reserved --] [AB] [-- scope --] [------ execution mode ------]

AB = Atomic Batch flag
scope = Approval scope constraint (2 bits)
execution mode = DEFAULT(0), VERIFY(1), SENDER(2)
```

And all pseudocode should use consistent 0-indexed bit operations.

### 6.2 Add a DELEGATE Mode

The current modes are DEFAULT (caller = ENTRY_POINT), VERIFY (static, caller = ENTRY_POINT), and SENDER (caller = tx.sender). A fourth mode, DELEGATE, where the frame executes in the context of the sender (like DELEGATECALL) but with a different target's code, would enable library-pattern calls without deploying code to the sender's address. This is particularly useful for smart accounts that want to use shared implementation libraries.

### 6.3 Inter-Frame Gas Donation (Opt-In)

Add a flag bit that allows a frame to donate its unused gas to the next frame. This would be opt-in (a new mode flag) and one-directional (only forward, never backward). It solves the gas estimation problem for multi-frame transactions where the total gas is predictable but the per-frame distribution is not.

```
| Mode bit | Meaning            | Valid with  |
|----------|--------------------|-------------|
| 12       | Donate unused gas  | Any mode    |
```

When a frame with this flag completes, its unused gas is added to the next frame's gas_limit. This is analogous to `Task.ContinueWith` inheriting the scheduler context -- the resource budget flows forward through the pipeline.

### 6.4 Frame-Level Return Data Accessibility

Currently, APPROVE takes offset/length parameters (like RETURN) to specify return data, but the spec does not describe how subsequent frames access this return data. Adding a `FRAMEDATARETURN` opcode (or extending TXPARAM) to read the return data from a prior frame would enable richer inter-frame communication. For example, a VERIFY frame could return a computed value (like a session key or spending limit) that subsequent SENDER frames use.

### 6.5 Extend TXPARAM for Blob Versioned Hash Access

TXPARAM provides `len(blob_versioned_hashes)` (param 0x07) but no way to access individual blob versioned hashes. For frame transactions that include blobs, the verification logic may need to check specific blob commitments. Add a param (e.g., 0x18) where `in2` is the blob index and the return value is the versioned hash.

### 6.6 Consider a Frame-Level Value Field

The rationale section says "No value in frame -- it is not required because the account code can send value." This is true but inefficient for the common case. A simple ETH transfer requires encoding the destination and amount in the frame's calldata and executing it via a SENDER frame that interprets the calldata. A native `value` field on the frame would save ~25 bytes of calldata encoding and reduce gas costs for the most common operation on the network.

The counter-argument is API surface minimalism, which I respect. But if data efficiency tables are being presented as a feature (as they are in this EIP), then the most common operation should be as efficient as possible. At minimum, the default code for SENDER mode could support a more compact encoding for simple ETH transfers.

### 6.7 Make MAX_FRAMES Smaller or Justify Its Size

MAX_FRAMES is 1000. The examples use 2-5 frames. The data efficiency section analyzes 2-4 frame transactions. What realistic use case needs 1000 frames? A large MAX_FRAMES increases the attack surface for DoS (1000 frames * 100,000 gas per validation frame = 100M gas just for validation overhead) and complicates gas estimation tooling. If no concrete use case needs more than ~20 frames, set MAX_FRAMES to 32 or 64 and increase it later if needed.

Conversely, if the intention is to support use cases like "batch 1000 token transfers in one transaction," the spec should describe this use case and analyze its gas and data efficiency characteristics.

### 6.8 Explicit Error Codes for APPROVE Failures

When APPROVE reverts (e.g., because `sender_approved` was already set, or `frame.target != tx.sender`), the revert reason is opaque. Adding standardized revert data (error selectors) would dramatically improve debuggability. Currently, a wallet seeing a reverted VERIFY frame has no protocol-level way to distinguish "wrong signature" from "insufficient balance for gas" from "sender already approved." This is the equivalent of catching `Exception` rather than `InvalidOperationException` vs `InsufficientFundsException` -- it makes error handling and user-facing error messages much harder.

### 6.9 Consider Separating Nonce Increment from Payment

Currently, APPROVE scope 0x2 both increments the nonce and collects gas payment. These are conceptually distinct operations. Separating them would:
- Allow nonce schemes that are independent of payment timing
- Enable multidimensional nonces (mentioned as a future extension in the TXPARAM notes) without modifying APPROVE
- Make the APPROVE opcode's behavior more predictable (fewer side effects per call)

The nonce increment could happen at the protocol level after successful validation (post-VERIFY), with APPROVE scope 0x2 only handling gas collection. This aligns with how existing transaction types work: the nonce is incremented by the protocol, not by user code.

### 6.10 Formalize the Canonical Paymaster Implementation

The mempool section references a "canonical paymaster implementation" repeatedly but does not include its code or specify it normatively. For a protocol-level EIP, the canonical paymaster should either be (a) included in the spec as normative code, or (b) published as a companion ERC with a specific reference. Currently, nodes must identify canonical paymasters "by runtime code match," but the code to match against is undefined. This is like documenting a type constraint without defining the type.

---

## Final Assessment

EIP-8141 is an ambitious and well-structured proposal that addresses a genuine protocol need. The frame abstraction is the right level of decomposition, and the separation of validation, execution, and payment is architecturally sound. The post-quantum migration story alone justifies the complexity.

However, the specification has several precision issues that will cause implementation divergence between clients if not resolved before finalization:

1. **Critical:** The bit numbering inconsistency between prose and pseudocode (Issue 2) must be resolved. This is a consensus-breaking ambiguity.
2. **Critical:** The `compute_sig_hash` mutation side effect (Issue 1) needs to specify copy semantics.
3. **High:** The nonce increment timing relative to validation (Issue 3) needs explicit multi-transaction-per-block analysis.
4. **High:** The APPROVE opcode's overloaded responsibilities (Section 2) will make future extensions difficult.
5. **Medium:** The gas isolation usability cost (Section 2) will create poor user experiences without tooling guidance.
6. **Medium:** The canonical paymaster is referenced but not defined (Section 6.10).

The design is strong enough that these issues are fixable without architectural changes. The frame transaction is the kind of well-factored abstraction that I would expect from this author list. Ship it, but fix the bit numbering first.
