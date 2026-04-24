# Review: EIP-8141 "Frame Transaction"

**Reviewer:** Deep protocol review  
**Date:** 2026-04-12  
**Status of EIP:** Draft  
**Authors:** Vitalik Buterin, lightclient, Felix Lange, Yoav Weiss, Alex Forshtat, Dror Tirosh, Shahaf Nacson, Derek Chiang

---

## 1. What I Like

### 1.1 The Frame Abstraction Is the Right Primitive

After years of watching account abstraction proposals accrete complexity at the wrong layer -- ERC-4337's off-chain `UserOperation` bundling, EIP-3074's `AUTH`/`AUTHCALL` dance, EIP-7702's delegation grafting -- the frame model finally asks the correct question: *what if the transaction itself was a programmable sequence of execution contexts?*

Frames are the right shape. A transaction is no longer a monolithic "sender signs, EVM executes" atom. It is an ordered list of typed execution stages with distinct callers, targets, and semantic purposes. This is the first AA proposal that treats the transaction as a *composition* rather than a *modification* of the existing model. That distinction matters enormously for forward compatibility.

### 1.2 VERIFY Mode as Static Validation Is Elegant

Making verification frames behave like `STATICCALL` -- no state writes, data elided from signature hash -- is a deeply considered design choice. It simultaneously:

- Prevents validation-time side effects that would create mempool invalidation vectors.
- Enables future signature aggregation by ensuring VERIFY data is opaque to other frames.
- Creates a clean separation between "proving you are who you claim" and "doing things."

The elision of VERIFY frame data from the signature hash is particularly thoughtful. It solves the recursive dependency problem (you cannot sign your own signature) while also leaving the door open for BLS or other aggregation schemes where individual proof data is replaced by a batch proof.

### 1.3 The APPROVE Opcode Is Surprisingly Well-Designed

Making approval a *new opcode* rather than a return value convention is the kind of decision that only comes from having been burned by the alternative. ERC-4337 suffered from the ambiguity of return values; existing contracts could not be retrofitted to return the right magic bytes. By introducing `APPROVE` as a distinct opcode that terminates execution and updates transaction-scoped state, the EIP:

- Avoids conflicts with existing contract return conventions.
- Makes approval semantically unambiguous -- there is no way to accidentally APPROVE.
- Allows the constraint that only `frame.target` can call APPROVE, preventing approval delegation attacks.

The scope operand (`0x1` execution, `0x2` payment, `0x3` both) cleanly separates the two fundamental authorization questions: "may this transaction act as me?" and "who pays?"

### 1.4 EOA Backwards Compatibility via Default Code

The default code mechanism is a masterclass in migration strategy. Rather than forcing users to choose between EOAs and smart accounts (as every prior proposal effectively did), EIP-8141 makes EOAs *behave as if they were smart accounts* with a protocol-defined default implementation. This means:

- Every existing EOA can use frame transactions immediately. No migration, no delegation, no 7702 setup.
- The default code supports both secp256k1 and P256, providing an immediate PQ migration path via the latter (though P256 is not itself PQ-secure, the extensibility is clear).
- SENDER mode default code interprets frame data as an RLP-encoded call list, giving EOAs native multicall capability.

This is the first AA proposal where the answer to "but what about my existing users?" is not a compromise -- it is a feature.

### 1.5 Atomic Batching as a Mode Flag

Using bit 11 of the mode field to mark atomic batch boundaries is an inspired piece of encoding economy. Rather than introducing a new frame type or a wrapper structure, consecutive SENDER frames with the flag set form implicit atomic groups. The "last frame without the flag terminates the batch" convention is clean and requires zero additional parsing logic.

This gives users approve-then-swap atomicity without smart contract wrappers, without multicall proxies, without any of the existing tooling that exists solely because the protocol could not express "these things succeed together or not at all."

### 1.6 Warm/Cold State Sharing Across Frames

Sharing the warm/cold access journal across frames is a subtle but important optimization. Without this, multi-frame transactions would pay repeated cold-access penalties for the same storage slots, making the frame model uneconomical for realistic use cases. This is the kind of detail that reveals the authors have actually profiled real transaction patterns.

