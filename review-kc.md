# EIP-8141 "Frame Transaction" -- Comprehensive Review

**Reviewer:** Kamil Chodola (QA perspective)
**Date:** 2026-04-12
**Status:** Draft Review

---

## 1. What I Like About It

### 1.1 The Frame Abstraction Is Genuinely Elegant

The core insight -- decomposing a transaction into an ordered sequence of typed frames with distinct execution contexts -- is a clean primitive. It avoids the "god object" problem that plagued earlier AA proposals (EIP-4337's UserOperation struct was monolithic). Each frame has a single concern: verify, execute, or call generically. The mode system (VERIFY / SENDER / DEFAULT) maps neatly to the lifecycle of a transaction: authenticate, then act.

### 1.2 Signature Hash Elision Is Well-Considered

Eliding VERIFY frame data from the signature hash (Section "Signature Hash") is one of the best design decisions in this EIP. It simultaneously solves three problems: (a) signatures cannot sign over themselves, (b) it enables future signature aggregation without breaking the format, and (c) it allows paymaster signatures to be appended after the sender signs. This is a non-obvious triple win.

### 1.3 APPROVE as an Opcode Rather Than a Return Value

The rationale explains this clearly -- using a new opcode instead of overloading RETURN means existing deployed contracts could theoretically be upgraded to support frame transactions. The separation of "exit successfully" from "grant approval" is clean. The fact that only `frame.target` can call APPROVE (preventing delegated approval injection) is a solid security boundary.

### 1.4 Atomic Batching via Flag Composition

Using bit 11 of the mode field to chain consecutive SENDER frames into an atomic batch is compact and expressive. The "approve then swap" example (Example 2) is a real-world pattern that currently requires multicall contracts or flash-loan wrappers. Making it native is a genuine UX improvement. The design that the last frame in a batch does NOT have the flag set provides a clean termination signal.

### 1.5 Default Code for EOAs Is a Practical Bridge

Supporting EOAs with "virtual" default code is pragmatic. It means existing users can benefit from gas abstraction and P256 signatures without deploying a smart account first. The P256 support in default code (signature type 0x1) is forward-looking for hardware key / passkey adoption.

### 1.6 Warm/Cold State Shared Across Frames

Sharing the warm/cold access journal across frames (Section "Frame interactions") avoids double-charging for storage that multiple frames touch. This is a subtle but important efficiency gain that shows attention to practical gas costs.

### 1.7 Mempool Rules Are Actually Specified

Unlike many EIPs that hand-wave mempool behavior with "nodes SHOULD be careful," this EIP defines concrete validation prefixes, banned opcodes, structural rules, and a reservation-based paymaster accounting system. The fact that it draws from ERC-7562's battle-tested approach but simplifies it (no staking, no reputation) is a reasonable trade-off for a protocol-level primitive.

---

## 2. What I Don't Like About It

### 2.1 Complexity Budget

This EIP introduces four new opcodes (APPROVE, TXPARAM, FRAMEDATALOAD, FRAMEDATACOPY), a new transaction type with a novel frame structure, new execution semantics (ORIGIN changes, transient storage clearing between frames, atomic batching), extensive mempool rules, a canonical paymaster contract, and default code for EOAs. This is an enormous surface area for a single EIP. Each piece is individually reasonable, but the combined complexity is daunting for client implementers. I would have expected this to be split into at least two EIPs -- one for the core frame mechanism and one for the mempool/paymaster rules.

### 2.2 ORIGIN Semantics Change Is a Landmine

The specification states: "The ORIGIN opcode returns frame caller throughout all call depths." This means ORIGIN returns ENTRY_POINT (0xaa) for DEFAULT/VERIFY frames and tx.sender for SENDER frames. This is a fundamental behavioral change. While the spec notes this is "consistent with the precedent set by EIP-7702," the reality is that ORIGIN is used in many deployed contracts for access control, reentrancy checks, and tx.origin-based guards. Changing its semantics per-frame creates a situation where the same contract behaves differently depending on which frame called it. This is not merely a backwards compatibility issue; it is a new class of confused-deputy vulnerability.

### 2.3 Transient Storage Clearing Between Frames Is Surprising

The spec says "Discard the TSTORE and TLOAD transient storage between frames." Transient storage (EIP-1153) was designed to be transaction-scoped. Clearing it between frames violates a fundamental assumption that many protocols (flash loans, reentrancy locks, callback patterns) rely on. A contract that uses TSTORE for reentrancy protection within a single transaction could be vulnerable if frame 1 sets the lock, the lock is cleared, and frame 2 re-enters. This needs much stronger justification and analysis of existing usage patterns.

### 2.4 Gas Isolation Between Frames Is Inflexible

"Unused gas from a frame is not available to subsequent frames." This means the transaction author must pre-allocate gas to each frame, which is impossible to do optimally without simulating the transaction first. If frame 1 uses less gas than allocated, that gas is wasted from the perspective of subsequent frames. This creates a perverse incentive to over-allocate gas to every frame (to avoid failures), which then increases the upfront cost the payer must lock up. For complex multi-frame transactions, this could mean significant capital inefficiency.

### 2.5 MAX_PENDING_TXS_USING_NON_CANONICAL_PAYMASTER = 1 Is Too Restrictive

Setting this to 1 means that for any non-canonical paymaster, only a single pending transaction can exist in the mempool at any time. This effectively makes non-canonical paymasters unusable for any meaningful throughput. If a user has a "gas account" EOA as a paymaster, they can only have one pending sponsored transaction at a time across the entire mempool. This feels like a parameter that was set conservatively but may need to be reconsidered.

### 2.6 The Canonical Paymaster Is Underspecified

The EIP references a "canonical paymaster implementation" multiple times and builds critical mempool rules around it, but the actual code/specification of this canonical paymaster is not included in the EIP. It references "runtime code exactly matches the canonical paymaster implementation" without defining what that implementation is. This is a critical dependency that is missing from the specification. Reviewers cannot evaluate whether the paymaster-specific accounting rules are sound without seeing the implementation.

### 2.7 No Value Field in Frames

The rationale says "It is not required because the account code can send value." While true, this forces every ETH transfer to go through contract code (either default code's RLP-decoded calls or a smart account's execution logic). This adds gas overhead and complexity for what is currently the simplest possible transaction. The "Example 1a: Simple ETH transfer" demonstrates this -- it requires encoding the destination and amount in frame data and relying on the default code to decode and execute it.

---

## 3. Issues (Bugs, Inconsistencies, Ambiguities, Edge Cases)

### 3.1 [CRITICAL] TXPARAM Param ID Gap: 0x09 to 0x10

The TXPARAM table has param IDs 0x00-0x09, then jumps to 0x10. This is clearly intentional (0x0A-0x0F are skipped), but the spec says "Invalid param values (not defined in the table above) result in an exceptional halt." This means there is a discontinuous valid range: 0x00-0x09 and 0x10-0x17. The gap at 0x0A-0x0F should either be explicitly reserved or the numbering should be made contiguous. As written, it is easy for implementers to introduce off-by-one errors if they use a simple range check.

### 3.2 [CRITICAL] TXPARAM 0x09 Says "can be zero" But Constraints Say len(frames) > 0

The TXPARAM table says `len(frames)` "can be zero" for param 0x09, but the static constraints section states `assert len(tx.frames) > 0`. These are contradictory. If the constraint is enforced, TXPARAM 0x09 can never return zero during execution, making the "(can be zero)" note misleading. If this is meant to handle a future extension where frames could be empty, the constraint needs to be relaxed. This needs clarification.

### 3.3 [CRITICAL] Nonce Increment Timing and Replay Protection

The nonce is incremented inside the APPROVE opcode (scope 0x2 or 0x3), not at the start of execution. This means that during the entire validation prefix (which may execute substantial code), the nonce has not been incremented. If the VERIFY frame reverts after some state changes (though VERIFY is STATICCALL-like, so this should not happen), or if the payer's APPROVE reverts, the nonce remains unchanged. However, the spec also says the stateful validation check is `tx.nonce == state[tx.sender].nonce`. The question is: what prevents replay of a frame transaction whose APPROVE(0x2) reverted? The sender's nonce was not incremented, so the same transaction can be resubmitted. This is by design for most cases but needs explicit discussion of the implications.

More critically: what happens if two different frame transactions from the same sender with the same nonce are both valid? The mempool rule says "at most one pending frame transaction per sender," but this does not prevent miners from including duplicates across blocks if the nonce is never incremented due to consistent APPROVE failure.

### 3.4 [HIGH] Atomic Batch Flag Validation Has an Off-by-One-Like Issue

The static constraint says:
```python
if (frame.mode >> 10) & 1 == 1:
    assert i + 1 < len(tx.frames)
    assert (tx.frames[i + 1].mode & 0xFF) == 2
```

This checks bit 11 via `>> 10`, but the mode flags table says bit 11 is the atomic batch flag. Shifting right by 10 and masking with 1 extracts bit 10 (zero-indexed), which is the second bit of the approval scope, NOT bit 11. This is a bug in the pseudocode. To extract bit 11, the code should be `(frame.mode >> 10) & 1` only if bits are 0-indexed starting from bit 0 as the LSB and "bit 11" means the 11th bit (0-indexed), which would indeed be `>> 10`. But the mode flags table says bits 9-10 are approval scope. If the bits are labeled 1-indexed (bit 1 = LSB), then bit 11 is `>> 10`. If 0-indexed, bit 11 is `>> 11`.

The ambiguity in bit numbering (are they 0-indexed or 1-indexed?) is a specification-level issue that will cause implementation bugs. The mode flags table says "bits 9-10" for approval scope, and the APPROVE section says `(frame.mode >> 8) & 3` to extract those bits. If we work backwards: `>> 8` with `& 3` extracts bits 8 and 9 (0-indexed). But the table calls them "bits 9-10." This implies the table uses 1-indexed bit numbering. Therefore "bit 11" in the table is bit 10 in 0-indexed, and `>> 10` is correct. But the inconsistency between the table labeling and the code will cause confusion. The EIP should explicitly state its bit-numbering convention.

### 3.5 [HIGH] APPROVE Scope 0x2 Requires sender_approved == true, But Who Sets It?

For scope 0x2 (payment only), the spec says: "If sender_approved == false, revert the frame." This means you cannot pay without first approving execution. But scope 0x1 requires `frame.target == tx.sender`. So the sender must first approve execution in a VERIFY frame targeting itself, then a different VERIFY frame (targeting the paymaster) must approve payment. The paymaster's APPROVE(0x2) increments the **sender's** nonce, not the paymaster's nonce, and collects gas cost from the **paymaster's** account (since frame.target is the paymaster).

Wait -- re-reading: scope 0x2 says "collect the total gas cost of the transaction from the account." Which account? It says "from the account" referring to `frame.target`. But it also says "Increment the sender's nonce." So scope 0x2 increments `tx.sender`'s nonce but collects payment from `frame.target` (the paymaster). This is internally consistent but the wording "from the account" is ambiguous. It should say "from frame.target" explicitly.

### 3.6 [HIGH] Default Code SENDER Mode: Reverting on ANY Sub-Call Revert Is Too Strict

The default code for SENDER mode says: "For each call in calls, execute the call with msg.sender = tx.sender. If any call reverts, revert the frame." This means if a multicall batch has 10 calls and the last one reverts, all 10 are reverted. But atomic batching exists at the frame level to handle this. The problem is that EOA default code batches calls within a single frame, so frame-level atomic batching does not help. An EOA wanting partial-success batching must use separate SENDER frames for each call, which is more expensive.

### 3.7 [HIGH] compute_sig_hash Mutates the Transaction Object

The signature hash computation function in the spec is:
```python
def compute_sig_hash(tx: FrameTx) -> Hash:
    for i, frame in enumerate(tx.frames):
        if (frame.mode & 0xFF) == VERIFY:
            tx.frames[i].data = Bytes()
    return keccak(rlp(tx))
```

This mutates `tx.frames[i].data` in place. If the implementation is not careful to clone the transaction before computing the signature hash, the original transaction data is destroyed. This should either use a copy or be specified as operating on a copy. As written, calling this function twice would produce different results (the first call would elide the data, the second call would hash already-elided data -- which is the same result, but the transaction object is now permanently modified).

### 3.8 [MEDIUM] What Happens When frame.target Has EIP-7702 Delegated Code?

The mempool validation trace rules say: "execution performs CALL* or EXTCODE* to an address that ... uses an EIP-7702 delegation, except for tx.sender default-code behavior." But what if tx.sender itself has EIP-7702 delegated code? Is the delegation followed? The default code section says "accounts with no code" get default code, but an account with 7702 delegation technically has a delegation designator, not "code" in the traditional sense. The interaction between EIP-7702 delegated accounts and EIP-8141 default code needs explicit specification.

### 3.9 [MEDIUM] FRAMEDATACOPY Gas Cost Is Underspecified

The spec says "The gas cost matches CALLDATACOPY" but CALLDATACOPY has a complex gas cost: 3 base + 3 * ceil(length/32) copy cost + memory expansion cost. The spec mentions "a fixed cost of 3 and a variable cost that accounts for the memory expansion and copying" but does not give the exact formula. For a specification document, relying on "matches CALLDATACOPY" is insufficient -- implementers need to know whether the gas schedule is literally identical or merely similar.

### 3.10 [MEDIUM] Skipped Frames in Atomic Batch: What Status Do They Report?

When frames are skipped due to atomic batch revert, what does TXPARAM(0x15, frame_index) return for them? The spec says status returns 0 for failure or 1 for success, but "skipped" is neither. Do skipped frames appear in the receipt? If so, what is their status, gas_used, and logs? The spec does not address this.

### 3.11 [MEDIUM] APPROVE Return Data Is Never Used

APPROVE takes offset and length from the stack (like RETURN) but the spec never describes what happens to the return data. Is it available to subsequent frames? Is it discarded? For scope 0x1 (execution approval), return data might be useful for passing information to subsequent frames. For scope 0x2 (payment), the payer might want to return data. This is unspecified.

### 3.12 [MEDIUM] Race Condition in Paymaster Reservation Accounting

The canonical paymaster accounting relies on nodes maintaining `reserved_pending_cost(paymaster)`. This is a local per-node value that is not synchronized across the network. Two nodes could independently admit transactions that exceed the paymaster's actual balance, because each node only tracks its own reservations. When these transactions are gossiped, the receiving node would reject one, but the submitting node has already propagated it. This is inherent to any mempool reservation scheme, but the EIP does not discuss how to handle the case where a node receives a gossiped transaction that exceeds its local reservation limit.

### 3.13 [MEDIUM] What If tx.sender Has Code But No APPROVE Logic?

If tx.sender has code (not an EOA) but that code does not implement APPROVE handling, then VERIFY frames targeting it will always fail (because APPROVE never gets called). The transaction is invalid per the spec. But this could be confusing for users who deploy arbitrary contracts and then try to use them as transaction senders. There is no mechanism to distinguish "this contract does not support frame transactions" from "this contract's validation logic rejected this specific transaction."

### 3.14 [MEDIUM] Default Code References TXPARAMLOAD Which Is Not Defined

The default code section says "Retrieve the mode with TXPARAMLOAD" but the opcode table defines TXPARAM (0xb0), not TXPARAMLOAD. The Python code below it uses comments referencing "TXPARAMLOAD(0x14, TXPARAMLOAD(0x10))". Similarly, the Rationale section references "TXPARAMLOAD" in "the canonical signature hash is provided in TXPARAMLOAD." This naming inconsistency will confuse implementers. The opcode name should be consistent throughout the document.

### 3.15 [LOW] ENTRY_POINT Address 0xaa Conflicts with APPROVE Opcode 0xaa

The ENTRY_POINT address is `0xaa` and the APPROVE opcode is also `0xaa`. While these are in entirely different namespaces (address vs. opcode), the collision is aesthetically unfortunate and could cause confusion in documentation and debugging. More practically, address 0xaa is a precompile-range address. Is there a precompile at this address? If not, what happens if someone sends ETH to it or calls it directly outside a frame transaction?

### 3.16 [LOW] calldata_cost Applies to rlp(tx.frames) But What About Other Fields?

The gas accounting formula is:
```
tx_gas_limit = FRAME_TX_INTRINSIC_COST + calldata_cost(rlp(tx.frames)) + sum(frame.gas_limit)
```

This charges calldata cost only for `rlp(tx.frames)`. But the transaction also includes chain_id, nonce, sender, fee fields, and blob hashes. Are these not charged calldata cost? In standard transactions, the entire RLP-encoded transaction is charged. Excluding the header fields from calldata cost means those bytes are "free" from a gas perspective, which could be exploited by stuffing large values into fields like nonce or chain_id (though both have upper bounds).

### 3.17 [LOW] No Maximum Size for frame.data

There is no specified maximum size for the data field of individual frames. A transaction could have a single frame with megabytes of data in it. While the block gas limit implicitly bounds this (via calldata cost), the calldata cost formula only charges for `rlp(tx.frames)`, so the data is at least charged. But there may be denial-of-service implications for deserialization and memory allocation. A MAX_FRAME_DATA_SIZE would provide a clearer bound.

### 3.18 [LOW] The Rationale Lists Three Reasons Under "Two Reasons"

The Rationale section on canonical signature hash says: "This is done for two reasons:" and then lists items 1, 2, and 3. This is a minor editorial error but worth fixing.

### 3.19 [LOW] Default Code self_verify Uses APPROVE(0x2) But Structural Rules Say APPROVE(0x0)

In the Structural Rules section (rule 3):
- "self_verify must call APPROVE(0x2)."
- "only_verify must call APPROVE(0x0)."

But APPROVE(0x0) is not a valid scope. The APPROVE spec says scope must be 0x1, 0x2, or 0x3 -- "Any other value results in an exceptional halt." So APPROVE(0x0) would cause an exceptional halt. This appears to be a bug. Looking at the intended semantics, `only_verify` should call APPROVE(0x1) (approve execution only), and `self_verify` should call APPROVE(0x3) (approve both execution and payment). The "self_verify must call APPROVE(0x2)" is also wrong -- a self-relayed transaction needs both sender and payer approval, which is scope 0x3.

Comparing with Examples 1 and 3: Example 1 says "Frame 0 verifies the signature and calls APPROVE(0x3)" for a self-relayed transaction. Example 3 says Frame 0 calls "APPROVE(0x1)" and Frame 1 calls "APPROVE(0x2)." This confirms the structural rules have the wrong scope values. The rules should say:
- `self_verify` must call `APPROVE(0x3)`.
- `only_verify` must call `APPROVE(0x1)`.

### 3.20 [CRITICAL] APPROVE Scope 0x2 Says "Increment the sender's nonce" -- But Whose Nonce?

When a paymaster calls APPROVE(0x2), the spec says "Increment the sender's nonce." This means the paymaster's VERIFY frame increments `tx.sender`'s nonce, not the paymaster's own nonce. This is necessary for replay protection of the sender's transaction. But it also means the paymaster has no nonce protection of its own -- a paymaster could approve the same payment twice if the sender replays the transaction. The mempool rules prevent this (one pending tx per sender), but in a builder/block-building context without mempool rules, a malicious builder could include the same frame transaction multiple times if the nonce is only checked at the start of execution and the first copy's APPROVE hasn't committed yet.

Actually, the stateful validation check `tx.nonce == state[tx.sender].nonce` happens at the start of transaction processing, and the nonce is incremented in APPROVE. If the first transaction is included and its APPROVE succeeds (incrementing the nonce), the second copy would fail the stateful validation check. But if two copies are in the same block and the first one's APPROVE reverts (so nonce is not incremented), the second copy could also execute and also revert. This wastes block space but is probably not a security issue beyond DoS.

### 3.21 [MEDIUM] Behavior Section Numbering Starts at 2

The "Behavior" section lists steps starting at "2. Execute a call..." and "3. If frame has mode VERIFY..." There is no step 1 (the stateful validation and initialization are listed as bullet points before the numbered list, but the numbering starting at 2 suggests a step 1 was removed or mis-numbered). This is an editorial issue but could indicate missing specification content.

### 3.22 [MEDIUM] VERIFY Mode Is STATICCALL-Like, But What About APPROVE's State Changes?

VERIFY mode says "The execution behaves the same as STATICCALL, state cannot be modified." But APPROVE(0x2) and APPROVE(0x3) modify state (increment nonce, transfer balance). How can a STATICCALL-like frame modify state? The answer must be that APPROVE is an exception to the STATICCALL restriction -- it is a privileged opcode that can modify state even within a STATICCALL context. But this exception is not stated in the spec. This needs explicit clarification: "VERIFY mode frames execute as STATICCALL, except that the APPROVE opcode may modify transaction-scoped state (nonce increment and balance transfer for payment)."

### 3.23 [MEDIUM] No Specification of What Happens When gas_limit Is 0

Can a frame have gas_limit = 0? The constraints do not forbid it. A frame with gas_limit = 0 would immediately run out of gas when any opcode is executed. For VERIFY frames, this would mean the transaction is always invalid (APPROVE never called). For SENDER frames, the call would fail. For DEFAULT frames, the call would fail. Should gas_limit = 0 be explicitly rejected?

### 3.24 [LOW] Missing blob_versioned_hashes from Signature Hash

The signature hash computation elides VERIFY frame data but includes blob_versioned_hashes. Since blob_versioned_hashes are part of the RLP-encoded transaction, they are covered by the signature hash. This is correct but worth noting: a sender signing a frame transaction commits to specific blobs. If the sender does not want blobs, the list must be empty per the constraint, so this is fine.

### 3.25 [LOW] Stack Ordering for TXPARAM Is Ambiguous

TXPARAM says "It takes two values from the stack, param and in2 (in this order)." But "in this order" is ambiguous -- does it mean param is on top and in2 is below, or param is popped first (from the top)? In EVM convention, the first listed stack element is typically the topmost. If param is `top - 0` and in2 is `top - 1`, this should be stated explicitly as done for APPROVE.

### 3.26 [LOW] What Counts as "Existing Contract" for CALL* in Validation?

The validation trace rules say `CALL*` may target "any existing contract or precompile." But at what point in execution is "existing" evaluated? If a deploy frame runs first and creates the sender's code, is the sender now "existing" for purposes of subsequent VERIFY frames? Presumably yes, since deploy runs before verify. But what about contracts that are created by the deploy frame as side effects (e.g., a factory that deploys multiple contracts)? Are those also "existing" for subsequent frames?

---

## 4. Usage Scenarios / Possibilities I Like

### 4.1 Passkey-Based Wallets Without Smart Account Deployment

With default code supporting P256 (signature type 0x1), hardware keys and passkeys can be used directly with EOAs. A user's phone creates a P256 key pair, derives an Ethereum address as `keccak(qx|qy)[12:]`, and immediately has a working account that can send frame transactions signed with the device's secure enclave. No contract deployment needed for the first transaction (though a smart account migration would be needed for key rotation).

### 4.2 Gas-Free Onboarding

A new user can be onboarded with zero ETH:
1. Paymaster deploys the user's smart account (deploy frame).
2. Smart account verifies signature (VERIFY frame).
3. Paymaster approves payment (VERIFY frame with APPROVE(0x2)).
4. User's intended action executes (SENDER frame).

The entire flow is a single atomic transaction. The user never needs to acquire ETH.

### 4.3 Atomic DeFi Workflows

Approve-then-swap, borrow-then-leverage, or any multi-step DeFi interaction can be expressed as an atomic batch of SENDER frames. If any step fails, the entire batch reverts. This eliminates the need for multicall contracts, router contracts, or flash loans for atomic composability.

### 4.4 Post-Quantum Migration Path

This is the stated motivation and it is compelling. EOAs can migrate to PQ-safe signature schemes by deploying a smart account with PQ verification logic. The transition is per-user and gradual -- no hard fork deadline for key migration.

### 4.5 Session Keys and Delegated Execution

A smart account could implement VERIFY logic that accepts signatures from multiple keys with different permissions. For example, a "session key" could authorize transactions up to a certain value or only to specific contracts, while the master key has full access. This is a natural extension of the VERIFY frame's programmability.

### 4.6 Social Recovery Without Intermediaries

Smart accounts can implement social recovery logic in their VERIFY frames -- e.g., requiring 3-of-5 guardian signatures to approve a recovery transaction. This is currently possible with smart contract wallets but frame transactions make it protocol-native.

---

## 5. Additional Possibilities Enabled That We're Not Thinking Of

### 5.1 Cross-Account Atomic Operations

While the EIP envisions single-sender transactions, the frame structure could theoretically support cross-account atomicity in future extensions. If multiple senders could each APPROVE in separate VERIFY frames, you could build atomic swaps without smart contract escrow. The current spec restricts this (sender_approved can only be set once, and only for tx.sender), but the architecture does not preclude a future extension.

### 5.2 Transaction-Level Intent Expression

The frame structure, combined with TXPARAM introspection, enables a new pattern: "intent frames." A SENDER frame could contain an intent (e.g., "swap X for at least Y") and a solver could fill the execution details in a DEFAULT frame. The SENDER frame could verify the outcome by checking post-state conditions. This is not possible today without off-chain infrastructure.

### 5.3 Programmable Fee Markets

Since any contract can be a paymaster, it is possible to build fee market abstractions where users bid in ERC-20 tokens, and the paymaster converts to ETH at the current market rate. This enables a user-facing fee market denominated in stablecoins -- a significant UX improvement. The canonical paymaster could be extended to support this with DEX integration in a post-op frame.

### 5.4 MEV-Aware Transactions

Smart accounts could use VERIFY logic that is MEV-aware -- for example, rejecting transactions that are being front-run by checking TXPARAM values or block parameters (though block params are banned in the mempool, they are allowed at execution time). This opens design space for MEV-protection at the account level.

### 5.5 Composable Transaction Building

The frame structure enables a new pattern where different parties contribute different frames to a single transaction:
- The user creates VERIFY + SENDER frames.
- A paymaster adds its VERIFY frame.
- A relayer adds a DEFAULT post-op frame.

Each party only needs to know about its own frames (plus the signature hash). This is a more modular transaction-building model than today's monolithic signed transaction.

### 5.6 Onchain Governance Actions as Frames

DAO governance proposals could be expressed as frame transactions where the VERIFY frame checks that a governance vote has passed (by reading DAO contract state) and the SENDER frames execute the proposal's actions atomically. This makes governance execution more transparent and auditable.

### 5.7 Account Abstraction for Rollup Bridges

Rollup bridges could use frame transactions to implement trustless bridge exits. The VERIFY frame could verify a Merkle proof from the rollup, the SENDER frame could release funds, and the DEFAULT frame could update the bridge state. All in a single atomic transaction.

### 5.8 Programmable Transaction Deadlines Without Relying on block.timestamp

Since TXPARAM gives access to the nonce and sig_hash, smart accounts could implement replay protection with embedded deadlines (encoded in the frame data and committed via the sig_hash) without using banned opcodes. The account's VERIFY logic would check the deadline against its own storage rather than block.timestamp.

---

## 6. Changes to Increase Optionality Further

### 6.1 Allow Gas Forwarding Between Frames (Optional)

Add an optional mode flag or frame field that allows unused gas from one frame to overflow into the next. This would be opt-in (to preserve the current isolation semantics by default) but would dramatically improve gas efficiency for multi-frame transactions where gas requirements are uncertain. Implementation: a "gas-forward" flag on a frame that causes its unused gas to be added to the next frame's gas_limit.

### 6.2 Add a FRAMESTATUS Opcode or Extend TXPARAM for Richer Status

Currently, TXPARAM(0x15) returns only 0 (failure) or 1 (success). Adding return data access from previous frames would enable richer inter-frame communication. For example, a post-op frame might need to know exactly how much gas a SENDER frame used (available via receipt, but not in-EVM). Consider adding a `gas_used` param to TXPARAM for completed frames.

### 6.3 Introduce an OPTIONAL Frame Mode

Add a mode or flag that marks a frame as "optional" -- if it reverts, execution continues with the next frame rather than the transaction being invalid (for VERIFY) or the state being discarded (for atomic batch). This would enable "best effort" patterns: try to do X, if it fails, do Y instead. The atomic batch mechanism handles all-or-nothing, but there is no mechanism for graceful degradation.

### 6.4 Support Multiple Senders (Future Extension)

The current design is single-sender. A future extension could allow multiple senders, each with their own VERIFY frame, to participate in a single transaction. This would enable protocol-native atomic cross-account operations. The architecture already supports this conceptually (multiple VERIFY frames with different targets), but the sender_approved flag is a singleton. Changing it to a set of approved senders would unlock this.

### 6.5 Explicit Chain of Custody for Frame Assembly

Add a mechanism for frames to commit to the full frame list (not just the sig_hash). Currently, the paymaster signs the sig_hash, which covers all non-VERIFY frame data. But the paymaster might want to commit to specific frame ordering or the presence/absence of specific frames. Consider adding a TXPARAM for the full frame list hash (before VERIFY elision).

### 6.6 Allow Conditional Frame Execution Based on Previous Frame Status

Extend the atomic batch concept with conditional execution: "execute this frame only if frame N succeeded/failed." This would enable try-catch-like patterns at the transaction level. For example: "try to swap on DEX A (frame 2); if it fails, swap on DEX B (frame 3)." Currently, you would need a smart contract router for this.

### 6.7 Add a MAX_FRAME_GAS_LIMIT to Prevent Single-Frame Dominance

While there is a MAX_FRAMES (1000), there is no maximum gas_limit per frame. A single frame could consume the entire block gas limit. Adding a per-frame gas cap would improve block packing efficiency and prevent a single frame transaction from monopolizing block resources.

### 6.8 Standardize Error Reporting for Failed APPROVE

When APPROVE reverts (e.g., insufficient balance, double approval), the error reason is not captured anywhere accessible. Adding standardized error codes to APPROVE failures would improve debugging and wallet UX. Currently, a failed APPROVE just reverts the frame, and the user sees "transaction invalid" with no explanation.

### 6.9 Consider Not Clearing Transient Storage Between Frames

As noted in section 2.3, clearing transient storage between frames breaks the mental model of "transaction-scoped" storage. Instead of clearing, consider making transient storage frame-scoped but allowing explicit opt-in to sharing via a new TSTORE_SHARED / TLOAD_SHARED opcode pair, or simply keeping transient storage continuous across frames. The security concern (a contract relying on TSTORE state from a different caller context) already exists today with reentrancy, and the EVM does not protect against it.

### 6.10 Explicitly Define the Canonical Paymaster in This EIP or a Companion EIP

The canonical paymaster is critical infrastructure that the mempool rules depend on. It should be fully specified, audited, and deployed as part of this EIP's rollout. Without it, the paymaster mempool rules are unimplementable and untestable. At minimum, reference a companion EIP that specifies it, with both EIPs required to ship together.

---

## Appendix: Summary of Findings by Severity

### Critical
- 3.19: Structural rules reference invalid APPROVE(0x0) and likely wrong scope values for self_verify
- 3.20: Nonce increment by paymaster needs careful analysis for replay attacks in builder contexts
- 3.1: TXPARAM param ID gap (0x09 to 0x10) is error-prone
- 3.2: TXPARAM 0x09 "can be zero" contradicts static constraint `len(frames) > 0`

### High
- 3.4: Bit numbering convention ambiguity for mode flags
- 3.5: APPROVE scope 0x2 "from the account" wording is ambiguous
- 3.6: Default code SENDER mode all-or-nothing revert is too strict
- 3.7: compute_sig_hash mutates the transaction object
- 3.22: VERIFY is STATICCALL-like but APPROVE modifies state -- exception not stated

### Medium
- 3.8: Interaction with EIP-7702 delegated code is unspecified
- 3.9: FRAMEDATACOPY gas cost formula not fully specified
- 3.10: Skipped frame status in atomic batch is undefined
- 3.11: APPROVE return data handling is unspecified
- 3.12: Paymaster reservation race condition between nodes
- 3.13: No error differentiation for "contract doesn't support frames" vs "validation rejected"
- 3.14: TXPARAMLOAD vs TXPARAM naming inconsistency
- 3.21: Behavior section numbering starts at 2
- 3.23: gas_limit = 0 is not explicitly forbidden

### Low
- 3.15: Address 0xaa / opcode 0xaa collision
- 3.16: Calldata cost only for frames, not header fields
- 3.17: No maximum size for frame.data
- 3.18: Rationale lists three items under "two reasons"
- 3.24: blob_versioned_hashes in sig hash (correctly included, just noting)
- 3.25: TXPARAM stack ordering ambiguity
- 3.26: Definition of "existing contract" during deploy + verify sequence

### Design Concerns
- 2.1: Overall complexity budget is very high
- 2.2: ORIGIN semantics change is dangerous
- 2.3: Transient storage clearing between frames is surprising and potentially breaking
- 2.4: Gas isolation between frames is inflexible
- 2.5: MAX_PENDING_TXS_USING_NON_CANONICAL_PAYMASTER = 1 is too restrictive
- 2.6: Canonical paymaster is not specified in this EIP
- 2.7: No value field forces ETH transfers through code

---

*End of review. Overall assessment: This is an ambitious and architecturally sound proposal that addresses a real need (PQ migration + native AA). The frame abstraction is the right primitive. However, the specification has several inconsistencies that would cause implementation divergence if not fixed before moving beyond Draft status. The most critical issues are the wrong APPROVE scope values in the structural rules (3.19), the unspecified VERIFY/STATICCALL/APPROVE interaction (3.22), and the missing canonical paymaster specification (2.6). I recommend addressing these before soliciting client implementations.*
