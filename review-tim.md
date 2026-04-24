# EIP-8141 "Frame Transaction" -- Security & Cryptography Review

**Reviewer:** Tim Seaward (Drawaes)
**Date:** 2026-04-12
**Status:** Draft Review

---

## 1. What I Like

### The PQ migration path is the single most important thing Ethereum needs right now

I have been saying for years that the Ethereum community is dangerously underinvesting in post-quantum readiness. EIP-8141 is the first proposal I have seen that provides a genuinely credible, protocol-native off-ramp from secp256k1 ECDSA without requiring every existing account to perform an emergency migration. The "default code" mechanism that lets EOAs immediately participate in frame transactions -- including with P256 signatures -- is a pragmatic bridge that does not sacrifice the longer-term PQ goal. This is the correct architectural posture: make the secure path the easy path.

### Signature scheme agility is handled correctly

By pushing signature verification into user-defined EVM code rather than baking a specific PQ algorithm into the protocol, the design avoids the "pick the wrong NIST finalist" problem. If ML-DSA gets broken next year, or if the community wants to adopt a hash-based scheme like SPHINCS+ or a lattice scheme like Falcon, no hard fork is required. The frame transaction is a vessel; the cryptography is cargo. This is exactly the right separation of concerns.

### The VERIFY frame / APPROVE opcode design is elegant

Requiring explicit APPROVE calls rather than inferring validation success from a return value is a meaningful improvement over ERC-4337's validateUserOp pattern. It eliminates an entire class of accidental-approval bugs where a contract returns a truthy value without intending to authorize anything. The restriction that only `frame.target` can call APPROVE prevents confused-deputy attacks where a malicious library contract could approve on behalf of a sender it should not control.

### Signature hash elision for VERIFY frames is forward-looking

Eliding `frame.data` from the canonical signature hash is necessary (the signature cannot sign itself), but the rationale goes further -- it explicitly preserves the option for future signature aggregation. This is critical for BLS and future lattice-based aggregate schemes. The authors clearly thought about this.

### The mempool DoS model is taken seriously

The validation prefix concept, MAX_VERIFY_GAS cap, banned opcode list during validation, and the canonical paymaster code-matching approach are all evidence that the authors understand the adversarial mempool environment. The explicit exclusion of TIMESTAMP, BLOCKHASH, COINBASE, and similar environment-dependent opcodes from validation is exactly what is needed to prevent mass-invalidation attacks.

### Atomic batching solves a real security problem

The approve-then-swap pattern has caused enormous losses when the two operations can be separated. Atomic batching at the protocol level -- rather than requiring users to deploy multicall wrappers -- eliminates dangling approval risk for the most common DeFi interaction pattern. This is a genuine security improvement.

### ORIGIN semantics change is correct

Returning the frame caller rather than the original transaction sender for ORIGIN is the right call. The traditional ORIGIN has been a persistent footgun for access control, and this change further reduces the utility of relying on it.

---

## 2. What I Do Not Like / Concerns

### The default code P256 path derives addresses from raw public key hashes

The default code specifies that for signature type `0x1` (P256), the sender address must equal `keccak256(qx || qy)[12:]`. This means the public key is embedded directly in the address derivation, and the full public key must be transmitted with every transaction (128 bytes for the uncompressed point coordinates).

This has three problems:

1. **No point validation is specified.** The EIP does not mandate that `(qx, qy)` is a valid point on the P256 curve before using it. Invalid curve attacks against P256 are well-documented. If the P256VERIFY precompile does not perform full point validation (and the spec does not say it must in this context), an attacker could submit a signature against an invalid point and potentially forge authorization.

2. **Point compression is not supported.** Requiring both qx and qy adds 32 bytes per P256 transaction. Point compression would halve this. For a protocol that counts bytes carefully enough to publish detailed data-efficiency tables, this is an odd omission.

3. **The address derivation scheme is novel and unreviewed.** Using `keccak256(qx || qy)[12:]` for P256 keys creates a new address derivation scheme with no domain separation from the existing secp256k1 scheme (which uses `keccak256(pubkey_x || pubkey_y)[12:]` with the uncompressed point prefix stripped). This means if someone finds a P256 keypair whose public key, when hashed, collides with an existing secp256k1-derived address, they could claim that account. The probability is negligible given 160-bit addresses, but the lack of domain separation is sloppy from a cryptographic engineering perspective.

