# Review of EIP-8141: Frame Transaction

**Reviewer:** Tomasz K. Stanczak
**Date:** 2026-04-12
**Status of EIP:** Draft

---

## 1. What I Like

### The frame abstraction is the right primitive

After years of watching account abstraction proposals accumulate complexity at the wrong layers -- EntryPoint contracts, bundler networks, off-chain UserOperation mempools -- this EIP finally moves the abstraction where it belongs: into the protocol itself. The `[mode, target, gas_limit, data]` frame tuple is a clean, composable unit of execution. It does not invent new execution semantics; it sequences existing EVM calls with explicit mode annotations. That is a design that client teams can actually implement without rewriting half their transaction processing pipeline.

The separation of VERIFY, SENDER, and DEFAULT modes is particularly well-done. VERIFY being STATICCALL-equivalent means validation cannot modify state, which is exactly the guarantee the mempool needs to reason about transaction validity without full simulation of side effects. This is something ERC-4337 never achieved at the protocol level, instead relying on off-chain bundler reputation systems.

### Explicit gas isolation per frame

Each frame having its own `gas_limit` with no spillover between frames is a strong design choice. It means a validation frame cannot be gas-starved by a preceding execution frame, and a misbehaving execution frame cannot consume gas allocated to a post-op cleanup. This is the kind of isolation that makes reasoning about worst-case behavior tractable -- you know the maximum cost of validation is bounded independently of what happens during execution.

### The signature hash elision for VERIFY frames

Eliding VERIFY frame data from the signature hash is subtle and correct. The signature obviously cannot sign over itself, but the deeper insight is that this enables future signature aggregation. If VERIFY data were visible to other frames or included in the hash, aggregation schemes (BLS, SNARK-based batch verification) would be blocked. The EIP authors are leaving the door open for a massive throughput optimization without committing to it now. That is good engineering.

### The canonical paymaster design is pragmatic

The dual-track paymaster system -- canonical paymasters identified by code match, non-canonical paymasters throttled to one pending transaction -- is a pragmatic solution to a real problem. ERC-4337 struggled with paymaster DoS for years. This design acknowledges that you cannot have both arbitrary paymaster logic and unlimited mempool propagation, so it draws a clean line: be canonical and get full mempool access, or be custom and accept throttling. The timelocked withdrawal mechanism for canonical paymasters is exactly the right constraint -- it prevents the "rug pull the mempool" attack where a paymaster withdraws its balance to invalidate all sponsored transactions simultaneously.

### EOA default code is the correct migration path

The default code specification that gives EOAs implicit VERIFY behavior (ECDSA + P256) and implicit SENDER behavior (RLP-decoded call list) means every existing Ethereum user can start using frame transactions immediately. No migration, no delegation, no smart account deployment required for basic usage. The P256 support in particular is forward-looking -- it enables passkey/WebAuthn authentication from day one.

### Post-quantum motivation is honest and timely

The EIP does not pretend blockchain needs to be "quantum resistant today." It correctly identifies that we need a native off-ramp from ECDSA *before* quantum computers are a threat, because migrating 200+ million accounts after the fact is not feasible. The frame transaction is that off-ramp.

---

## 2. What I Do Not Like / Concerns

### MAX_FRAMES = 10^3 is too large

A thousand frames in a single transaction is excessive. The practical examples in the EIP use 2-5 frames. Even a complex deployment + verification + sponsored execution + post-op scenario needs at most 6-8 frames. Allowing 1000 frames creates a surface area for abuse: a transaction with 1000 frames, each with its own gas limit, creates complex gas accounting, receipt bloat, and potential edge cases in atomic batching semantics.

I would recommend MAX_FRAMES = 16 or at most 32. If someone genuinely needs more than 16 sequential execution frames in a single transaction, that is a design smell in their application, not a limitation of the protocol. If a future use case demands more, it is easier to raise a limit than to lower one.

### Gas isolation means gas waste at scale

