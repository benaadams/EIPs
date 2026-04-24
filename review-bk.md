# Review: EIP-8141 "Frame Transaction"

**Reviewer:** Protocol review (systems-level)
**Date:** 2026-04-12
**Status of EIP:** Draft
**Authors:** Vitalik Buterin, lightclient, Felix Lange, Yoav Weiss, Alex Forshtat, Dror Tirosh, Shahaf Nacson, Derek Chiang

---

## 1. What I Like

### 1.1 The Transaction Is Finally a Program, Not a Ceremony

Every prior account abstraction proposal treated the transaction as a sacred artifact that needed to be preserved and carefully extended. EIP-8141 just says: a transaction is an ordered list of execution frames with typed semantics. This is the right answer. It is the answer that should have been obvious five years ago, and the fact that it took this long tells you how much institutional momentum the "one sender, one signature, one call" model had.

The frame list as the unit of composition means you can express deploy-then-verify, verify-then-pay-then-execute, atomic multi-calls -- all within a single transaction type. No bundlers, no off-chain infrastructure, no mempool hacks. The transaction *is* the bundle.

### 1.2 VERIFY as STATICCALL With Data Elision

This is genuinely clever. VERIFY frames cannot write state (line 80: "The execution behaves the same as STATICCALL"), and their data is stripped from the signature hash (lines 83-84). These two properties together solve two hard problems simultaneously:

1. Validation cannot create side effects that produce mempool invalidation vectors.
2. The signature does not need to commit to its own bytes, eliminating the recursive signing problem cleanly.

The elision also leaves the door wide open for future signature aggregation. If you can replace individual VERIFY frame data with a batch proof later, the rest of the transaction structure does not need to change. That is good design -- not because they planned for aggregation (they clearly did), but because the mechanism that solves today's problem also happens to not block tomorrow's.

### 1.3 Scope Separation in APPROVE

The `scope` operand on APPROVE (lines 159-168) is one of the better pieces of protocol design in this EIP. Separating "I authorize execution on my behalf" (0x1) from "I will pay for this transaction" (0x2) from "both" (0x3) enables the entire paymaster ecosystem without any paymaster-specific protocol machinery. The paymaster is just an account that calls APPROVE(0x2). The sender is just an account that calls APPROVE(0x1). The protocol does not need to know or care about the economic relationship between them.

The constraint that sender_approved must be true before payer_approved can be set (line 190) is a subtle but important ordering invariant. It means the payer always knows the sender has committed before it commits funds. Good.

### 1.4 Default Code for EOAs

The default code mechanism (lines 322-415) is the single most important feature for adoption. Without it, this EIP would be a smart-account-only feature that 99% of users cannot use on day one. With it, every existing EOA can send frame transactions using their existing private key. The default code handles ECDSA verification, P256 verification, and even interprets SENDER mode data as an RLP-encoded call list for multicall support.

This is the correct engineering decision. Build the general mechanism, then make the common case work out of the box.

### 1.5 Intrinsic Cost Calculation Is Transparent

The gas accounting model (lines 426-449) is refreshingly simple. Total gas is intrinsic cost plus calldata cost plus the sum of per-frame gas limits. Each frame has its own gas budget. Unused gas from one frame does not spill into the next. The refund goes to whoever called APPROVE(0x2). This is easy to reason about, easy to implement, and hard to exploit. Compared to the gas accounting nightmares in ERC-4337 (where the bundler, the paymaster, and the account all have interlocking gas budgets with complex refund mechanics), this is a breath of clean air.

### 1.6 The Mempool Section Is Serious Work

Lines 452-659 are the mempool rules, and they are *dense*. This is not an afterthought. The validation prefix concept (shortest prefix of frames that sets payer_approved), the four recognized prefix patterns, the canonical vs non-canonical paymaster distinction, the banned opcode list, the per-paymaster balance reservation accounting -- this is the work of people who have spent years fighting DoS vectors in ERC-4337 bundler mempools and decided to do it right at the protocol level this time.

The decision to limit non-canonical paymasters to MAX_PENDING_TXS_USING_NON_CANONICAL_PAYMASTER=1 (line 462) is pragmatic. It gives users the escape hatch of using any contract as a paymaster while limiting the blast radius of a misbehaving one.

---

## 2. What I Don't Like

### 2.1 The Mode Encoding Is Unnecessarily Clever