### The secp256k1 default code does not enforce canonical S values

The default code Python pseudocode for signature type `0x0` simply calls `ecrecover(sig_hash, v, r, s)` without checking that `s <= secp256k1n / 2`. The CanonicalPaymaster contract does enforce this (`SECP256K1N_DIV_2` check), but the default EOA code does not.

Yes, ecrecover will recover a valid address for both `s` and `n - s`, and since the check is against the recovered address, a malleable signature will still recover the correct address. But this means the same transaction can have two valid signature encodings, which creates a transaction hash malleability vector. The frame transaction hash includes the VERIFY data, so two encodings produce two distinct transaction hashes for the same logical transaction. This affects transaction tracking, deduplication, and any system that uses transaction hashes as identifiers.

The CanonicalPaymaster gets this right (line 56: `if (uint256(s) > SECP256K1N_DIV_2) revert InvalidSignature()`). The default code should match.

### MAX_FRAMES = 10^3 is too high

One thousand frames is an enormous attack surface. Each frame can have its own gas limit, its own target, and its own mode. The combinatorial complexity of validating, executing, and reasoning about the interactions between 1000 frames is immense. I would want to see formal analysis of the worst-case execution paths before accepting this limit. The atomic batching state snapshot mechanism, in particular, could create pathological memory consumption with deeply nested batch groups across hundreds of frames.

A limit of 16 or 32 would cover every realistic use case I can imagine (verify + deploy + N batch operations + post-op), and would be far easier to reason about from a DoS perspective.

### Transient storage semantics create a subtle footgun

Discarding TSTORE/TLOAD state between frames breaks a reasonable mental model where transient storage persists for the duration of a transaction. Developers who expect EIP-1153 semantics (transient storage lives for the whole transaction) will write verification logic that silently fails when data set in one frame is not available in the next. This should be more prominently warned about, and ideally there should be a mechanism for frames to explicitly opt into shared transient storage when needed.

### Gas isolation between frames prevents efficient gas usage

The design specifies that "unused gas from a frame is not available to subsequent frames." While this simplifies gas accounting and prevents a frame from consuming gas intended for another, it forces users to over-provision gas for every frame. In a sponsored transaction with 5 frames, the user must guess the gas for each frame independently. If frame 2 uses less gas than allocated, that gas is wasted from the perspective of frame 3 even if frame 3 needs it.

This creates an incentive to set generous gas limits on every frame, which in turn increases the maximum cost that must be reserved against the payer, reducing capital efficiency for paymasters.

### The ENTRY_POINT address (0xaa) collision with APPROVE opcode (0xaa)

Both the ENTRY_POINT precompile address and the APPROVE opcode share the value `0xaa`. While these exist in completely different namespaces (address space vs. opcode space), using the same numeric value for two security-critical components is needlessly confusing and will inevitably cause bugs in implementations. Rename one of them.

---

## 3. Security Issues

### CRITICAL: No replay protection across signature schemes

The signature hash is computed as `keccak(rlp(tx))` with VERIFY data elided. The hash does not include any domain separator for the signature scheme being used. If the same account is accessible via both secp256k1 (type 0x0) and P256 (type 0x1) -- which is possible since the signature type is in the VERIFY frame data that gets elided -- then the same signature hash is presented to both verification paths.

This is not immediately exploitable because the two curves use different key material, but it violates the principle of domain separation. A future signature scheme addition to the default code could inadvertently create cross-scheme replay if the same message is valid under two different schemes.

**Recommendation:** Include the signature type in the signature hash computation, or add an explicit domain separator.

### HIGH: VERIFY frame data is invisible to other frames but shapes execution

VERIFY frame data is elided from introspection by other frames (FRAMEDATALOAD/FRAMEDATACOPY return zero for VERIFY frames). This is correct for preventing signature leakage, but it means a paymaster in a later VERIFY frame cannot inspect the sender's signature to make informed decisions about sponsorship. More critically, it means the paymaster's own VERIFY frame data (which might contain its authorization signature) is also invisible to the sender's verification logic.

This creates an asymmetric information model where the sender must blindly trust the frame ordering and targets without being able to verify what data the paymaster will receive. The `frame.target` for VERIFY frames is part of the signature hash, so the sender does commit to *who* will verify, but not *what* they will verify. Combined with the fact that the paymaster's VERIFY frame data is malleable (added after the sender signs), a malicious bundler could substitute different paymaster authorization data.