The "no spillover" gas model is safe but inefficient. If a validation frame is allocated 100,000 gas but uses 30,000, those 70,000 units are dead -- they cannot be used by subsequent frames, though they are refunded to the payer after all frames execute. This is fine for the simple case but creates a tension: users must over-allocate gas per frame to handle worst-case paths, which means the effective gas utilization of a block filled with frame transactions is lower than a block filled with legacy transactions.

The EIP does refund unused gas after all frames complete, so the economic cost is bounded. But the block gas capacity impact is real -- those allocated-but-unused gas units still count against the block gas limit during execution, reducing the number of transactions a block can include. This deserves more analysis in the EIP.

### The ORIGIN opcode semantic change is more disruptive than claimed

The EIP says: "This is consistent with the precedent set by EIP-7702, which already modified ORIGIN semantics." But EIP-7702 changed ORIGIN in a narrow, opt-in context (delegated EOAs). EIP-8141 changes ORIGIN for *all* frame transactions -- a new transaction type that is intended to eventually replace legacy transactions. Contracts that use `tx.origin` for reentrancy guards or same-transaction detection (a pattern that exists in production DeFi despite being discouraged) will behave differently. The backwards compatibility section is too brief on this point.

The specific change -- ORIGIN returns the frame caller rather than the transaction sender -- means ORIGIN varies *within* a single transaction depending on which frame you are in. This is a fundamental semantic shift. If Frame 0 has caller ENTRY_POINT and Frame 2 has caller tx.sender, contracts called from those frames see different ORIGIN values despite being in the same transaction. This needs a dedicated analysis of impacted deployed contracts.

### Transient storage being discarded between frames is surprising

The specification states: "Discard the TSTORE and TLOAD transient storage between frames." Transient storage (EIP-1153) was designed to be transaction-scoped. Discarding it between frames means frame transactions have a *different* transient storage lifetime than legacy transactions. This creates a subtle compatibility hazard: a contract that relies on transient storage to communicate between nested calls within a single frame works fine, but the same pattern fails if the calls are split across frames.

This is a defensible design choice (it prevents cross-frame side channels during validation), but it should be called out more prominently and the rationale should be explicit. The security section should discuss what attacks this prevents.

### The mempool specification is underspecified for client diversity

The mempool section describes what validation MUST occur but leaves implementation details to clients. For a protocol change that introduces EVM execution during mempool validation, the specification needs to be more precise about:

- What happens if two clients disagree on whether a transaction's validation prefix is valid (e.g., due to different gas metering implementations)?
- How does this interact with the Ethereum consensus layer's existing transaction gossip protocol?
- What is the computational budget for mempool validation per transaction? MAX_VERIFY_GAS of 100,000 bounds EVM execution, but what about the overhead of RLP decoding, frame structure analysis, and canonical paymaster code matching?

### No explicit treatment of state access conflicts between concurrent frame transactions

Two frame transactions from different senders but using the same canonical paymaster will both read and potentially write the paymaster's balance during validation. The EIP describes per-node accounting via `reserved_pending_cost`, but does not address what happens when a block builder orders these transactions and the second one fails because the paymaster's actual balance was consumed by the first. This is not a bug in the EIP per se -- it is the normal mempool eviction problem -- but the interaction between paymaster reservation accounting and block building deserves more attention.

---

## 3. Issues

### Issue 1: TXPARAM param numbering gap

The `TXPARAM` opcode uses param values `0x00` through `0x09`, then jumps to `0x10` for frame-specific parameters. The gap from `0x0A` to `0x0F` is wasted without explanation. Either these should be reserved explicitly (with a note about intended future use) or the frame parameters should start at `0x0A` to keep the encoding dense. A sparse encoding wastes bits on every TXPARAM call and creates ambiguity about whether values in the gap are invalid or simply undefined.

### Issue 2: Atomic batch semantics need snapshot cost analysis

Atomic batching requires "a snapshot of the state before executing the first frame in the batch." State snapshots in the EVM are not free -- they involve journaling all state changes. For a batch of N SENDER frames, the journal can grow proportional to the total state modifications across all N frames. The EIP does not specify:

