# Review of EIP-8141: Frame Transaction

**Reviewer:** Hideo Kojima Analytical Framework
**Date:** 2026-04-12
**Status:** Draft Review

---

## I. What I Like About It

### The Transaction Becomes a Score, Not a Single Note

The single most important thing EIP-8141 does is redefine the transaction from an atomic imperative -- "do this one thing, signed this one way" -- into a composed sequence of frames with distinct roles. Verification is separated from execution. Payment is separated from authorization. This is not incremental improvement. This is a structural reconception of what a transaction IS.

Prior to this, Ethereum transactions were monolithic. One signer, one action, one payer. The entire account abstraction effort (ERC-4337, EIP-7702) has been working around this monolith rather than replacing it. EIP-8141 replaces it. The frame model says: a transaction is a program with phases, and the protocol should understand those phases natively.

### APPROVE as a Protocol-Level Consent Mechanism

The APPROVE opcode is the design's most elegant element. It is not merely a return value. It is a transaction-scoped state mutation that says: "I, the executing contract, consent to this role." The separation of sender_approved and payer_approved into independent flags, each settable exactly once, creates a protocol-level model of consent that cannot be circumvented by clever calldata construction.

The constraint that sender must approve before payer is particularly well-considered. It encodes a trust hierarchy directly into the execution model: you cannot pay for something the owner has not authorized. This is not just security engineering. It is a statement about what authorization means in this system.

### Signature Hash Elision is Forward-Looking

Eliding VERIFY frame data from the signature hash is a decision that trades immediate simplicity for long-term optionality. The specification correctly identifies that this enables future signature aggregation. More importantly, it enables the sponsor's data to be added after the sender signs, which is the mechanical prerequisite for a functional gas sponsorship market. The sender commits to WHO can sponsor (via frame.target in the sig hash) but not to WHAT the sponsor will provide. This is a well-drawn trust boundary.

### Default Code Preserves the Installed Base

The default code for EOAs is not glamorous, but it is essential. It means every existing Ethereum user can use frame transactions immediately, with ECDSA signatures they already have, while gaining access to gas abstraction and atomic batching. The inclusion of P256 in the default code is a pragmatic concession to the mobile hardware ecosystem. This is the kind of boring correctness that determines whether a protocol change actually gets adopted.

### Atomic Batching Solves a Real Problem Cleanly

The atomic batch flag on SENDER frames is a minimal mechanism that solves the approve-then-swap problem without introducing a new frame mode. The bit-flag design on consecutive SENDER frames is economical: no new structural complexity, just a single bit that says "I am part of a group." The rollback semantics (snapshot before batch, restore on any revert) are clean and unsurprising. This will prevent a category of stuck-state bugs that have plagued smart account users.

### Warm/Cold State Sharing Across Frames

Sharing the warm/cold access journal across frames while discarding transient storage between them is a precise decision. It means the gas cost of a multi-frame transaction reflects reality (you do not re-pay for state you already touched) while maintaining frame isolation for transient data. This distinction shows careful thinking about what should persist across frame boundaries and what should not.

---

## II. What I Do Not Like About It

### The Sender Field in the Transaction Envelope is a Conceptual Regression

Traditional Ethereum transactions derive the sender from the signature. The sender is not declared; it is proven. EIP-8141 reverses this: the sender is declared in the envelope, and then a VERIFY frame checks whether that declaration is legitimate. This means the protocol now accepts a transaction that CLAIMS to be from an address and then runs code to see if the claim is true.

This is necessary for the design to work -- you need to know the sender before you can run their verification code -- but it creates a subtle inversion. The sender field is now an assertion, not a fact. The protocol trusts the VERIFY frame to validate this assertion, but the trust model is different from what Ethereum has today. I would like the specification to be more explicit about this inversion and its security implications, particularly for indexing services and block explorers that may assume sender means "proven sender."

### Gas Isolation Between Frames is a Double-Edged Sword