The mode field is doing too many things. The lower 8 bits are the mode (DEFAULT/VERIFY/SENDER, lines 63-71). Bits 9-10 are approval scope (line 96). Bit 11 is atomic batching (line 97). This is a bit-packed integer that encodes three orthogonal concerns into a single field.

The spec calls this the "mode" but it is actually mode + constraints + grouping. The approval scope bits (9-10) constrain what APPROVE can do, which is a validation-time policy concern. The atomic batch bit is a grouping concern. Neither of these is a "mode" in any meaningful sense.

This will be a source of bugs. Implementers will forget that mode 0x0402 means "SENDER with atomic batch" while mode 0x0102 means "SENDER with scope restricted to execution-only." The spec already has to explain this in multiple places with bit-shift arithmetic (lines 109, 113-117, 173-176). That is a smell.

A cleaner encoding would be separate fields: `mode`, `scope`, `atomic_batch`. Three bytes instead of one packed integer. The calldata cost difference is negligible. The implementation clarity difference is significant.

### 2.2 The Specification Mixes Normative and Informational Content

The spec has a habit of burying critical behavioral requirements in "Notes" sections. For example, line 318:

> It is implied by the handling that the sender must approve the transaction *before* the payer...

This is not "implied." This is a fundamental ordering invariant enforced by the check at line 190 ("If sender_approved == false, revert the frame"). It should be stated as a normative rule, not as an implication that the reader is expected to derive.

Similarly, the Python code for default_code (lines 354-415) is listed as an illustration, but it IS the specification. The prose description above it (lines 324-345) is less precise than the code. If I am implementing this, which do I follow when they disagree?

### 2.3 Four New Opcodes Is a Lot

APPROVE, TXPARAM, FRAMEDATALOAD, FRAMEDATACOPY -- four new opcodes for one transaction type. FRAMEDATALOAD and FRAMEDATACOPY are essentially CALLDATALOAD and CALLDATACOPY with a frame index parameter. They exist because frames need to read each other's data, but the spec already elides VERIFY frame data from introspection (line 246: "When targeting a frame in VERIFY mode, the returned data is always zero"). So you have new opcodes that work like existing opcodes except when they don't.