### 1.7 Data Efficiency Is Competitive

At 134 bytes for a basic smart account transaction, the frame transaction is within striking distance of a legacy EIP-1559 transaction. The overhead is justified by the capabilities gained. The comparison with ERC-4337's ABI-encoded `UserOperation` (which wastes bytes on 32-byte field padding) is stark.

---

## 2. What I Don't Like / Concerns

### 2.1 MAX_FRAMES = 10^3 Is Absurdly High

One thousand frames in a single transaction is a surface area gift to adversaries. Even with per-frame gas limits, the structural overhead of initializing, snapshotting (for atomic batches), and tearing down 1000 execution contexts will create implementation-specific performance cliffs in client code.

No legitimate use case needs 1000 frames. The examples in the EIP use 2-5. A generous production estimate for the most complex sponsored-deployment-with-post-op transaction is maybe 8-10 frames. Setting MAX_FRAMES to 32 or even 64 would cover every foreseeable use case while dramatically reducing the attack surface for client implementation bugs.

The concern is not gas cost -- it is that client implementations will have code paths (frame index validation, atomic batch tracking, receipt construction) that are only exercised at high frame counts, and those paths will contain bugs that an attacker can trigger with a 999-frame transaction that nobody ever tested against.

### 2.2 ORIGIN Semantic Change Is More Breaking Than Acknowledged

The EIP states that `ORIGIN` returns the frame caller rather than the transaction origin, and waves this away by citing EIP-7702 precedent. This understates the impact.

EIP-7702 changed ORIGIN semantics for delegated accounts specifically. EIP-8141 changes ORIGIN semantics *for every contract called by a frame transaction*. Any contract anywhere in the call stack that uses `tx.origin` (and yes, many do, despite it being discouraged) will behave differently depending on whether it was called via a legacy transaction or a frame transaction.

This creates a class of bugs where contracts work correctly when called directly but fail when called through a frame transaction. The failure is silent -- `tx.origin` returns a valid address, just not the one the contract expected. For contracts that use `tx.origin == msg.sender` as an EOA-only gate, frame transactions break this invariant completely (ORIGIN returns ENTRY_POINT or tx.sender depending on mode, while CALLER is always the frame-level caller).

The EIP should enumerate the specific interaction patterns that break and provide guidance for contract developers.

### 2.3 Transient Storage Discarded Between Frames Is Surprising

The decision to discard `TSTORE`/`TLOAD` transient storage between frames is correct from a security perspective (preventing information leakage between validation and execution), but it violates the expectations of contracts that use transient storage for reentrancy guards.

Consider: a contract uses a transient storage flag to prevent reentrancy. Frame 1 calls this contract and sets the flag. Frame 2 calls the same contract -- the flag is gone. If the contract's security model assumes the flag persists for the duration of the transaction, frame transactions silently break that assumption.

The EIP should explicitly discuss this interaction and whether transient storage should persist across SENDER-mode frames (which are all acting on behalf of the same sender and conceptually represent a single user operation).

### 2.4 The Nonce Model Is Insufficiently Specified

The nonce is incremented inside `APPROVE(0x2)` or `APPROVE(0x3)` -- that is, during validation. But the EIP also requires `tx.nonce == state[tx.sender].nonce` as a stateful validation check *before* frame execution begins.

This means:
- Only one frame transaction per account can be pending at a time (since the nonce check happens before validation completes).
- There is no way to pipeline multiple frame transactions from the same account.
- The "possible future extension to allow indices for multidimensional nonces" mentioned in TXPARAM notes is doing enormous load-bearing work for future scalability, but is completely unspecified.

For high-frequency accounts (automated trading, governance multisigs), the single-nonce bottleneck is a real limitation. The EIP should either specify the multidimensional nonce extension or explicitly state that single-pending-transaction-per-account is a deliberate constraint and explain the reasoning.

### 2.5 Gas Accounting Creates Perverse Incentives