Each frame has its own gas_limit and unused gas from one frame cannot flow to the next. This is clean from an isolation perspective but creates a gas estimation problem. Users (or their wallet software) must estimate gas for each frame independently. If the verification frame uses 50,000 gas and the execution frame uses 200,000, the user cannot simply say "250,000 total." They must correctly partition gas across frames.

In practice, this means every frame will be over-provisioned for safety, and the aggregate over-provisioning across 3-5 frames will be worse than the over-provisioning of a single transaction today. The refund mechanism mitigates the cost but does not eliminate the UX complexity. This will be a source of user-facing errors, especially in sponsored transaction flows where the sponsor and the sender are estimating gas independently.

### MAX_FRAMES = 1000 is Excessively Generous

One thousand frames per transaction is a theoretical maximum that no legitimate use case in the specification requires. The examples use 2-5 frames. Even an ambitious batch transaction should not need more than 20-30. A limit of 1000 invites abuse: constructing transactions that force validators to process long frame sequences, each with their own execution context setup and teardown. The intrinsic cost of 15,000 gas does not scale with frame count, so a 1000-frame transaction pays the same intrinsic cost as a 2-frame transaction.

This should either be reduced significantly (to perhaps 64 or 128) or the intrinsic cost should include a per-frame component.

### The ORIGIN Semantic Change is More Disruptive Than Acknowledged

The specification notes that ORIGIN now returns the frame's caller rather than the transaction origin, consistent with EIP-7702. But EIP-7702 changed ORIGIN in a narrow context (delegated EOAs). EIP-8141 changes it for an entirely new transaction type. Any contract that uses tx.origin for any purpose -- and there are many, despite the pattern being discouraged -- will behave differently when called via a frame transaction. The backwards compatibility section acknowledges this in one sentence. It deserves more analysis, particularly an enumeration of known contracts on mainnet that use tx.origin in security-relevant ways.

### The Mempool Section Feels Like Two Documents Stitched Together

The specification oscillates between defining protocol-level behavior (opcodes, execution semantics, gas accounting) and defining mempool policy (validation prefixes, banned opcodes, paymaster accounting). These are different concerns operating at different layers. The mempool policy is necessarily more opinionated and less permanent than the protocol specification. Combining them creates ambiguity: when a client implementer reads "must reject," do they mean "the transaction is invalid per the protocol" or "the transaction should not be propagated per mempool policy"?

The four recognized validation prefixes in particular are policy choices that may need to evolve. Encoding them in the EIP rather than in a separate mempool policy document means changing them requires amending the core specification.

---

## III. Issues I See

### 1. Scope Operand Constraints via Mode Bits Are Under-Specified in the Execution Section

The APPROVE opcode section defines scope constraints based on bits 9-10 of frame.mode, but the main execution behavior section does not explicitly describe how these bits are validated during frame execution. The constraint logic appears only in the APPROVE opcode definition. A reader trying to understand the full execution flow must cross-reference two sections to understand that the mode bits constrain what APPROVE can do. This should be unified or at least cross-referenced explicitly.

### 2. The Nonce Increment Timing Creates an Asymmetry

The nonce is incremented inside APPROVE (scope 0x2 or 0x3), which means it happens during a VERIFY frame. But VERIFY frames are static -- they cannot modify state. The specification makes an exception for APPROVE's nonce increment and balance deduction, but this exception is implicit. A VERIFY frame that calls APPROVE performs state mutations (nonce increment, balance deduction) despite executing under STATICCALL semantics. The specification should explicitly state that APPROVE is the sole exception to VERIFY's static execution guarantee, and explain why this exception is safe.

### 3. Signature Type Extensibility in Default Code is Limited

The default code supports exactly two signature types: secp256k1 (0x0) and P256 (0x1). Any other value reverts. There is no mechanism for registering additional signature types for the default code path. When post-quantum signature schemes become available, EOA users will need to either deploy custom account code or wait for a protocol upgrade that adds new signature types to the default code. This somewhat undermines the "native off-ramp to PQ-secure systems" motivation, since the off-ramp only works for accounts with custom code.

Consider reserving a signature type range (e.g., 0x80-0xFF) for future protocol-defined schemes, or specifying a precompile-based extension mechanism.