The EIP acknowledges this in the rationale ("the input data to the sponsor is intentionally left malleable so it can be added onto the transaction after the sender has made its signature"), but the security implications of this malleability deserve deeper analysis. A malicious bundler that controls the paymaster authorization data could, for example, submit a valid paymaster signature that approves payment but with modified terms (different fee structure, different post-op behavior).

### HIGH: ecrecover precompile returns zero address on invalid input

The default code checks `if frame.target != ecrecover(sig_hash, v, r, s)`. If ecrecover fails (returns the zero address), this check passes if `frame.target` is also the zero address. While the constraints require `len(tx.sender) == 20`, they do not explicitly exclude the zero address. If a frame targets address(0) and ecrecover fails, the comparison succeeds, and APPROVE would be called for a transaction "from" the zero address.

The constraint `assert len(tx.frames[n].target) == 20 or tx.frames[n].target is None` checks length, not value. An explicit check that `frame.target != address(0)` is needed, or equivalently that `ecrecover` did not return zero.

### HIGH: CanonicalPaymaster uses verbatim assembly for custom opcodes

The CanonicalPaymaster contract (lines 98-109) uses `verbatim_0i_1o` and `verbatim_0i_0o` to emit raw bytecode for TXPARAM and APPROVE. This is brittle:

1. The verbatim facility is not standardized and its behavior may vary across Solidity versions.
2. The raw hex sequences (`hex"60006008b0"` and `hex"600160006000aa"`) are not self-documenting and are trivially easy to get wrong. A single byte error produces a contract that silently does the wrong thing.
3. There is no compile-time verification that these sequences are correct.

For a contract that is intended to be deployed canonically and matched by code hash across all nodes, this fragility is concerning. A reference implementation in raw bytecode (like a Huff contract or hand-assembled EVM) with a formal verification proof would be more appropriate.

### MEDIUM: Nonce increment timing creates a window for parallel execution attacks

The nonce is checked at the start (`tx.nonce == state[tx.sender].nonce`) but only incremented when APPROVE(0x2) or APPROVE(0x3) is called. Between the initial check and the increment, multiple transactions with the same nonce could pass the initial check. The mempool rule of one transaction per sender mitigates this for the public mempool, but private mempools and block builders could exploit this window.

If a block builder includes two frame transactions from the same sender with the same nonce, and the first one's APPROVE increments the nonce, the second one would fail the initial nonce check. But during parallel transaction validation (which some clients do for performance), both could pass the nonce check simultaneously.

**Recommendation:** Explicitly specify that frame transactions from the same sender must be serialized, not just that nonce checks must pass.

### MEDIUM: Warm/cold state journal sharing across frames leaks information

The specification states that "for the purposes of gas accounting of warm / cold state status, the journal of such touches is shared across frames." This means a VERIFY frame can probe whether storage slots or addresses are warm (by observing gas costs) to learn information about what previous frames accessed. In the adversarial paymaster model, a malicious paymaster VERIFY frame could use gas metering to infer information about the sender's verification logic.

More practically, this creates a side channel: the gas cost of the paymaster's VERIFY frame depends on what the sender's VERIFY frame accessed, which leaks information about the sender's contract internals.

### MEDIUM: No explicit protection against signature reuse across chain forks

The signature hash includes `chain_id`, which prevents cross-chain replay. However, during a chain fork (contentious hard fork), both chains share the same chain_id until one of them changes it. Frame transactions signed before a fork are valid on both chains. This is the same situation as existing transactions, but the much richer frame structure means the consequences of replay may be more severe (e.g., a multi-frame atomic batch with specific DeFi interactions could cause catastrophic losses if replayed on a fork with different state).

### LOW: The banned opcode list during validation may be incomplete

The banned list includes BALANCE and SELFBALANCE, but does not ban EXTCODEHASH, EXTCODESIZE, or EXTCODECOPY for arbitrary addresses (only restricting them to existing contracts). A contract's code hash could change between validation and inclusion if the contract self-destructs and is redeployed (SELFDESTRUCT is banned in validation but could happen in a previous transaction in the same block). The spec partially addresses this by only allowing CALL*/EXTCODE* to existing contracts, but the definition of "existing" is at validation time, not inclusion time.