"Unused gas from a frame is not available to subsequent frames." This means transaction constructors must pre-estimate gas for each frame independently and cannot share a pool. In practice:

- Users will over-provision gas on each frame to avoid failures, wasting block space.
- The gas refund goes to the payer, but the block's gas capacity was consumed. A 10-frame transaction where each frame uses 50% of its allocation effectively wastes 50% of its total gas from the block's perspective.
- There is no mechanism for a frame to "donate" unused gas to a subsequent frame, which would be the natural solution for workflows where validation gas is predictable but execution gas is variable.

This is a meaningful inefficiency that will manifest as higher effective costs for frame transactions compared to equivalent legacy transactions.

### 2.6 The Paymaster Design Creates a Two-Tier System

The canonical paymaster carve-out is pragmatic but creates a protocol-level distinction between blessed and unblessed contract implementations. This has concerning implications:

- **Ossification risk:** The canonical paymaster implementation becomes de facto immutable. Any bug in it becomes a protocol-level vulnerability that cannot be patched without a hard fork or a new canonical implementation.
- **Innovation suppression:** Non-canonical paymasters are limited to 1 pending transaction per paymaster across the entire mempool. This makes them useless for any paymaster that serves more than a handful of users, effectively forcing all serious paymaster operators to use the canonical implementation.
- **Centralization pressure:** The canonical paymaster's timelocked withdrawal mechanism means paymaster operators must lock capital with no ability to withdraw quickly. This favors well-capitalized operators.

The 1-transaction limit for non-canonical paymasters is especially aggressive. Even a limit of 4-8 would allow experimentation without meaningfully increasing DoS risk.

---

## 3. Technical and Security Issues

### 3.1 APPROVE Reentrancy via Delegatecall Chains