Could TXPARAM have subsumed FRAMEDATALOAD/FRAMEDATACOPY? The data access pattern (offset into a specific frame's data) is more complex than a single stack return value, but a TXPARAM that returns a memory pointer to a frame's data could have avoided two opcodes. The opcode space is finite, and each new opcode is a permanent tax on every EVM implementation forever.

### 2.4 The Sender Field in the Transaction Is Unusual

The transaction explicitly includes `sender` as a field (line 51). In every other transaction type, the sender is derived from the signature. Here, the sender is declared and then validated by whatever VERIFY frame the sender chooses. This means the sender address is part of the unsigned transaction data.

This is a necessary consequence of supporting arbitrary signature schemes, but the spec does not adequately discuss the implications. For example: the sender address is included in the signature hash (via the full transaction RLP). But the sender address is also the thing that determines *which* verification logic runs. A malicious node could in principle try different sender values to see if any account's verification logic accepts a given signature blob. The default code checks `frame.target != tx.sender` (line 332) which prevents this for the default case, but custom smart accounts need to be very careful here.

### 2.5 ORIGIN Semantics Change Is Under-Discussed

Line 287 states that "The ORIGIN opcode returns frame caller throughout all call depths." This is a breaking semantic change. ORIGIN has historically meant "the EOA that signed this transaction." Now it means "the caller address of the current frame." For DEFAULT and VERIFY frames, that is ENTRY_POINT (address 0xaa). For SENDER frames, that is tx.sender.

The Backwards Compatibility section (lines 858-861) acknowledges this but waves it away by saying "consistent with the precedent set by EIP-7702." That precedent does not make the change safe. Any contract that uses `tx.origin == msg.sender` as an EOA check (a common if discouraged pattern) will behave differently when called from a SENDER frame, because ORIGIN will be the sender address but the call chain may include intermediary contracts.

More importantly, ORIGIN changes *between frames within the same transaction*. A DEFAULT frame sees ORIGIN as ENTRY_POINT. A SENDER frame sees ORIGIN as tx.sender. If a contract is called from both frame types in the same transaction (which atomic batching makes plausible), it will see different ORIGIN values. This is a new footgun.

---

## 3. Issues

### 3.1 The Nonce Increment Timing Is Ambiguous

The nonce is checked against state at validation time (line 269: "Ensure tx.nonce == state[tx.sender].nonce"). The nonce is *incremented* inside APPROVE when scope includes payment (lines 187, 191). But what happens if the VERIFY frame that calls APPROVE(0x2) reverts after incrementing the nonce? Line 185 says "If sender_approved was already set, revert the frame." But the nonce increment happens *as part of* APPROVE(0x2).

The spec says (line 318) that once sender_approved or payer_approved become true "they cannot be re-approved or reverted." But APPROVE is defined to revert the frame on certain conditions (e.g., insufficient balance, line 189). If APPROVE(0x2) partially executes -- incrementing the nonce and debiting balance -- and then the debit fails because of insufficient balance, is the nonce increment also reverted? The spec implies yes (the whole frame reverts), but this needs to be stated explicitly.

The real danger: if the nonce increment can be reverted independently of payer_approved, a sender could replay transactions.

### 3.2 APPROVE Can Only Be Called by frame.target, but frame.target Can DELEGATECALL

Line 149 says APPROVE reverts if `ADDRESS != frame.target`. ADDRESS returns the current code's address. But if frame.target DELEGATECALLs into a library that calls APPROVE, ADDRESS will still be frame.target (because DELEGATECALL preserves the calling context). This means APPROVE CAN be called from delegated code, which is probably intended, but should be explicitly stated.

Conversely, if frame.target CALLs another contract that tries to call APPROVE, it will fail because ADDRESS will be the called contract's address, not frame.target. Good -- this prevents delegation of approval authority via regular calls. But the spec does not make this distinction clear.

### 3.3 Atomic Batching Revert Semantics Need Tightening

Lines 291-313 define atomic batching. When a frame in a batch reverts:
1. Restore state to the snapshot before the batch.
2. Mark remaining frames as skipped.

But what about gas? If frame 0 in a batch uses 50k gas and frame 1 reverts, the state is restored, but is the 50k gas still consumed? The spec says "unused gas from a frame is not available to subsequent frames" (line 443) but does not clarify whether gas consumed by reverted frames in an atomic batch is refunded or burned.

For consistency with how the EVM handles reverted calls (gas is consumed, state is reverted), the gas should be consumed. But this means an atomic batch of 5 frames where the 5th reverts burns all gas from frames 1-5 while producing no state changes. This is correct behavior but users need to understand it.

Also: what is the `status` (TXPARAM 0x15) of a frame that was reverted due to atomic batching? It did not revert itself -- it succeeded but was rolled back because a later frame in its batch reverted. Is that status 0 (failure) or 1 (success)? The receipt should reflect the actual outcome (failure), but this needs clarification.

### 3.4 The `sender` Field Creates an Ambiguity for CREATE Transactions

The constraints section (line 110) says `len(tx.frames[n].target) == 20 or tx.frames[n].target is None`. A null target with DEFAULT mode creates what? The behavior section (line 279) says "If target is null, set the call target to tx.sender." But the default mode sets caller to ENTRY_POINT and calls the target. If the target is null and resolves to tx.sender, this is a DEFAULT call to the sender's code with ENTRY_POINT as caller. This is probably fine, but the interaction between null targets and the three modes needs a clearer truth table.

What about contract creation? Traditional transactions use a null `to` field for deployment. Here, null means "sender." If I want to deploy a contract from a SENDER frame, I apparently cannot use the null-target convention. The default code for SENDER mode (lines 339-343) uses RLP-encoded call lists with explicit targets, which sidesteps this, but the protocol-level null semantics should be unambiguous.

### 3.5 Signature Hash Computation Cost

TXPARAM(0x08) returns compute_sig_hash(tx) (line 214). This requires hashing the entire transaction with VERIFY frame data elided. For a transaction with large frames (each frame can have up to ~128KB of calldata), this hash computation could be significant. The TXPARAM opcode has a gas cost of 2 (line 199), which is wildly insufficient for a keccak hash over potentially hundreds of kilobytes.

Either the signature hash needs to be computed once at transaction processing time and cached (which the spec does not mandate), or the gas cost of TXPARAM(0x08) needs to reflect the actual computation cost. At 6 gas per word for keccak256 (per the SHA3 opcode pricing), a 128KB transaction would cost ~24,000 gas just for the hash. Charging 2 gas is a mispricing.

### 3.6 TXPARAM 0x09 Comment Says "Can Be Zero" -- But Constraints Say Otherwise

Line 215: `0x09` returns `len(frames)` with the comment "(can be zero)." But the constraints at line 107 assert `len(tx.frames) > 0`. These contradict each other. If frames must be non-empty, then TXPARAM(0x09) can never return zero for a valid frame transaction. The comment should be removed.

### 3.7 Transient Storage Reset Between Frames Is a Footgun

Line 423: "Discard the TSTORE and TLOAD transient storage between frames." This means transient storage is frame-scoped, not transaction-scoped. This is a significant departure from EIP-1153, which defines transient storage as transaction-scoped.

This has real consequences. If a contract uses transient storage for reentrancy guards (the primary use case of EIP-1153), those guards reset between frames. A SENDER frame could enter a contract, set a reentrancy guard via TSTORE, complete, and then a subsequent frame could enter the same contract and the guard would be gone. This is not a vulnerability in the frame transaction itself, but it changes the security properties of transient storage in a way that existing contracts did not anticipate.

The rationale does not explain this choice. There should be a rationale entry explaining why per-frame transient storage is necessary and what the alternatives were.

### 3.8 The Mempool "Only Verify" Approval Scope Is Wrong

Line 547 says `only_verify` must call `APPROVE(0x0)`. But the APPROVE spec (lines 161-162) says scope 0x0 is not a valid value -- "Any other value results in an exceptional halt." The valid scopes are 0x1, 0x2, and 0x3. There is no scope that means "approve sender only without payment."

Wait -- 0x1 is "Approval of execution." So `only_verify` should call `APPROVE(0x1)`, not `APPROVE(0x0)`. This appears to be a bug in the mempool section. The mode subclassification table (line 490) says `only_verify` "approves only the sender," which maps to scope 0x1. But the structural rule (line 547) says `APPROVE(0x0)`. This is a specification error.

### 3.9 No Discussion of EIP-7702 Interaction

EIP-7702 allows EOAs to delegate to a contract. EIP-8141 has default code for EOAs without code. What happens when an EOA has an EIP-7702 delegation?

Line 285 says "If frame.target has no code, execute the logic described in default code." An EOA with a 7702 delegation *has code* (the delegated code). So the default code would not execute, and the delegated code would run in VERIFY mode. This means the delegated code needs to handle APPROVE correctly. Does the delegated code even know about APPROVE? If it was written before EIP-8141, it does not have the APPROVE opcode in its instruction set.

The validation trace rules (line 565) ban `CALL*` or `EXTCODE*` to "an address that uses an EIP-7702 delegation, except for tx.sender default-code behavior." So in the validation prefix, you cannot call into delegated accounts. But what if tx.sender itself has a 7702 delegation? Is the sender's delegated code used, or is the default code used? The exception "for tx.sender default-code behavior" suggests default code, but only when the target has no code. A 7702 delegation gives the account code.

This interaction is under-specified and critical for the transition period where many accounts will have 7702 delegations.

---

## 4. Usage Scenarios I Like

### 4.1 Day-One Post-Quantum Migration Path

An EOA holder can, today (post-activation), send a frame transaction with a P256 VERIFY frame. No contract deployment, no migration ceremony, no 7702 delegation. Just a different signature type byte in the VERIFY frame data. When a PQ-secure signature scheme is added to the default code (or when the user deploys a smart account with one), the transition is seamless. The frame structure does not change, only the VERIFY frame's contents.

This is the most credible PQ migration story any EIP has offered.

### 4.2 Gas Sponsorship Without Off-Chain Infrastructure

Example 3 (lines 764-778) shows the full sponsorship flow. The sponsor validates in a VERIFY frame, pays via APPROVE(0x2), and the sender pays the sponsor in ERC-20 tokens in a SENDER frame. No bundlers. No relayers. No off-chain matching. The transaction itself encodes the entire economic agreement. This kills a massive chunk of the ERC-4337 infrastructure stack.

### 4.3 Atomic Approve-and-Swap

Example 2 (lines 754-762) is the use case every DeFi user has been burned by. You approve a token, the swap fails, and now you have a dangling approval. With atomic batching, the approval and swap either both succeed or neither does. This is not just a convenience -- it eliminates an entire class of approval-griefing attacks.

### 4.4 Smart Account Deployment in the First Transaction

Example 1b (lines 742-753) shows deploying a smart account and making the first transaction in a single atomic operation. No prefunding. No separate deployment transaction. The first transaction IS the deployment. This is how account creation should have always worked.

---

## 5. What Additional Things Become Enabled

### 5.1 Programmable Transaction Receipts

Each frame has its own status, gas_used, and logs in the receipt (lines 122-129). This means a single transaction can produce a structured, per-operation receipt. Indexers and block explorers can parse individual operations within a frame transaction. This is a huge improvement over the current model where a multicall contract emits a blob of logs with no structural separation.

DeFi aggregators, portfolio trackers, and tax software can parse frame receipts to understand what a transaction actually did, operation by operation.

### 5.2 Cross-Frame State Observation as a Coordination Primitive

TXPARAM(0x15) lets a frame read the status of a previous frame (line 221). A later frame can condition its behavior on whether an earlier frame succeeded or failed. Combined with atomic batching, this enables *conditional execution within a single transaction*.

Think about this: frame 0 tries a swap on DEX A. Frame 1 checks if frame 0 succeeded. If not, frame 1 tries DEX B. This is in-transaction routing without a router contract. The transaction itself is the execution plan, and later frames can adapt based on earlier results. No smart contract needed for the decision logic.

### 5.3 Session Keys and Temporary Authorization Patterns

A smart account can implement VERIFY logic that checks for time-limited or scope-limited session keys. The account code can verify that a session key signed the transaction hash, check the session key's permissions (e.g., "only transfers to whitelisted addresses, max 1 ETH per day"), and APPROVE(0x1). The paymaster logic is entirely separate.

This is not just account abstraction. This is programmable authorization at the transaction level. Gaming, subscription services, and automated DeFi strategies all benefit.

### 5.4 MEV Protection via Frame Ordering Commitments

A smart account's VERIFY logic can inspect the full frame list via TXPARAM and enforce ordering invariants. For example: "I will only approve this transaction if frame 2 targets Uniswap and frame 3 targets my account." This lets the sender commit to a specific execution plan at signature time, preventing block builders from reordering or inserting frames.

This is not a complete MEV solution, but it gives users a protocol-native tool for expressing execution constraints that was previously only available through specialized MEV protection services.

### 5.5 Multi-Signer Transactions

Nothing prevents multiple VERIFY frames targeting different accounts. Account A's VERIFY frame calls APPROVE(0x1). Account B's VERIFY frame calls APPROVE(0x2). The SENDER frames then execute on A's behalf, paid for by B. This is a native multi-signer transaction pattern.

Extend this further: a smart account's VERIFY logic could require M-of-N signatures, where each signer provides a separate VERIFY frame. The account aggregates the approvals internally. Multi-sig transactions without a multi-sig contract.

### 5.6 Trustless Relay Networks

The separation of sender approval from payer approval means a relay network can accept signed frame transactions from users, attach its own VERIFY frame for payment, and submit the combined transaction. The relay pays gas and gets compensated in the SENDER frames. No trust required -- the sender's signature commits to the frame structure (minus VERIFY data), so the relay cannot modify the intended operations.

This turns gas payment into a competitive market at the transaction level, without bundler infrastructure.

### 5.7 Intent-Based Execution with Verifiable Outcomes

A VERIFY frame can inspect the entire frame list via TXPARAM. A solver network could construct the frame list to fulfill a user's intent, and the user's VERIFY logic could verify the outcome matches the intent before approving. "I want to end up with at least X tokens of Y" becomes a verifiable condition checked by the sender's own code before APPROVE is called.

This is native, protocol-level intent execution. No separate intent mempool, no trust assumptions about solvers.

---

## 6. Changes to Increase Optionality

### 6.1 Separate the Mode Encoding

Replace the bit-packed `mode` integer with separate fields: `mode` (uint8), `scope` (uint8), `flags` (uint8). The RLP encoding cost is 2 extra bytes per frame. The clarity gain is enormous. Every implementer who has to write `(frame.mode >> 8) & 3` to extract the approval scope will thank you.

The current encoding saves a few bytes at the cost of specification clarity, implementation safety, and extensibility. If you need more flags later, you are already out of bits in a uint16. With a separate flags field, you have 8 bits of expansion room.

### 6.2 Specify the Signature Hash Gas Cost

TXPARAM(0x08) should have a gas cost formula based on the transaction size, not a flat 2 gas. Something like `2 + 6 * ceil(tx_rlp_size / 32)` to match SHA3 pricing. Alternatively, mandate that implementations cache the signature hash and charge 2 gas for retrieval, but state this explicitly. The current spec will lead to either underpriced computation or inconsistent caching behavior across clients.

### 6.3 Add a TXPARAM for Blob Versioned Hash Access

TXPARAM(0x07) returns the count of blob versioned hashes but there is no TXPARAM to access individual hashes. For a transaction type that aims to be general-purpose, this is an odd omission. Add a `0x18` param that takes a blob index in `in2` and returns the versioned hash. Without this, any frame that needs to reference blob data must use the existing BLOBHASH opcode, which is banned in the validation prefix (line 577).

### 6.4 Clarify Transient Storage Semantics with a Rationale

Add a rationale section for the per-frame transient storage reset. If the reason is to prevent information leakage between VERIFY frames and SENDER frames, say so. If the reason is to prevent cross-frame reentrancy guard bypass, say so. The current spec just states the behavior with no justification. Implementers and auditors need to understand the *why* to correctly reason about edge cases.

Also consider: would a mode flag that preserves transient storage across frames in the same atomic batch be useful? If two frames in an atomic batch are meant to be a single logical operation, having shared transient storage might be the correct semantics.

### 6.5 Define the EIP-7702 Interaction Explicitly

Add a section that specifies exactly what happens when tx.sender has an EIP-7702 delegation. The two plausible answers are:

1. The delegation code runs (consistent with 7702 semantics), and it must handle APPROVE.
2. The default code runs regardless of delegations (special case for frame transactions).

Option 1 is more consistent but creates a chicken-and-egg problem for accounts that set up 7702 delegations before EIP-8141 existed. Option 2 is cleaner for the transition but creates a precedent for frame transactions overriding delegation semantics.

Either answer is defensible. Not specifying the answer is not.

### 6.6 Add a Frame-Level Value Field

The rationale (line 699-701) says "No value in frame" because "the account code can send value." This is true but it makes simple ETH transfers more expensive and more complex than they need to be. A basic ETH transfer requires a SENDER frame with RLP-encoded call data (in the default code path) or a contract that parses calldata and sends value. A frame-level value field would make the most common transaction type (sending ETH) as simple as it is today.

The data efficiency section (lines 796-821) shows a "Simple ETH transfer" requiring 134 bytes. A current EIP-1559 ETH transfer is ~112 bytes. The 22-byte overhead is partly because the value must be encoded in the frame's data instead of being a first-class field. Adding a `value` field to the frame structure closes most of this gap.

The counterargument is "keep frames simple." Fair. But ETH transfers are 40%+ of all transactions. Making the most common operation pay an encoding tax to keep the frame structure minimal is a tradeoff worth questioning.

### 6.7 Consider a MAX_FRAME_DATA_SIZE Limit

The spec limits the number of frames (MAX_FRAMES = 1000, line 33) but does not limit the size of individual frame data. A transaction with 1000 frames, each with 128KB of data, would be ~128MB. The calldata gas cost provides an economic limit, but the protocol should also enforce a structural limit to prevent pathological transactions from consuming excessive memory during parsing and validation.

### 6.8 Make the Param ID Space in TXPARAM Less Sparse

The TXPARAM param IDs jump from 0x09 to 0x10 (lines 215-216). This wastes opcode address space and suggests the encoding was designed with some future expansion in mind. If expansion is the goal, use a more structured approach (e.g., 0x00-0x0F for transaction-level params, 0x10-0x1F for frame-level params). The current assignment is *almost* this but not quite -- 0x09 (frame count) is in the transaction-level range, and 0x10 (current frame index) could be in either. Tighten the allocation scheme now while it is free to change.

---

## Summary

EIP-8141 is the most serious account abstraction proposal Ethereum has produced. It is not perfect -- the mode encoding is too clever, the 7702 interaction is unspecified, some gas costs are mispriced, and there are specification bugs (the APPROVE(0x0) issue in the mempool section). But the core design is sound. The frame abstraction is the right primitive. VERIFY-as-STATICCALL with data elision is elegant. The APPROVE opcode with scope separation is well-designed. The default code for EOAs is the correct migration strategy. The mempool rules are thorough and battle-informed.

The authors clearly learned from years of ERC-4337 operational experience and from the partial solutions of EIP-3074 and EIP-7702. This EIP does not try to be clever. It tries to be correct and general. That is the right instinct.

Fix the bugs, clarify the ambiguities, separate the mode encoding, specify the 7702 interaction, and price TXPARAM(0x08) correctly. The rest is solid.