- Is there a maximum batch size?
- What is the gas cost of the snapshot mechanism itself?
- Can nested atomic batches exist? (The current spec says consecutive SENDER frames form a batch, but can two batches be adjacent?)

The third question is answered implicitly (the boundary is the SENDER frame without the flag), but the first two are not addressed. A malicious user could construct a batch of, say, 500 SENDER frames (within MAX_FRAMES=1000), each modifying significant state, forcing the client to maintain a massive journal for potential rollback. The gas cost of the individual frames may not adequately cover the snapshot overhead.

### Issue 3: Default code P256 address derivation creates a second address space

For P256 signatures in default code, the sender address is derived as `keccak(qx|qy)[12:]`. This means P256-authenticated accounts have addresses that are indistinguishable from regular EOA or contract addresses but are derived from a completely different key type. There is no on-chain way to determine whether an address `0xABC...` is an ECDSA EOA, a P256 EOA, or a contract, short of checking if it has code.

This creates a collision risk: a P256 key could produce the same address as an existing ECDSA EOA. The probability is negligible (2^-160), but the *implication* of a collision is catastrophic (two different keys controlling the same account). The EIP should explicitly acknowledge this and state whether it is considered acceptable.

### Issue 4: `APPROVE` scope 0x2 requires `sender_approved == true` but scope 0x3 does not

Looking at the APPROVE behavior:
- Scope `0x2` (payment only): "If `sender_approved == false`, revert the frame."
- Scope `0x3` (both): Sets `sender_approved = true` and `payer_approved = true` atomically.

This means scope `0x3` can be called even when the sender has not been separately approved, because it approves the sender itself. But scope `0x2` requires a prior sender approval. This is logically consistent but creates an asymmetry: the self-relay pattern (single VERIFY frame with scope `0x3`) works, but a "pay only" frame can never be the first frame. The EIP should make this ordering constraint more explicit, perhaps with a state machine diagram showing valid transitions.

### Issue 5: Signature hash mutation of tx object

The `compute_sig_hash` function as written mutates the transaction object:

```python
def compute_sig_hash(tx: FrameTx) -> Hash:
    for i, frame in enumerate(tx.frames):
        if (frame.mode & 0xFF) == VERIFY:
            tx.frames[i].data = Bytes()
    return keccak(rlp(tx))
```

This is a specification clarity issue. The pseudocode modifies `tx.frames[i].data` in place. Implementers must understand this is a pure function that operates on a copy, not the original transaction. The function should explicitly state it operates on a copy, or be rewritten to construct the modified frame list without mutation. Client implementation bugs from this kind of ambiguity are common.

### Issue 6: The `only_verify` scope in the mempool section does not match the APPROVE spec

The mempool section states: "`only_verify` must call `APPROVE(0x0)`." But the APPROVE specification says scope values must be one of `0x1`, `0x2`, or `0x3`, and "any other value results in an exceptional halt." Scope `0x0` is not a valid APPROVE argument. This appears to be a bug in the mempool specification. Based on context, `only_verify` should likely call `APPROVE(0x1)` (approval of execution only).

---

## 4. Usage Scenarios / Possibilities I Find Compelling

### Post-quantum migration without flag day

The most compelling use case is the one stated in the motivation, but its implications deserve elaboration. Today, every Ethereum account is protected by a single secp256k1 key. If a quantum computer can break secp256k1, every account is vulnerable simultaneously. Frame transactions allow individual accounts to migrate to post-quantum signature schemes at their own pace. A smart account can implement ML-DSA-65, SPHINCS+, or any future PQ scheme in its VERIFY logic. There is no coordinated migration event. No "hard fork to save everyone." Users who care about PQ security can migrate now; users who do not can migrate later. This is how you do cryptographic agility at scale.

### Passkey-native Ethereum accounts

The P256 support in default code means every smartphone with a secure enclave can be an Ethereum signing device without any smart account deployment. Combined with WebAuthn/FIDO2, this enables Ethereum transactions authenticated by fingerprint or face scan, with the private key never leaving the hardware security module. The user experience could be indistinguishable from a banking app. This is the UX breakthrough that every "mass adoption" presentation has been promising for eight years.