### 4. Atomic Batch Rollback Interacts Poorly with Gas Accounting

When an atomic batch is rolled back, the state is restored to the pre-batch snapshot. But gas has already been consumed by the rolled-back frames. The specification says unused gas is refunded to the payer after all frames execute, but it does not clarify whether gas consumed by rolled-back frames counts as "used" or "unused." If rolled-back frames still consume gas (which seems likely, since computation was performed), then the payer pays for work that was undone. This is probably correct but should be stated explicitly.

### 5. ENTRY_POINT Address Collision Risk

The ENTRY_POINT is defined as address(0xaa). This is a low-numbered address in the precompile range. If any existing contract or precompile uses this address, there will be a collision. The specification should state whether address(0xaa) is currently unoccupied and whether it is being reserved by this EIP. Additionally, since ENTRY_POINT is used as the caller for DEFAULT and VERIFY frames, any contract that checks msg.sender == address(0xaa) for access control will now have a new caller that can trigger those checks.

### 6. TXPARAM Index Gap Between 0x09 and 0x10

The TXPARAM opcode uses param values 0x00-0x09 for transaction-level fields and 0x10-0x17 for frame-level fields. There is a gap from 0x0A to 0x0F. This is presumably intentional (reserving space for future transaction-level fields), but the gap should be documented. Invalid param values in this range cause an exceptional halt, which is correct but could surprise developers who expect contiguous numbering.

### 7. No Mechanism to Inspect Blob Data from Frames

The specification includes blob_versioned_hashes in the transaction envelope and provides len(blob_versioned_hashes) via TXPARAM, but there is no TXPARAM value that returns individual blob hashes. A frame that needs to verify blob content (e.g., a verification frame for a blob-carrying frame transaction) cannot access the specific hashes. Either add a TXPARAM entry for individual blob hash access or document why this was intentionally omitted.

### 8. The "Known Deployer" Requirement is Not Defined

The mempool section requires that deployment frames use a "known deterministic deployer" but does not define what "known" means in protocol terms. Is there a list? Is it defined by each client implementation? Is it consensus-critical? The reference to EIP-7997 in the examples suggests a specific deployer, but the specification does not commit to one. This ambiguity will lead to divergent mempool behavior across client implementations.

---

## IV. Usage Scenarios and Possibilities I Like

### Gas Abstraction Without Middleware

The most immediately impactful use case is the elimination of the bundler/relayer layer that ERC-4337 requires. A user with a smart account can construct a frame transaction that verifies their signature (VERIFY frame), pays for gas from their own balance or via a sponsor (APPROVE with appropriate scope), and executes their intended action (SENDER frame) -- all in a single native transaction that goes directly into the mempool. No bundlers, no off-chain infrastructure, no second-class transaction status. This is account abstraction that is actually native.

### ERC-20 Gas Payment as a First-Class Flow

Example 4 in the specification -- an EOA paying gas in ERC-20 tokens -- is the use case that will drive mainstream adoption. A user holding only USDC can transact on Ethereum without ever acquiring ETH. The sponsor provides the ETH for gas and receives USDC in return, within the same atomic transaction. This has been possible via relayers, but making it native removes the trust assumption on the relayer and the latency of the relay path.

### Smart Account Deployment on First Use

The deploy-then-verify-then-execute pattern (Example 1b) means a smart account can be created at a predetermined address and used in the same transaction. The user never needs to fund an EOA, deploy a contract in a separate transaction, and then use the contract. The entire lifecycle from "I have an address" to "I am transacting" collapses into a single transaction. This is particularly powerful for onboarding flows where a new user receives tokens to a counterfactual address and activates their account in their first transaction.

### Atomic DeFi Compositions

The atomic batch flag enables approve-and-swap, borrow-and-deposit, and other multi-step DeFi operations to execute atomically from the user's perspective. This is not new (flash loans and multicall contracts provide similar atomicity) but doing it at the transaction level means it works with any contract, not just contracts that support multicall. The user's wallet can compose arbitrary sequences of contract calls with all-or-nothing semantics.