### LOW: The 100,000 gas limit for MAX_VERIFY_GAS may be insufficient for PQ signatures

Post-quantum signature verification is computationally expensive. ML-DSA-44 verification costs are estimated at roughly 50,000-80,000 gas in optimized EVM implementations. Adding the overhead of contract dispatch, storage reads for the account's public key, and any additional validation logic, 100,000 gas may be tight for PQ schemes. If future PQ algorithms require more gas, this constant becomes a bottleneck that requires a hard fork to change.

**Recommendation:** Either increase MAX_VERIFY_GAS or make it a protocol parameter that can be adjusted via governance.

---

## 4. Usage Scenarios That Excite Me

### Hardware security module integration

Frame transactions could enable HSM-backed accounts where the VERIFY frame delegates to a contract that checks signatures from a specific HSM attestation scheme. This would allow institutional custody solutions to operate natively on Ethereum without wrapping everything in a multisig. The arbitrary signature verification means you could verify TPM attestations, YubiKey signatures, or cloud KMS signatures directly.

### Threshold signature schemes without trusted setup

A VERIFY frame could implement a threshold signature verification contract (e.g., Shamir-based or Frost-based) where k-of-n signers must collaborate to produce a valid signature. Unlike current multisig wallets that require N on-chain transactions or a trusted aggregator, the threshold scheme produces a single compact signature that the VERIFY frame validates. This is strictly better than existing multisig approaches for both gas efficiency and privacy (the number of signers is not revealed on-chain).

### Social recovery with cryptographic guarantees

The VERIFY frame could implement a social recovery scheme where the "signature" is actually a set of attestations from guardian accounts. Unlike existing social recovery wallets that rely on contract storage for guardian lists, the frame transaction model allows the guardian set to be committed to in the account's code, making it immutable and verifiable.

### Time-locked and conditional transactions

While the mempool bans TIMESTAMP in validation, post-approval frames can use any opcode. This means a frame transaction could include a VERIFY frame for authorization, followed by SENDER frames that check time conditions, oracle prices, or other state before executing. The atomic batching ensures that either all conditions are met and the operations execute, or nothing happens.

### Gasless onboarding with verifiable sponsorship

The sponsored transaction model (Example 3 in the EIP) enables genuinely gasless onboarding where a new user deploys their smart account, verifies their first transaction, and has gas paid by a sponsor -- all in a single transaction. The frame structure makes the sponsorship relationship explicit and auditable, unlike current meta-transaction relayers where the sponsorship logic is opaque.

---

## 5. Additional Capabilities People Might Not Be Thinking About

### Programmable transaction introspection enables MEV protection

The TXPARAM and FRAMEDATALOAD opcodes give verification logic full visibility into the transaction structure. A smart account could implement MEV-aware validation that inspects subsequent frames to ensure the transaction has not been modified by a searcher. For example, a VERIFY frame could check that the execution frames match an expected pattern and reject the transaction if unexpected frames have been appended.

### The frame model is a natural fit for intent-based architectures

Frames map cleanly onto the intent/solver paradigm: a VERIFY frame commits to an intent (signed by the user), and subsequent DEFAULT/SENDER frames contain the solver's solution. The solver's execution is constrained by the verification logic but flexible in implementation. This is a strictly better architecture than current intent protocols that rely on off-chain solvers with on-chain settlement contracts.

### Cross-frame state journal sharing enables gas-efficient batched operations

The warm/cold journal sharing means that the first frame that touches a storage slot or address pays the cold cost, and all subsequent frames benefit from the warm cost. For batched DeFi operations that touch the same contracts (e.g., multiple swaps on the same DEX), this provides gas savings that are not achievable with separate transactions.

### The APPROVE return data could enable rich authorization protocols

APPROVE takes offset and length parameters for return data. This return data is currently underspecified but could carry authorization metadata -- for example, spending limits, time bounds, or conditional permissions. Future extensions could use this return data to implement fine-grained authorization that flows between frames.

### Canonical paymaster as a DeFi primitive

The canonical paymaster's timelocked withdrawal mechanism creates a contract pattern that is useful beyond gas payment. Any protocol that needs "funds committed for a purpose with delayed withdrawal" -- staking, escrow, insurance -- could use similar patterns. The fact that the canonical paymaster is recognized by code match rather than address means it becomes a composable building block.

### Enabling stateless verification for light clients