### Trustless ERC-20 gas payment

Example 3 in the EIP (sponsored transaction with ERC-20 payment) is the most economically significant use case. Today, every Ethereum user must hold ETH to transact, even if their entire portfolio is in stablecoins. Frame transactions enable a pattern where a sponsor verifies the user has sufficient ERC-20 balance, approves gas payment from its own ETH, and the user's execution frame transfers ERC-20 tokens to the sponsor as compensation. The sponsor bears the ETH gas cost and receives ERC-20 in return. This is a trustless exchange embedded in the transaction itself.

This unlocks Ethereum for the enormous population of users who hold stablecoins but not ETH. Layer 2s have tried to solve this with proprietary paymaster infrastructure. Frame transactions make it a protocol-level capability.

### Atomic approve-and-swap eliminating the two-transaction dance

The atomic batching feature directly addresses one of DeFi's worst UX patterns: the separate "approve" transaction followed by the "swap" transaction. With frame transactions, an ERC-20 approval and the subsequent DEX swap can be atomic -- if the swap fails, the approval is reverted. No more dangling unlimited approvals sitting in contracts waiting to be exploited. This alone would eliminate a class of approval-based exploits that has cost users hundreds of millions of dollars.

### Multi-call as a first-class citizen

Frame transactions natively support multiple execution frames in a single transaction. Today, multi-call requires either a smart account (with a multicall dispatcher) or a batching contract (like Multicall3). Frame transactions make multi-call a protocol primitive. A user can approve a token, swap on a DEX, and deposit LP tokens into a vault -- all in a single atomic transaction with explicit gas budgets per operation.

---

## 5. Additional Possibilities People May Not Be Thinking Of

### Session keys and delegated authorization

A smart account's VERIFY logic is arbitrary EVM code. This means it can implement session key patterns: "this secondary key is authorized to spend up to X tokens per day on these specific contracts." The VERIFY frame checks the session key's signature, validates the spending limits against storage, and calls APPROVE. The SENDER frames execute the authorized operations. All within a single transaction, all enforced by the smart account's own code, no off-chain relayer required.

This enables a model where a user grants a dApp limited, time-bounded, operation-bounded authority to act on their behalf -- without giving up their master key and without trusting an intermediary. Gaming, subscription services, and automated trading strategies all benefit.

### Cross-frame information flow for MEV protection

Because TXPARAM allows a later frame to read the status of an earlier frame (param `0x15`), and because VERIFY frames can read transaction parameters, a smart account can implement MEV-resistant patterns. For example, a VERIFY frame could commit to a maximum slippage, and a post-op DEFAULT frame could check whether the execution frame's actual outcome (inspected via return data or state) exceeded that slippage, triggering a revert of the atomic batch if so.

More subtly, because the signature hash is canonical and covers everything except VERIFY data, a user can sign over exact execution parameters while a sponsor can attach their own VERIFY data (signature, paymaster logic) without affecting the user's commitment. This separation of concerns between "what the user authorized" and "who pays for it" is a foundation for MEV-aware transaction construction.

### Account recovery without social recovery contracts

A smart account's VERIFY logic could implement threshold signature schemes: 2-of-3 keys, where one key is the user's primary device, one is a hardware backup, and one is a social recovery guardian. Today this requires deploying and interacting with a dedicated social recovery contract. Frame transactions make it a property of the account itself. The VERIFY frame checks whether sufficient signatures are present in the frame data and calls APPROVE. Recovery is just a regular transaction with a different set of signatures.

### Programmable transaction fee markets

Because the payer is decoupled from the sender, and because the payer approval is explicit EVM code, frame transactions enable programmable fee markets. A canonical paymaster could implement a fee auction: "I will sponsor your transaction if you pay me X tokens, where X is determined by an on-chain oracle price feed." Or a DAO treasury could automatically sponsor transactions from its members, with gas costs deducted from their governance token allocations. The fee market becomes programmable at the application layer rather than fixed at the protocol layer.