---

## V. Additional Possibilities You Might Not Be Thinking Of

### Cross-Frame Condition Checking via TXPARAM Status

TXPARAM(0x15, frameIndex) returns the status of a previously executed frame. This means a later frame can branch on whether an earlier frame succeeded or failed. Combined with the fact that non-VERIFY frame reverts do not invalidate the transaction, this enables conditional execution patterns: "Do X, and if X succeeds, do Y; otherwise do Z." This is a primitive form of transaction-level control flow that does not exist in Ethereum today.

The implications are significant. A sponsor's post-op frame (DEFAULT mode) can check whether the user's execution frame succeeded and adjust its behavior accordingly -- for example, skipping a token conversion if the user's call reverted. This moves error handling from the contract level to the transaction level.

### Programmable Transaction Policies via VERIFY Frames

VERIFY frames can read the entire transaction structure via TXPARAM and FRAMEDATALOAD. This means a smart account's verification code can enforce arbitrary policies on the transaction, not just signature validity. For example:

- A multisig account can require that the total value transferred across all SENDER frames does not exceed a daily limit stored in account storage.
- A corporate account can require that execution frames target only whitelisted contracts.
- A time-locked account can require that certain operations are only permitted after a delay period (checked via the signature hash committing to a future nonce).

This transforms the VERIFY frame from a signature checker into a programmable transaction firewall. The account's code becomes a policy engine that gates what the account can do, not just who can make it do things.

### Social Recovery and Key Rotation Without External Contracts

A smart account can implement key rotation entirely within its own code, using VERIFY frames to check the new key against an on-chain registry stored in its own storage. More importantly, social recovery schemes -- where N-of-M guardians can authorize a key change -- can be implemented as account code that validates guardian signatures in a VERIFY frame. Today this requires external contracts (Safe modules, social recovery modules). With EIP-8141, the account IS the recovery system.

### MEV Protection via Sponsor Commitment

In the sponsored transaction flow, the sponsor commits to paying for gas in a VERIFY frame that runs before the user's execution frames. The sponsor can inspect the user's execution frames via FRAMEDATALOAD and TXPARAM before approving payment. This means the sponsor can implement MEV-protection policies: refusing to pay for transactions that interact with known sandwich-vulnerable pools, or requiring that execution frames include slippage protection. The sponsor becomes a programmable gatekeeper for MEV exposure.

### Intent-Like Execution Without New Infrastructure

The frame model is structurally similar to intent-based execution: the user expresses what they want (in SENDER frames), and a filler/sponsor provides the execution context (in VERIFY and DEFAULT frames). The difference is that this happens within the existing transaction model and mempool, without requiring new intent-specific infrastructure. A wallet could construct "intent-like" frame transactions where the SENDER frames describe desired outcomes and the sponsor's VERIFY frame checks that those outcomes are achieved, reverting if not. This is not full intent expressiveness, but it captures a significant subset without protocol changes beyond EIP-8141.

### Batch Nonce Advancement for Parallel Transaction Submission

While the specification limits the mempool to one pending frame transaction per sender, the frame model itself does not prevent constructing transactions that advance the nonce and then execute multiple independent operations in subsequent frames. A single frame transaction can effectively replace what would be N sequential transactions today. This is particularly valuable for automated systems (trading bots, liquidation bots) that need to execute many operations per block but are currently limited by nonce ordering.

---

## VI. Changes to Increase Optionality Further

### 1. Add a Per-Frame Intrinsic Cost

The current intrinsic cost is flat: 15,000 gas regardless of frame count. Adding a per-frame cost (e.g., 1,000 gas per frame) would make the gas model more honest about the work validators perform per frame and would discourage gratuitous frame inflation. It also creates a natural economic pressure toward frame-efficient transaction design.

### 2. Allow Optional Gas Forwarding Between Frames

The strict gas isolation between frames is safe but inflexible. Consider adding an optional "gas forwarding" flag (another mode bit) that allows unused gas from frame N to be added to frame N+1's gas limit. This would only apply to non-VERIFY frames and would be opt-in. It would dramatically simplify gas estimation for multi-frame execution sequences while preserving isolation for the validation prefix.