The APPROVE opcode checks that `ADDRESS == frame.target`. But what about delegatecall chains? If contract A is the frame target and A delegatecalls to library B, and B executes APPROVE, `ADDRESS` is A (since delegatecall preserves the caller's context), so the check passes.

This is probably *intended* -- it allows smart accounts to use library contracts for validation logic. But it also means that any contract that can be delegatecalled by a frame target effectively inherits the ability to APPROVE transactions. If a smart account delegatecalls to an upgradeable library, and that library is compromised, the attacker can APPROVE arbitrary transactions.

The EIP should explicitly state that APPROVE is valid in delegatecall context and discuss the security implications for upgradeable validation libraries.

### 3.2 Signature Hash Malleability Window

The signature hash elides VERIFY frame data but includes everything else, including the frame ordering, targets, gas limits, and mode flags of all frames. This means:

- A sponsor's `pay` frame data is elided (it is a VERIFY frame), which is intentional -- the sender signs the transaction before the sponsor adds their signature.
- But the sender's VERIFY frame is *also* elided. So the sender signs a hash that commits to everything except the actual signature data in any VERIFY frame.

The concern: what happens if an attacker can submit a valid transaction with different VERIFY frame data? The signature hash is the same regardless of what signature scheme is used in the VERIFY frame, as long as the VERIFY frame succeeds. If the sender's smart account has multiple valid key configurations (e.g., during a key rotation), different signatures could produce the same signature hash, and a front-runner could substitute one valid signature for another.

In practice this is mitigated by the fact that different valid signatures produce the same outcome. But for exotic signature schemes that embed authorization metadata in the signature (e.g., "this signature is valid but rate-limited"), this malleability could be exploitable.

### 3.3 Atomic Batch Snapshot Depth Attack

Atomic batches take a state snapshot before the first frame and restore it if any frame reverts. With MAX_FRAMES at 1000 and the ability to create many consecutive atomic batches, an attacker could construct a transaction like:

```
[VERIFY, SENDER(atomic), SENDER, SENDER(atomic), SENDER, ...]
```

Each pair of SENDER frames forms a separate atomic batch. The client must maintain 500 independent snapshots (or 500 snapshot-restore cycles). Depending on the client's snapshot implementation (copy-on-write journal, database savepoint, etc.), this could be extremely expensive in memory or I/O.

Even if each batch is cheap to snapshot, 500 snapshot-restore cycles could exceed assumptions made by client implementations that were tested with 1-2 atomic batches.

### 3.4 The `sender_approved` Ordering Constraint Is Load-Bearing but Implicit

`APPROVE(0x2)` (payment approval) requires `sender_approved == true`. This means the sender must approve before the payer. This is stated as a note rather than as a formal constraint, and the implications are significant:

- It precludes "blind sponsorship" patterns where a sponsor pre-approves payment for a transaction they have not yet seen the sender approve.
- It forces a specific frame ordering in sponsored transactions: sender VERIFY must precede payer VERIFY.
- For aggregation scenarios where multiple senders and payers are involved, this ordering constraint limits the possible topologies.

This should be a first-class constraint with explicit rationale, not an implication buried in the APPROVE behavior section.

### 3.5 No Explicit Revert Data in Receipts

The receipt format includes `status` and `gas_used` per frame, but no revert data. For a transaction type that explicitly supports multi-frame execution where individual frames can fail, the inability to introspect *why* a frame failed (other than "it failed") is a debugging nightmare.

Legacy transactions already have this problem; frame transactions amplify it by having multiple independent failure points in a single transaction. The receipt should include revert data (at least a bounded prefix) for failed frames.

### 3.6 No Frame-Level Events for Indexing

There is no protocol-level event emitted when frames transition, when APPROVE is called, or when atomic batches are rolled back. This means indexers, block explorers, and analytics tools must reconstruct the frame execution flow from the receipt and trace data, with no standardized way to identify frame boundaries in the trace.

This will result in every indexer implementing slightly different frame-boundary detection heuristics, leading to inconsistent data across the ecosystem.

### 3.7 CREATE in SENDER Mode During Execution Frames

The banned opcode list applies only to the validation prefix. After `payer_approved = true`, execution frames can use any opcode. But what are the implications of CREATE/CREATE2 in SENDER mode?

In SENDER mode, the caller is `tx.sender`. So `CREATE` would deploy a contract whose deployer address is `tx.sender`, and `CREATE2` would use `tx.sender` as the deploying address for address calculation. This is correct and expected.

However, if the sender is an EOA (using default code), the nonce used for `CREATE` address derivation is... unclear. The EIP increments the nonce during APPROVE, but does it increment it again for CREATE? The interaction between the frame transaction nonce and the EVM's internal nonce tracking for CREATE needs explicit specification.

---

## 4. Compelling Usage Scenarios

### 4.1 Post-Quantum Migration Without Flag Day

This is the headline use case and it is genuinely important. Frame transactions allow a gradual PQ migration:

1. Users deploy smart accounts with PQ verification logic (e.g., ML-DSA, SLH-DSA, or lattice-based schemes via precompiles that can be added later).
2. The default code already supports P256, providing immediate hardware-backed key support via secure enclaves.
3. The signature scheme is account-specific, so different users can migrate at different times.
4. No hard fork is needed to add a new signature scheme -- just deploy a new smart account implementation.

This is the first proposal that makes PQ migration a user-space decision rather than a protocol-level flag day. That alone may justify the complexity.

### 4.2 ERC-20 Gas Payment Without Trusted Relayers

The sponsored transaction flow (Example 3 in the EIP) enables a trust-minimized ERC-20 gas payment pattern:

1. Sender signs the transaction.
2. Sponsor validates that the sender will pay them in ERC-20 tokens (frame 1 checks ERC-20 balance and the subsequent frame's calldata).
3. Sponsor's APPROVE(0x2) pays gas in ETH.
4. Frame 2 transfers ERC-20 tokens to the sponsor.
5. Frame 4 (post-op) handles refunds for unused gas.

This eliminates the trusted relayer from the ERC-20 gas payment flow. The sponsor only pays ETH if they are guaranteed ERC-20 compensation, and the sender only transfers tokens if the transaction executes. The atomicity is protocol-enforced.

### 4.3 Social Recovery Without Intermediary Contracts

A smart account can implement social recovery directly in its validation logic:

- Normal operation: single-key APPROVE.
- Recovery mode: VERIFY frame targets a recovery contract that collects guardian signatures and calls APPROVE.

The recovery contract does not need to be the account itself -- it can be a shared recovery protocol. The frame structure means the recovery flow is a single transaction: VERIFY (collect guardian approvals) -> SENDER (rotate key). No multi-transaction ceremony, no timelock contracts as intermediaries.

### 4.4 Programmable Transaction Policies

Smart accounts can enforce arbitrary transaction policies during VERIFY:

- Spending limits: VERIFY checks cumulative daily spend against storage.
- Destination whitelists: VERIFY examines subsequent frame targets.
- Time-based restrictions: "This key can only transact during business hours."
- Value-based key tiers: "Transactions above 10 ETH require the cold key."

The TXPARAM opcode makes this natural -- VERIFY frames can introspect the entire transaction structure (targets, gas limits, modes of all frames) before deciding whether to APPROVE.

### 4.5 Multi-Operation Transactions as a First-Class Primitive

The combination of SENDER mode and atomic batching gives every account native multicall:

- Approve + swap in one transaction (atomic).
- Claim rewards + restake in one transaction.
- Withdraw from multiple positions + deposit into a new position, all atomic.

This eliminates an entire class of smart contract wrappers (multicall, batch executor, etc.) that exist solely because the protocol could not express multi-operation transactions.

---

## 5. Capabilities People May Not Be Thinking About

### 5.1 Cross-Frame Verification for Intent Protocols

VERIFY frames can introspect subsequent frames via TXPARAM and FRAMEDATALOAD. This means a VERIFY frame can act as an *intent verifier*: the user signs an intent ("swap X for at least Y"), and the VERIFY frame checks that the subsequent execution frames satisfy the intent before calling APPROVE.

This is a protocol-native intent system. No off-chain solvers, no trusted matchers. The user's smart account *is* the intent verifier. An MEV-resistant swap becomes:

```
VERIFY: Check that frame 2 calls DEX.swap() and frame 3 calls token.balanceOf(sender) >= Y
SENDER: approve token
SENDER: swap
```

The VERIFY frame can reject the transaction if the execution frames do not satisfy the user's constraints. A builder who tries to sandwich the swap would need to modify the execution frames, which would change the signature hash, invalidating the transaction.

### 5.2 Composable Account Policies via Frame Inspection

Because VERIFY frames can read all frame metadata via TXPARAM, a new pattern emerges: *policy contracts*. A user's smart account can delegatecall to a policy library that inspects the transaction structure and enforces constraints:

- "All SENDER frames must target contracts in my whitelist."
- "Total value transferred across all frames must not exceed my daily limit."
- "If any frame targets the governance contract, require 2-of-3 multisig."

These policies compose: a user can stack multiple policy checks in their VERIFY logic. The policies are enforced *before* execution, not after, which means invalid transactions never execute. This is a fundamentally different security model than post-hoc slashing or dispute resolution.

### 5.3 Programmable MEV Resistance via Frame Ordering Constraints

A smart account's VERIFY logic can enforce constraints on frame ordering that make MEV extraction difficult:

- "The swap frame must immediately follow the approval frame" (prevents sandwich insertion).
- "No DEFAULT-mode frames may appear between my SENDER frames" (prevents builder-injected execution contexts).
- "The last frame must be a balance check that reverts if my position decreased" (enforces post-condition).

Combined with atomic batching, this creates user-level MEV protection that is enforced by the protocol, not by social convention or trusted builders.

### 5.4 Delegated Execution Without Delegation

SENDER mode frames act on behalf of the sender. But the *content* of those frames is committed to in the signature hash. This means a user can grant specific, bounded execution authority without any on-chain delegation setup:

- "Execute this specific swap with these specific parameters, on my behalf."
- No approval, no allowance, no delegate registry.
- The authority is single-use (bound to this transaction's nonce) and fully specified (bound to the exact calldata in the frame).

This is "just-in-time delegation" -- the user signs a complete execution plan, and the plan is its own authorization. This eliminates entire categories of approval-related vulnerabilities (infinite approvals, stale allowances, approval front-running).

### 5.5 Key Rotation Ceremonies as Single Transactions

A key rotation for a smart account becomes:

```
VERIFY: Old key signs APPROVE(0x1) + APPROVE(0x2)
SENDER: Call account.rotateKey(newPublicKey)
```

One transaction. No timelock. No social recovery ceremony unless the old key is lost. The old key authorizes its own replacement in a single atomic operation.

For more paranoid setups:

```
VERIFY: Old key signs APPROVE(0x1) with scope constraint
VERIFY: Guardian co-signs APPROVE(0x2)
SENDER: Call account.rotateKey(newPublicKey)
```

Two-party key rotation in a single transaction, with the guardian paying gas.

### 5.6 Governance Without Governor Contracts

A DAO could implement its governance logic as a smart account:

```
VERIFY: Collect member signatures (multi-VERIFY frames or aggregated proof)
SENDER: Execute proposal (arbitrary calls from the DAO account)
```

The DAO's account IS the governance system. No Governor contract, no Timelock contract, no proposal lifecycle. A quorum of members signs a frame transaction, and the proposal executes atomically. This is governance reduced to its essential form: collective authorization of a specific action.

### 5.7 Conditional Execution via Post-Conditions

Since SENDER frames can revert, and since atomic batching groups frames, users can create post-condition checks:

```
VERIFY: Sign APPROVE(0x3)
SENDER (atomic): Swap ETH for TOKEN on DEX
SENDER: Check TOKEN balance >= minimum, revert if not
```

If the balance check fails, the atomic batch reverts, including the swap. The user never receives fewer tokens than they specified. This is protocol-native slippage protection without relying on the DEX's own slippage parameters.

### 5.8 Account Abstraction for L2 Bridges

Frame transactions on L1 can encode bridge operations as atomic multi-frame sequences:

```
VERIFY: Authorize
SENDER (atomic): Approve tokens to bridge
SENDER (atomic): Call bridge.deposit()
SENDER: Verify bridge receipt
```

The atomicity ensures the token approval and bridge deposit happen together, preventing the "approved but not deposited" failure mode that currently requires multicall wrappers.

---

## 6. Suggestions for Increasing Optionality

### 6.1 Add a Frame-Level Value Field

The EIP omits a value field from frames, noting "the account code can send value." While true, this forces every ETH transfer to go through a smart account's SENDER-mode code path, adding unnecessary gas overhead for the most common operation on Ethereum. A direct value field on frames would be more efficient and would simplify the default code for EOAs.

Even a minimal `value` field (0 for most frames, non-zero for transfers) would save gas on ETH transfers by avoiding the calldata encoding overhead.

### 6.2 Lower MAX_FRAMES, Add a Future Extension Path

Set MAX_FRAMES to 32 for the initial deployment. Add a note that this can be increased in a future hard fork. No legitimate use case needs more than 32 frames in the near term, and the reduced surface area significantly simplifies client implementation and testing.

If a use case genuinely needs more than 32 frames in the future, it can be accommodated by increasing the constant. Going the other direction (reducing from 1000) is politically much harder.

### 6.3 Specify Multidimensional Nonces or Remove the Hint

The TXPARAM table mentions "possible future extension to allow indices for multidimensional nonces" but provides no specification. Either:

- Specify the multidimensional nonce scheme now, even if the initial implementation only uses dimension 0. This allows smart accounts to implement nonce channels from day one.
- Remove the mention entirely. Half-specified future extensions create confusion and encourage incompatible implementations.

Nonce channels (a la ERC-4337's nonce key space) are important for enabling concurrent transactions from the same account. The frame transaction already supports the concept (the sender's smart account can implement arbitrary nonce logic in VERIFY), but protocol-level support would be more efficient.

### 6.4 Allow Gas Donation Between Frames

Add a mechanism for unused gas from one frame to be forwarded to a subsequent frame. This could be a mode flag ("donate unused gas to next frame") or a new opcode. Without this, users must over-provision gas on each frame, reducing the efficiency of multi-frame transactions.

The current design where unused gas is refunded after all frames complete means the gas *is* available -- it is just not available to subsequent frames during execution. Allowing gas forwarding would not change the total gas consumed but would reduce the amount of wasted provisioning.

### 6.5 Include Revert Data in Frame Receipts

Extend the frame receipt to `[status, gas_used, logs, revert_data]` where `revert_data` is the first N bytes (e.g., 256) of the revert reason for failed frames, or empty for successful frames. This is critical for debugging multi-frame transactions.

### 6.6 Add a FRAMESTATUS Opcode or Extend TXPARAM

Allow frames to query the execution status of previous frames. TXPARAM `0x15` already returns the status of prior frames, but adding the ability to read revert data from prior frames would enable sophisticated error handling patterns -- e.g., a post-op frame that adjusts its behavior based on why the execution frame failed.

### 6.7 Consider a FRAME_SCOPE Precompile Instead of APPROVE Opcode

An alternative to the APPROVE opcode: a precompile at a well-known address that achieves the same effect. This would:

- Avoid consuming opcode space (opcodes are a finite resource).
- Allow the approval logic to be upgraded without an opcode-level hard fork.
- Be compatible with existing tooling that understands calls to precompiles but not new opcodes.

The trade-off is a slightly higher gas cost (call overhead vs. opcode overhead) and slightly more complex implementation. But opcode space conservation may be worth it, especially given that this EIP already introduces four new opcodes.

### 6.8 Explicitly Support Frame-Level Access Lists

The EIP rationale argues against access lists by citing their poor risk-reward. But frame-level access lists could be different: since each frame has a well-defined scope and purpose, the access list for a VERIFY frame (which only reads sender storage) is highly predictable. A per-frame access list would allow clients to pre-warm only the storage actually needed by each frame, reducing worst-case I/O.

This is not urgent for the initial deployment but should be considered as a future extension.

### 6.9 Define a Standard Frame Event Schema

Emit a protocol-level log (via a system contract or pseudo-precompile) at frame boundaries:

```
FrameExecuted(uint256 indexed frameIndex, uint8 mode, address target, bool success, uint256 gasUsed)
```

This would give indexers a standardized way to track frame execution without reconstructing it from traces.

### 6.10 Relax Non-Canonical Paymaster Limits with Staking

The EIP removes staking entirely from the mempool policy, citing simplicity. But staking served a useful purpose in ERC-7562: it allowed paymasters to earn mempool trust by putting capital at risk. A middle ground:

- Non-canonical paymasters with no stake: 1 pending transaction (current design).
- Non-canonical paymasters with N ETH staked to a well-known contract: up to M pending transactions, where M scales with stake.

This preserves the safety of the current design as a default while providing an escape hatch for innovative paymaster designs that cannot use the canonical implementation.

---

## 7. Summary Assessment

EIP-8141 is the most architecturally sound account abstraction proposal Ethereum has produced. The frame model is the correct abstraction level, the APPROVE opcode is well-designed, and the EOA backwards compatibility story is the best in the history of AA proposals.

The primary concerns are:

1. **MAX_FRAMES is too high.** Lower it to 32-64.
2. **Gas non-transferability between frames** will cause meaningful inefficiency in practice.
3. **The canonical paymaster carve-out** creates ossification risk and suppresses innovation.
4. **Transient storage semantics across frames** will surprise contract developers.
5. **ORIGIN semantic changes** are more impactful than acknowledged.
6. **The nonce model** needs either explicit multidimensional support or explicit acknowledgment of the single-pending-transaction constraint.

The capabilities this enables -- programmable MEV resistance, protocol-native intents, single-transaction key rotation, trust-minimized ERC-20 gas payment, gradual PQ migration -- are transformative. The frame model is not just an account abstraction mechanism; it is a transaction-level programmability primitive that will enable patterns we have not yet imagined.

The EIP should ship, but with the parameter adjustments and specification clarifications noted above. The frame model is the right shape. It just needs its pressure hull tightened before it goes to depth.
