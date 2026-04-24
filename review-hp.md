# EIP-8141: Frame Transaction -- Review

**Reviewer:** Hilmar Veigar Petursson perspective (sandbox MMO design philosophy)
**Date:** 2026-04-12
**Status of EIP:** Draft


## Preamble

Ethereum is a shared, persistent universe. It has a single shard -- one canonical state that every participant inhabits simultaneously. When an alliance falls in EVE Online, everyone knows. When a DeFi protocol collapses on Ethereum, everyone knows. That shared reality is the foundation of meaning. EIP-8141 is an infrastructure change to how citizens of this universe authenticate themselves and pay for their actions. It deserves scrutiny proportional to its ambition.

This review evaluates EIP-8141 through the lens of building systems where players -- users, developers, protocols -- are the content, and where the design goal is maximum emergent complexity from minimal prescriptive rules.


## 1. What I Like

### 1.1 The Frame Abstraction Itself Is Elegant

The core insight -- decomposing a transaction into an ordered sequence of typed frames (VERIFY, SENDER, DEFAULT) with distinct execution contexts -- is genuinely good design. It is the kind of primitive that enables emergent complexity. Rather than prescribing "here is how account abstraction works," the EIP says "here are composable execution contexts; build what you need."

This mirrors the best sandbox design principle: give players tools, not scripts. The frame list is a programmable pipeline where verification, execution, and payment are separable concerns that users compose freely. A transaction with `[deploy, verify, execute]` and one with `[verify, sponsor_verify, erc20_transfer, execute, post_op]` use the same primitive in fundamentally different ways. That is good.

### 1.2 Sender Is Explicit, Not Derived

The `sender` field being explicit in the transaction payload rather than derived from `ecrecover` is a profound change. It decouples identity from cryptography. This is the equivalent of saying "your character in this universe is not your password." The address becomes a persistent identity that can outlive any particular authentication mechanism. This is essential for post-quantum migration, but its implications go far beyond that.

In EVE Online, a corporation continues to exist even when its CEO changes. The identity of the entity persists across leadership transitions. Explicit sender enables the same for Ethereum accounts: the account persists across key rotations, signature scheme migrations, and governance model changes.

### 1.3 Atomic Batching via Mode Flags

The atomic batch design (bit 11 on consecutive SENDER frames) is clean. The approve-then-swap pattern in Example 2 is the most common source of user grief in DeFi today -- dangling approvals from failed swaps. Making this atomic at the transaction level rather than requiring wrapper contracts is a meaningful improvement.

The flag-based approach (rather than a new mode) is pragmatic. It avoids combinatorial explosion of mode types while enabling the most important multi-call pattern. The boundary detection rule (batch ends at the SENDER frame without the flag) is unambiguous.

### 1.4 Default Code for EOAs

The default code behavior that gives EOAs implicit smart-account-like capabilities (ECDSA and P256 verification in VERIFY mode, RLP-decoded multi-call in SENDER mode) is a strong concession to backwards compatibility without sacrificing the architecture. It means the 200+ million existing EOAs are not second-class citizens on day one.

The P256 support (signature type `0x1`) in default code is forward-thinking. Passkey-based authentication becomes natively available to every EOA without deployment.

### 1.5 VERIFY Data Elision from Signature Hash

Eliding VERIFY frame data from the signature hash (`compute_sig_hash`) is a subtle but important design choice. It means the signature itself is not part of what gets signed (obviously necessary), but it also means sponsor data can be added after the sender signs. This enables a genuinely trust-minimized sponsorship flow where the sender commits to the sponsor address (via `frame.target`) but the sponsor's proof-of-payment commitment can be attached later. This is good separation of concerns.

### 1.6 The Receipt Structure

Including `payer` in the receipt and per-frame `[status, gas_used, logs]` is the right level of observability. You cannot manage what you cannot measure. In EVE, we track every transaction, every market movement, every ship loss. Ethereum needs equivalent granularity. Per-frame receipts make debugging, analytics, and MEV analysis tractable for this new transaction type.

### 1.7 TXPARAM and FRAMEDATALOAD/FRAMEDATACOPY Opcodes

These opcodes let frames reason about the transaction they are part of. A verification contract can inspect the signature hash, the number of frames, the targets and modes of other frames. This is genuine introspection -- the transaction can examine itself. This enables validation logic that is context-aware: "I approve this transaction only if frame 2 targets this specific DEX contract" or "I approve payment only if total gas is below X."