Because the validation prefix is constrained to access only the sender's storage and code (plus known deterministic deployers), a light client can verify transaction validity by requesting a small state proof covering only those specific storage slots. This is dramatically less data than would be needed for unconstrained validation, and it makes light client mempool participation feasible.

---

## 6. Changes That Could Increase Optionality Further

### Add an explicit signature scheme registry to the default code

Rather than hardcoding signature types 0x0 (secp256k1) and 0x1 (P256) in the default code, define a mapping from signature type byte to precompile address or verification contract. This would allow new signature schemes (ML-DSA, Falcon, SPHINCS+) to be added to EOA default code via precompile deployment without modifying the frame transaction specification. Reserve type bytes 0x00-0x0F for NIST-standardized schemes, 0x10-0x1F for Ethereum-specific schemes, and 0x20-0xFF for experimental/custom schemes.

### Allow optional gas sharing between frames via an explicit mechanism

Instead of strict gas isolation, allow frames to specify a "gas pool" identifier. Frames with the same pool ID share a gas budget. This preserves the simplicity of isolated gas accounting for the common case while enabling efficient gas usage for complex multi-frame transactions. The VERIFY frames would always be in their own pool (for DoS resistance), but SENDER frames could share.

### Add a frame-level access list for state dependencies

While the EIP rationale explains why a transaction-level access list is omitted, a frame-level mechanism that declares which storage slots a VERIFY frame will read would enable parallel validation. Validators could check whether two pending transactions' validation prefixes have overlapping state dependencies and, if not, validate them in parallel. This becomes important at scale.

### Include an explicit "capabilities" bitfield in the transaction envelope

Add a bitfield to the transaction that declares which features are used (PQ signatures, paymaster, atomic batching, deployment). This enables fast filtering by nodes, wallets, and block builders without parsing the full frame structure. It also provides a natural extension point for future capabilities.

### Define a standard "upgrade" frame mode for account migration

Add a mode specifically for migrating an EOA to a smart contract account within the same transaction. Currently, deployment is handled via a DEFAULT frame calling a known deployer, but an explicit UPGRADE mode could enforce additional safety properties: the new code must be at the sender's address, the old key must authorize the upgrade, and the upgrade is atomic with the rest of the transaction.

### Consider supporting compressed points and compact signatures

For data efficiency, support SEC1 compressed point encoding (33 bytes instead of 64 for P256 public keys) and compact ECDSA signatures (64 bytes instead of 65 by encoding recovery bit in the S value). For PQ schemes, support application-specific compression where it exists (e.g., Falcon signatures have known compression techniques). Every byte saved is multiplied across every transaction forever.

### Add a mechanism for VERIFY frames to communicate with each other

Currently, VERIFY frame data is invisible across frames and transient storage is discarded between frames. This means the sender's VERIFY frame and the paymaster's VERIFY frame cannot communicate. A narrow, well-defined communication channel (e.g., a shared read-only buffer set by APPROVE's return data) would enable richer authorization protocols where the paymaster's authorization depends on properties of the sender's authorization, without introducing the full complexity of shared mutable state.

### Specify behavior under concurrent execution models

As Ethereum clients move toward parallel transaction execution, the frame transaction's multi-step validation-then-execution model needs explicit concurrency semantics. Can two frame transactions from different senders be validated in parallel? Can the validation of one frame transaction overlap with the execution of another? The current spec is written for sequential execution, but it should explicitly address parallelism to avoid implementation divergence.

---

## Summary Assessment

EIP-8141 is the most significant proposed change to Ethereum's transaction model since EIP-1559. It addresses a genuine and urgent need (PQ migration), provides a clean abstraction (frames with modes), and takes security seriously (mempool DoS resistance, canonical paymaster, validation restrictions).

The critical issues -- lack of domain separation across signature schemes, zero-address ecrecover vulnerability, and S-value malleability in default code -- are fixable without architectural changes. The design concerns about MAX_FRAMES, gas isolation, and P256 point validation warrant discussion but do not invalidate the approach.

My primary recommendation is to tighten the default code's cryptographic hygiene (canonical S values, explicit zero-address checks, P256 point validation, domain separators), reduce MAX_FRAMES to something defensible, and add a formal security model for the VERIFY frame data malleability that the paymaster model depends on.

This EIP deserves to move forward. The perfect should not be the enemy of the good, and Ethereum's PQ clock is ticking.

-- Tim