### 3. Define a TXPARAM Entry for Individual Blob Hashes

Add TXPARAM param 0x0A with in2 as blob index, returning the individual blob versioned hash. This enables VERIFY frames to validate commitments to specific blob data, which is necessary for blob-carrying frame transactions to be fully self-describing.

### 4. Reserve TXPARAM Entries for Future Multidimensional Gas

The specification notes that TXPARAM entries 0x03 and 0x04 have "possible future extension" for multidimensional gas. Make this extension point explicit by defining the behavior when in2 != 0 for these entries (e.g., exceptional halt today, reserved for future gas dimensions). This prevents client implementations from making assumptions about these entries that would break when multidimensional gas is added.

### 5. Consider a DELEGATE Frame Mode

A fourth frame mode -- DELEGATE -- that executes code at a target address but in the context of the sender (like DELEGATECALL) would enable powerful patterns. A smart account could delegate complex execution logic to a library contract without giving that contract the ability to act as the sender. This is different from SENDER mode because the code runs in the sender's storage context, and different from DEFAULT because the caller is the sender, not ENTRY_POINT. This would enable upgradeable execution logic for smart accounts without requiring the account itself to contain all logic.

### 6. Explicit Frame Dependency Declarations

Allow frames to optionally declare dependencies on previous frames: "this frame should be skipped if frame N reverted." Currently this is partially achievable via atomic batching (which reverts the whole group) and via TXPARAM status checks (which require the frame to execute and check). An explicit dependency declaration would let the protocol skip frames without executing them, saving gas on dependent operations that are known to be unnecessary.

### 7. Separate the Mempool Policy into a Companion Document

Move the mempool section (validation prefixes, banned opcodes, paymaster accounting) into a separate ERC or informational EIP. The core protocol specification should define what frame transactions ARE and how they execute. The mempool policy should define how nodes handle them in the transaction pool. These are different stability levels -- protocol rules are permanent; mempool policy evolves with the threat landscape. Separating them allows mempool policy to be updated without amending the core EIP.

### 8. Add a Frame-Level Return Data Introspection Mechanism

TXPARAM provides the status (success/failure) of previous frames, but there is no mechanism to read the return data of a previous frame. Adding FRAMERETURNDATA opcodes (analogous to RETURNDATASIZE and RETURNDATACOPY but indexed by frame) would enable powerful composition patterns. A sponsor's post-op frame could inspect what the user's execution frame returned, enabling more sophisticated settlement logic.

---

## VII. Summary Assessment

EIP-8141 is the most structurally ambitious Ethereum protocol change since the merge. It does not patch account abstraction onto the existing transaction model -- it replaces the transaction model with one that has account abstraction as a first principle. The frame metaphor is correct: a transaction is not a single action but a sequence of actions with distinct roles and trust boundaries.

The design is strongest in its core execution model: the frame modes, the APPROVE consent mechanism, the signature hash elision, and the atomic batching. These are clean, minimal, and composable.

The design is weakest in its boundary conditions: the excessively high MAX_FRAMES, the flat intrinsic cost, the gas isolation that will create UX friction, and the mempool policy that is entangled with the protocol specification.

The most important thing this EIP enables is not any single use case -- it is the elimination of the distinction between "protocol-native accounts" and "smart accounts." After this EIP, every account is a smart account. Some just happen to use the default code. That is the correct end state for Ethereum's account model, and this specification gets there with less complexity than I would have expected.

The post-quantum motivation is real but should not overshadow the broader significance. This EIP does not merely add PQ signature support. It makes the signature scheme a user-space decision rather than a protocol-level constraint. That is a fundamentally different kind of flexibility, and the design decisions made here will constrain or enable Ethereum's evolution for the next decade.

Build it. But reduce MAX_FRAMES, add per-frame intrinsic cost, separate the mempool policy, and be explicit about the VERIFY-frame static execution exception for APPROVE. The foundation is sound. The edges need sharpening.