The ability for a sponsor's VERIFY frame to inspect subsequent frames (their targets, gas limits, data lengths) without seeing VERIFY frame data is a well-considered information boundary.


## 2. What I Do Not Like

### 2.1 MAX_FRAMES = 10^3 Is Excessive Without Justification

One thousand frames per transaction is a lot. The examples in the EIP use 2-5 frames. The most complex realistic scenario I can envision (deploy + verify + multi-sponsor-negotiation + batch-execute + post-ops) might use 15-20 frames. What scenario requires 1000?

This is the equivalent of allowing a fleet of 1000 ships in a single warp -- technically possible, but it creates a weapon that will be used in ways you did not intend. An attacker who crafts a 1000-frame transaction with carefully chosen gas limits creates a novel resource exhaustion vector at the execution layer, even if mempool rules constrain the validation prefix.

The EIP should either justify this limit with a concrete use case or reduce it substantially. A limit of 64 or 128 would cover every reasonable scenario while limiting the blast radius of adversarial usage. Alternatively, there should be a per-frame minimum gas floor or a pricing curve that makes high frame counts progressively more expensive.

### 2.2 Gas Isolation Between Frames Is Too Rigid

"Unused gas from a frame is not available to subsequent frames." This is stated as a feature but it is also a significant tax on usability.

Consider the sponsored transaction example (Example 3). The user must estimate gas for frames 2, 3, and 4 independently. If frame 3 (the actual user operation) uses less gas than allocated, that gas is wasted -- it cannot flow to frame 4 (the post-op). This means either:

1. Every frame must be over-provisioned for safety, wasting gas.
2. Complex off-chain gas estimation must predict each frame's usage precisely.
3. Post-op frames risk failure because they cannot absorb surplus from earlier frames.

I understand the motivation: gas isolation prevents one frame from starving another. But the current design creates a different problem -- it makes frame transactions systematically more expensive than their single-frame equivalents because of the required over-provisioning.

A middle ground would be to allow a "shared gas pool" opt-in via another mode flag, or to allow the final frame to access remaining gas from prior frames.

### 2.3 The Mempool Section Is Doing Too Much Work

The mempool rules section is nearly as long as the core specification. This is a smell. It suggests the core design creates problems that the mempool section must patch.

The validation prefix concept, the canonical/non-canonical paymaster distinction, the banned opcode list, the trace rules, the revalidation logic -- this is an enormous amount of complexity imposed on every node operator. Compare this to legacy transactions where mempool validation is: check nonce, check balance, check signature. Done.

More concerning: the banned opcode list during validation (`ORIGIN`, `GASPRICE`, `BLOCKHASH`, `COINBASE`, `TIMESTAMP`, `NUMBER`, `PREVRANDAO`, `GASLIMIT`, `BASEFEE`, `BLOBHASH`, `BLOBBASEFEE`, `GAS`, `CREATE`, `CREATE2`, `INVALID`, `SELFDESTRUCT`, `BALANCE`, `SELFBALANCE`, `SSTORE`, `TLOAD`, `TSTORE`) is 21 opcodes. This is a substantial restriction that will surprise smart account developers. The EIP should be much more prominent about this -- perhaps a dedicated "Validation Restrictions" section before the mempool details, since this directly affects what smart accounts can do during their most critical operation.

### 2.4 ORIGIN Semantics Change Is Buried

The change to `ORIGIN` -- returning frame caller rather than transaction origin throughout all call depths -- is mentioned in one sentence in the Behavior section and one paragraph in Backwards Compatibility. This is a breaking change for existing contracts.

Yes, `tx.origin` checks are a discouraged pattern. Yes, EIP-7702 already modified ORIGIN semantics. But "discouraged" does not mean "unused." There are deployed contracts with real value locked behind `tx.origin == msg.sender` checks. The EIP should explicitly quantify the risk surface: how many contracts use `ORIGIN`? What is the total value at risk?

This is the kind of change that, in a persistent universe, can destroy value that took years to accumulate. It deserves more than a paragraph.

### 2.5 No Explicit Upgrade/Migration Path from EIP-4337

EIP-4337 (ERC-4337) has been live since 2023. There is a substantial ecosystem of bundlers, paymasters, smart accounts, and SDK infrastructure built around it. EIP-8141 appears to subsume 4337's functionality at the protocol level, but the EIP says nothing about migration.