### Batch signature verification for rollup sequencers

A rollup sequencer that receives many frame transactions could, in a future protocol upgrade, aggregate the VERIFY frames. Since VERIFY frame data is elided from the signature hash and is not introspectable by other frames, a future EIP could replace N individual VERIFY frames across N transactions with a single batch verification proof (e.g., a BLS aggregate signature or a SNARK proving all N signatures are valid). The frame transaction design does not enable this today, but it explicitly avoids closing the door. This is the kind of forward-compatible design that pays dividends years later.

### Programmable nonce schemes

The TXPARAM opcode notes that param `0x01` (nonce) "has a possible future extension to allow indices for multidimensional nonces." This is a sleeper feature. Multidimensional nonces would allow parallel transaction submission from a single account without nonce conflicts. Today, if you send two transactions from the same account, the second must wait for the first to be included (or you must manually manage nonce gaps). Multidimensional nonces would allow independent "nonce channels," enabling true parallel transaction submission. The frame transaction architecture is positioned to support this without a new transaction type.

### Smart account as a policy engine for institutional custody

Institutional custody today relies on off-chain policy engines (Fireblocks, Fordefi, etc.) that approve or reject transactions before signing. Frame transactions move this on-chain: a VERIFY frame can enforce arbitrary policies (whitelisted destinations, spending limits, time locks, multi-party approval) in the EVM itself. The policy is auditable, immutable (or upgradeable via governed proxy), and enforced by the protocol rather than by trust in a custodian's infrastructure. This is a meaningful improvement for institutional DeFi participation.

---

## 6. Suggested Changes to Increase Optionality

### 6.1 Add a `DELEGATE` mode (or reserve it explicitly)

The three existing modes (DEFAULT, VERIFY, SENDER) cover the common cases, but there is a missing primitive: executing code in the context of the sender's storage (like DELEGATECALL). Today, SENDER mode calls a target with `msg.sender = tx.sender`, but the target executes in its own storage context. A DELEGATE mode that executes the target's code in the sender's storage context would enable:

- In-place account upgrades without proxy patterns
- Library calls that modify account state directly
- Plugin systems where the account delegates specific operations to specialized contracts

If this is too complex for the initial EIP, reserve mode value `3` explicitly for DELEGATE with a note about future intent. Do not leave it as generic "reserved."

### 6.2 Add a frame-level return data opcode

The TXPARAM opcode can read a frame's status (success/failure via param `0x15`), but there is no way to read a frame's return data from a subsequent frame. Adding a `FRAMERETURNDATA` opcode (or a TXPARAM extension) that copies the return data of a prior frame would enable:

- Post-op frames that verify execution results (e.g., checking actual swap output against expected minimum)
- Validation frames that commit to expected outcomes
- Cross-frame data passing without relying on storage writes

This would make the frame abstraction significantly more powerful for composed operations. The atomic batch feature handles the "revert everything if something fails" case, but there is no "inspect what happened and decide" case. FRAMERETURNDATA fills that gap.

### 6.3 Reduce MAX_FRAMES and add a per-frame calldata cost multiplier

As discussed in the concerns section, MAX_FRAMES = 1000 is too large. Reduce it to 16 or 32. Additionally, consider a small per-frame overhead cost (e.g., 500 gas per frame beyond the first two) to discourage gratuitous frame splitting. The intrinsic cost of 15,000 gas covers the transaction envelope, but does not account for the per-frame overhead of mode switching, scope tracking, and journal management.

### 6.4 Make the ORIGIN behavior configurable per mode, or deprecate ORIGIN entirely

Rather than changing ORIGIN semantics (which breaks backward compatibility) or keeping them (which leaks information), consider deprecating ORIGIN for frame transactions entirely. Return zero or revert when ORIGIN is called inside a frame transaction. This is a clean break that forces contracts to use explicit TXPARAM calls for transaction metadata, and it avoids the per-frame ORIGIN variation that will confuse developers.

If full deprecation is too aggressive, at minimum define ORIGIN behavior for each mode explicitly in a table, and add test vectors covering all combinations.

### 6.5 Specify a frame transaction version field

The frame list format `[mode, target, gas_limit, data]` has no version or extension mechanism. If a future EIP needs to add a field to frames (e.g., a value field, an access list, a blob commitment), it would require a new transaction type or an awkward encoding change. Adding a version byte to the frame format (or to the transaction envelope) would allow backward-compatible frame extensions.

Alternatively, since `mode` already uses bitflags for configuration, reserve bits 12-15 for a version indicator. This is inexpensive and future-proofs the format.

### 6.6 Add explicit gas cost for TXPARAM(0x08) (signature hash)

Computing `compute_sig_hash(tx)` involves RLP-encoding the entire transaction (with VERIFY data elided) and hashing it. For a transaction with large frames, this could be expensive. The current specification sets the gas cost of TXPARAM at a flat 2 gas, which does not cover the cost of hashing a large transaction. Either:

- Specify that the signature hash is computed once and cached (making the 2 gas cost correct for subsequent reads), or
- Add a variable gas cost for param `0x08` proportional to the transaction size.

Without this, a malicious user could craft a transaction with near-maximum calldata and call TXPARAM(0x08) repeatedly to force expensive recomputation at minimal gas cost.

### 6.7 Define behavior for empty sender code with non-default signature types

The default code supports signature types `0x0` (secp256k1) and `0x1` (P256). What happens if a future EIP introduces signature type `0x2`? The default code reverts for unknown types, which means EOAs cannot use new signature schemes without deploying smart account code. Consider adding a precompile-based extensibility mechanism: the default code could check if a precompile at address `0x100 + signature_type` exists, and if so, delegate verification to it. This would allow new signature schemes to be added via new precompiles without changing the default code specification.

### 6.8 Clarify interaction with EIP-7702 delegations

The mempool section bans `CALL*/EXTCODE*` to addresses using EIP-7702 delegations during validation, "except for tx.sender default-code behavior." But what if `tx.sender` itself has an EIP-7702 delegation? Is the delegated code used instead of default code? What if the delegation target is a smart account that implements its own VERIFY logic? The interaction between EIP-7702 delegations and EIP-8141 frame transactions needs a dedicated section with explicit rules and examples.

---

## Summary Assessment

EIP-8141 is the most architecturally sound account abstraction proposal I have seen in the Ethereum ecosystem. It succeeds where previous attempts failed by making three correct decisions: putting validation in the protocol (not in a contract), isolating gas per frame (not pooling it), and designing for cryptographic agility from the start (not bolting it on later).

The main risks are complexity-driven: the interaction surface between frame modes, approval scopes, atomic batching, default code, mempool rules, and ORIGIN semantics is large. Every client team will need to implement this correctly, and the specification must be precise enough to prevent consensus-breaking implementation divergence. The current draft is thorough but has the gaps noted above (the APPROVE(0x0) inconsistency is likely a bug, the TXPARAM numbering gap is cosmetic, the atomic batch snapshot cost is real).

The mempool specification is the hardest part of this EIP and also the most important. If the mempool rules are too restrictive, the EIP is unusable for its most compelling use cases (paymaster-sponsored transactions). If they are too permissive, node operators will face DoS attacks and disable frame transaction propagation, fracturing the network. The canonical paymaster design is a good start, but the one-pending-transaction-per-non-canonical-paymaster limit will need real-world tuning.

I would prioritize: (1) fixing the apparent APPROVE(0x0) bug in the mempool section, (2) reducing MAX_FRAMES, (3) adding FRAMERETURNDATA for cross-frame inspection, and (4) writing a comprehensive test vector suite covering all mode/scope/atomic-batch combinations before this EIP advances beyond Draft status.

This is the kind of EIP that, if implemented correctly, makes the previous five years of account abstraction workarounds unnecessary. That is both its promise and its burden.

---

*"The grade is not encouragement. It is a timestamp."*