What happens to existing 4337 smart accounts? Can they use frame transactions directly? Is the EntryPoint contract (4337's version) still relevant? What about existing paymaster contracts -- do they need to be redeployed as canonical paymasters?

A persistent universe must respect the investments its citizens have already made. Ignoring the 4337 ecosystem in this EIP is like redesigning corporation mechanics in EVE without addressing what happens to existing corporations.

### 2.6 The APPROVE Opcode Constraint (ADDRESS == frame.target) Is Unnecessarily Restrictive

`APPROVE` reverts if `ADDRESS != frame.target`. This means only the direct target of a frame can approve -- not a delegate, not a library, not a proxy implementation. If a smart account uses a proxy pattern (which most do), the proxy's implementation contract has a different `ADDRESS` than the proxy itself.

The EIP needs to clarify how this interacts with `DELEGATECALL`. If the smart account at `frame.target` uses `DELEGATECALL` to an implementation contract, does `ADDRESS` return `frame.target` (the proxy) or the implementation? If the former, this works. If the latter, every proxy-based smart account breaks. The specification says "the currently executing account" which in `DELEGATECALL` context is the caller's address -- so this likely works, but the EIP should state this explicitly.


## 3. Issues

### 3.1 TXPARAM Parameter Numbering Gap

The `TXPARAM` opcode parameters jump from `0x09` to `0x10`. Parameters `0x0A` through `0x0F` are unused but not marked as reserved. This is either a typo (should `0x10` be `0x0A`?) or intentional reservation without documentation. If intentional, the reserved range should be noted. If a typo, the frame-index-based parameters should start at `0x0A`.

This matters because smart account code will hardcode these values. Ambiguity here creates implementation divergence between clients.

### 3.2 Scope Operand Values in APPROVE Are Inconsistent with Mode Descriptions

In the `APPROVE` Behavior section, scope `0x2` is described as: "Increment the sender's nonce, collect the total gas cost." But the scope operand documentation says `0x2` is "Approval of payment."

More critically, scope `0x2` behavior says "If `sender_approved == false`, revert the frame." This means you CANNOT have a standalone gas-payer without a sender approval first. The dependency is: sender must approve before payer can approve. This is stated in the notes but it creates an ordering constraint that limits composability.

What if a paymaster wants to pre-commit to paying for a class of transactions before knowing the specific sender? The current design forbids this. The ordering constraint (sender before payer) is reasonable for security but should be explicitly called out as a design choice with its tradeoffs, not buried in the APPROVE behavior.

### 3.3 Nonce Increment Timing Creates a Window

The nonce increment happens inside `APPROVE(0x2)` or `APPROVE(0x3)`. Until that point in execution, the nonce has not been incremented. This means:

1. If a transaction's validation prefix fails before reaching APPROVE, the nonce is not consumed.
2. Multiple frame transactions with the same nonce can exist in the mempool simultaneously (the spec says "at most one pending frame transaction per sender" but this is a mempool policy, not a consensus rule).
3. The nonce check happens at the start ("Ensure `tx.nonce == state[tx.sender].nonce`") but the increment happens later. Between these two points, the transaction is in a liminal state where it has "claimed" a nonce but not consumed it.

This is probably fine for consensus (the whole transaction is atomic) but creates complexity for mempool implementations and MEV searchers who need to reason about nonce ordering.

### 3.4 Default Code SENDER Mode Is Dangerously Permissive

The default code for EOAs in SENDER mode reads `frame.data` as RLP-encoded `[[target, value, data], ...]` and executes each call. If any call reverts, the entire frame reverts. But there is no gas limit per sub-call -- the entire frame's gas allocation is available to each sequential call.

This means a single SENDER frame with default code can execute an arbitrary number of calls with arbitrary calldata. Combined with the fact that the EOA's private key holder has already approved execution via a VERIFY frame, this is probably intentional. But it means a compromised key can drain an EOA of all assets (ETH, ERC-20s, NFTs) in a single transaction by encoding transfers to every known token contract.

This is not necessarily a flaw -- it is the same as today (a compromised key can drain everything). But the multi-call capability makes it more efficient for attackers while being presented as a user-experience improvement.

### 3.5 Transient Storage Discard Between Frames

"Discard the TSTORE and TLOAD transient storage between frames." This means transient storage cannot be used for inter-frame communication within a single transaction. This is surprising given that transient storage (EIP-1153) was designed specifically for intra-transaction communication.

The warm/cold journal IS shared across frames, but transient storage is not. This inconsistency should be explained. If the rationale is that transient storage could leak information between validation and execution phases, say so. As written, it appears arbitrary.

### 3.6 No Mechanism for Frame-Level Gas Refund Targeting

The gas refund goes to "the target that called APPROVE(0x2) or APPROVE(0x3)." In a sponsored transaction, this is the paymaster, not the user. But what if the user overpaid in ERC-20 tokens for the gas the paymaster fronted? The post-op frame (Example 3, Frame 4) is supposed to handle this, but if the post-op itself runs out of gas, the refund goes to the paymaster in ETH while the user has already transferred ERC-20 tokens in frame 2.

This creates a systematic advantage for paymasters: they receive ETH refunds for gas they were paid for in ERC-20 tokens. The spread between "ERC-20 charged to user" and "ETH refunded to paymaster" becomes paymaster profit even when no actual spread was intended.


## 4. Usage Scenarios I Like

### 4.1 Key Rotation Without Account Migration

A smart account deployed via frame 0 (DEFAULT mode, known deployer) can implement key rotation in its verification logic. Today, changing your key means changing your account or using a proxy. With frame transactions, you deploy once and your verification frame accepts signatures from whatever key is currently authorized in your account's storage.

This is identity persistence. Your Ethereum address becomes your permanent identity, not your ECDSA key. This is profoundly important for a persistent universe.

### 4.2 Social Recovery via Verification Composition

A smart account could require multiple VERIFY frames: one for the owner's signature, one for a guardian's co-signature. The account's verification logic checks that both frames approved before executing SENDER frames. This is on-chain multisig at the transaction level, not the contract level.

More interestingly, the guardian's VERIFY frame could be a different account type entirely -- a Gnosis Safe, a timelock, a social recovery contract. The composability of verification frames enables governance structures that the EIP authors did not need to design.

### 4.3 Progressive Security Tiers

A smart account could implement tiered verification based on transaction value:
- Low value (< 0.1 ETH): single passkey signature
- Medium value (< 10 ETH): passkey + time delay
- High value (> 10 ETH): passkey + hardware key + guardian co-sign

Each tier uses different frame structures, but the same account address. The account's verification code inspects the frame list via TXPARAM and enforces the appropriate security level.

### 4.4 Gasless Onboarding

The deploy-then-verify-then-execute pattern (Example 1b) combined with canonical paymaster sponsorship means a new user can:
1. Generate a key pair (or passkey)
2. Compute their counterfactual address
3. Receive funds/tokens at that address
4. Deploy their account, verify, and execute in a single transaction sponsored by a third party

Zero ETH required. Zero prior on-chain presence required. This is the "create a character and start playing" experience that every new Ethereum user should have.

### 4.5 MEV-Protected Transaction Bundles

Atomic batching of SENDER frames enables MEV-protected sequences at the user level. An approve-swap-verify-balance sequence that reverts atomically if the output is below expectations is no longer something that requires Flashbots or private mempools -- it can be expressed in a single frame transaction.

### 4.6 Cross-Protocol Operations as Single Transactions

A user could compose: `[verify, deposit_to_aave, borrow_from_aave, swap_on_uniswap, provide_liquidity]` as a single atomic frame transaction. Today this requires a router contract or a bundler. With frame transactions, the user's wallet constructs the frame sequence directly.

This is "players building their own content" -- users composing financial operations without intermediary contracts, using the protocol's primitives directly.


## 5. What This Enables That We Are Not Thinking About

### 5.1 On-Chain Organizations with Native Authentication

Frame transactions enable DAOs where membership and voting authority are encoded in the verification layer. A DAO's smart account could require VERIFY frames from N-of-M council members. The frame transaction IS the governance vote AND the execution in one atomic operation.

This moves governance from "proposal, vote, timelock, execute" (four transactions over days) to "compose frame transaction, collect signatures, submit" (one transaction). The governance IS the transaction.

### 5.2 Programmable Transaction Policies

Enterprises and institutions can deploy smart accounts with verification logic that enforces spending policies: daily limits, approved counterparties, required approvals above thresholds. These policies execute at the protocol level during VERIFY frames, not in application-layer wrapper contracts. This means the policy cannot be bypassed by calling the account directly -- the verification is part of every frame transaction.

This is "letting players build governments" -- organizations encoding their rules into their account's transaction validation.

### 5.3 Account-Level MEV Capture

A smart account's VERIFY frame could implement an auction: instead of a simple signature check, it could verify that the transaction was submitted by a searcher who included a payment to the account owner. The account itself becomes an MEV capture mechanism.

More concretely: a smart account could accept any valid signature but only APPROVE if the frame list includes a SENDER frame that transfers a "tip" to the account. The account's owner delegates transaction submission to searchers who compete to include the most profitable execution. This is player-driven market-making at the account level.

### 5.4 Automated Account Hierarchies

A parent smart account could authorize child accounts whose VERIFY frames check authorization against the parent's storage. Frame transactions enable institutional account hierarchies where:
- A treasury account authorizes department accounts
- Department accounts authorize individual operator accounts
- Each level has its own spending rules and verification requirements

The authorization chain is verified in VERIFY frames, not application logic. This is alliance management infrastructure at the protocol level.

### 5.5 Dead Man's Switch / Account Recovery

A smart account's verification logic could include a time-based fallback: "if the primary key has not been used in 12 months, accept the recovery key." Combined with the banned-opcode exception (TIMESTAMP is banned in mempool validation but not at consensus level), this could be implemented as a two-phase process where the recovery transaction is submitted directly to a block builder.

This addresses the real "permadeath" problem in crypto: lost keys. A well-designed smart account can include account recovery mechanisms that activate after dormancy periods, preserving the persistent identity even when the original keyholder is unavailable.

### 5.6 Composable Authentication Standards as a Market

Frame transactions create a market for verification modules. Developers can publish verification contracts that smart accounts delegate to via their VERIFY frames: "Passkey verifier v2," "Multi-sig verifier with social recovery," "Biometric bridge verifier." Accounts switch between these by updating their verification logic, not by migrating to new accounts.

This is an entirely new category of smart contract -- the authentication module -- that does not exist today because verification has been hardcoded to ECDSA. Frame transactions make authentication a composable, upgradeable, tradeable primitive.

### 5.7 Session Keys and Ephemeral Authorization

A smart account could implement session keys: temporary keys authorized for specific operations within specific time windows. The VERIFY frame checks the session key's authorization against the account's storage (which was set by a prior full-authority transaction). The session key can then execute SENDER frames within its authorized scope.

This enables "log in once, play for hours" UX without persistent private key exposure. The session key has limited authority (specific contract targets, limited value, time-bounded) and the full key remains in cold storage.

### 5.8 Interoperable Cross-L2 Account Models

If L2s adopt EIP-8141, the same smart account at the same address can have different verification logic on different chains (or the same logic) while maintaining a consistent identity. The frame transaction structure is chain-agnostic by design. This enables an account that uses passkeys on Base (for mobile UX), hardware keys on mainnet (for high-value operations), and social recovery on Arbitrum (for its social-heavy applications).


## 6. Changes to Increase Optionality

### 6.1 Add a Gas Forwarding Mechanism Between Frames

As discussed in Section 2.2, strict gas isolation between frames creates systematic waste. Proposal: add a mode flag (bit 12?) for "gas remainder forwarding" that allows unused gas from a frame to flow to the next frame, or to a designated "shared pool" frame.

This does not compromise the isolation benefits for validation frames (where gas limits must be strict for mempool safety) but enables execution frames to share gas more efficiently.

### 6.2 Define a Frame Callback / Continuation Mechanism

Currently, frames execute strictly sequentially. A "callback" frame type that executes after a specified earlier frame completes (regardless of success/failure) would enable try-catch patterns at the transaction level.

Use case: "Execute this swap, and regardless of whether it succeeds, update my account's state to reflect the attempt." Currently, the atomic batch flag provides all-or-nothing. A callback/finally mechanism provides guaranteed-execution-regardless-of-outcome.

### 6.3 Reduce MAX_FRAMES and Add Frame Count Pricing

Replace `MAX_FRAMES = 10^3` with `MAX_FRAMES = 64` and add a small per-frame surcharge to the intrinsic cost (e.g., 500 gas per frame beyond the first two). This preserves the ability to compose complex transactions while disincentivizing adversarial frame counts.

### 6.4 Allow TSTORE/TLOAD Across Frames with an Opt-In

The current design discards transient storage between frames. This prevents useful patterns like a VERIFY frame storing parsed authorization data for a subsequent SENDER frame to consume. An opt-in mechanism (another mode flag, or a designated "shared transient storage" scope) would enable inter-frame communication without compromising isolation by default.

### 6.5 Explicit EIP-4337 Compatibility Section

Add a section describing:
- How existing ERC-4337 smart accounts can migrate to frame transactions
- Whether existing EntryPoint contracts interact with frame transactions
- A recommended migration path for paymaster contracts
- Whether bundlers should/can convert UserOperations into frame transactions

This is not just courtesy -- it is strategic. The 4337 ecosystem represents significant invested effort. A clear migration path accelerates adoption. An unclear one creates ecosystem fragmentation.

### 6.6 Add a FRAMESTATUS Opcode or Extend TXPARAM

Currently, `TXPARAM(0x15, frameIndex)` returns status but halts if you query the current or future frame. A post-op frame cannot check whether the user's operation succeeded. The post-op must infer success from state changes.

Adding the ability for a DEFAULT-mode post-op frame to check the status of prior SENDER frames would enable conditional post-processing: "if the swap failed, do not charge the user."

Looking at the spec again, `TXPARAM(0x15, frameIndex)` does allow reading status of prior frames -- the exceptional halt is only for current/future frames. This is correct but the post-op pattern should be explicitly documented as a supported use case, since it is the primary reason post-op frames exist.

### 6.7 Consider a Frame-Level Value Field

The EIP explicitly excludes value from frames ("No value in frame -- It is not required because the account code can send value"). While technically true, this forces every ETH transfer to go through SENDER mode with calldata encoding. For the most common operation on Ethereum (sending ETH), this adds bytes and complexity.

A `value` field in SENDER frames, defaulting to 0, would save calldata for the most common case and reduce the cognitive overhead for wallet developers.

### 6.8 Reserve Mode Values for Future Cross-Frame Patterns

Modes 3-255 are reserved. Consider explicitly reserving specific modes for anticipated future needs:
- A `CONDITIONAL` mode for frames that execute only if prior frames meet certain conditions
- A `PARALLEL` mode for frames that do not touch overlapping state and could theoretically execute concurrently
- A `READONLY` mode (like VERIFY but without the APPROVE requirement) for frames that just need to read state for use by subsequent frames

These do not need to be implemented now, but reserving the semantic space signals intent and prevents future mode assignments from conflicting with ecosystem assumptions.

### 6.9 Clarify the Interaction with EIP-7702 Delegations

The EIP mentions that EIP-7702 delegations are restricted during validation (cannot `CALL*` or `EXTCODE*` to a delegation target), but the interaction between frame transactions and 7702 delegations deserves its own subsection. Can a 7702-delegated EOA use frame transactions? If so, which code runs -- the delegated code or the default code? If a frame transaction's sender has a 7702 delegation, is that delegation active during VERIFY frames?

These are not edge cases. EIP-7702 is live. Frame transaction users with 7702 delegations will exist on day one.


## Concluding Observations

EIP-8141 is ambitious infrastructure for a persistent shared universe. It does for authentication and transaction execution what EIP-1559 did for fee markets: it takes something that was previously rigid and protocol-prescribed and makes it flexible and user-defined.

The frame abstraction is the right primitive. The composability it enables -- from simple EOA transactions to complex sponsored, batched, multi-verified operations -- is a generational upgrade to Ethereum's expressiveness.

The concerns are real but tractable: the mempool complexity is significant but manageable (the 4337 ecosystem has already proven this at scale); the gas isolation rigidity can be relaxed incrementally; the ORIGIN change needs more visibility but affects a declining number of contracts; and the migration path from 4337 can be documented.

The most important thing this EIP gets right is that it does not prescribe HOW accounts should authenticate, pay for gas, or compose operations. It provides the execution contexts and lets the ecosystem build. That is the sandbox philosophy: your job is to be the universe, the players are the content.

The stories that will emerge from this primitive -- the authentication markets, the organizational hierarchies, the session key protocols, the MEV capture mechanisms, the social recovery networks -- are stories the EIP authors have not written and cannot predict. That is exactly how it should be.
