# EIP-8141 "Frame Transaction" Security Audit Report

**Date**: 2026-04-12
**Specification**: EIP-8141 (Draft) -- Frame Transaction
**Authors**: Vitalik Buterin, lightclient, Felix Lange, Yoav Weiss, Alex Forshtat, Dror Tirosh, Shahaf Nacson, Derek Chiang
**Auditor**: Automated multi-domain EVM security audit (5 parallel specialist agents) + manual design review
**Methodology**: 500+ checklist items from evm-audit-skills applied across 7 domains, supplemented by architectural analysis of the EIP's stated goals vs. delivered mechanisms

---

## Executive Summary

EIP-8141 introduces a new transaction type (`0x06`) enabling native account abstraction with arbitrary signature schemes, multi-frame execution, and gas sponsorship. It is the most general account abstraction proposal Ethereum has seen at the protocol level. However, the audit reveals a significant gap between its ambitions (PQ migration, full AA) and its constraints (100K verify gas, single nonce, rigid per-frame gas budgets).

The audit identified **55 deduplicated findings** from 83 raw checklist findings across 5 specialist domains, plus 6 additional findings from architectural design review.

| Severity | Count |
|----------|-------|
| Critical | 1     |
| High     | 12    |
| Medium   | 25    |
| Low      | 14    |
| Info     | 3     |
| **Total**| **55**|

The most severe issues cluster around: (1) a specification inconsistency where mempool rules reference an undefined APPROVE scope, (2) underspecified gas collection semantics in APPROVE, (3) integer overflow potential in gas accounting, (4) signature malleability in the default EOA code, (5) mempool DoS vectors from the late nonce increment design, and (6) a verification gas cap that contradicts the EIP's own PQ migration motivation.

---

## Critical Findings

### [C-1] APPROVE Scope 0x0 Referenced in Mempool Rules Is Undefined -- Specification Self-Contradiction

**Severity**: Critical
**Category**: evm-audit-access-control, evm-audit-erc4337
**Location**: Structural Rules (item 3); APPROVE Scope Operand; CanonicalPaymaster.sol
**Sources**: GEN-2, AA-14

**Description**:
The mempool structural rules state: "`only_verify` must call `APPROVE(0x0)`" and "`pay` must call `APPROVE(0x1)`." However, the APPROVE opcode defines only scopes `0x1` (execution), `0x2` (payment), and `0x3` (both). Scope `0x0` explicitly causes an exceptional halt: "Any other value results in an exceptional halt."

This creates a cascading failure:
1. `only_verify` calling `APPROVE(0x0)` triggers an exceptional halt -- the VERIFY frame fails, the transaction is invalid.
2. `pay` calling `APPROVE(0x1)` means "approval of execution," which requires `frame.target == tx.sender`. But the paymaster is NOT the sender. APPROVE reverts.
3. The canonical paymaster bytecode pushes `0x01` to the stack before APPROVE, confirming the inconsistency.

The entire sponsored transaction flow (canonical paymaster prefix) is broken as specified. No valid transaction can use the `[only_verify, pay]` prefix.

**Proof of Concept**:
```
Frame 0 (only_verify): APPROVE(0x0) -> exceptional halt -> tx invalid
Frame 1 (pay):         APPROVE(0x1) -> frame.target != tx.sender -> revert -> tx invalid
```

**Recommendation**:
Almost certainly a numbering error. Fix to:
- `only_verify` must call `APPROVE(0x1)` (sender execution approval)
- `pay` must call `APPROVE(0x2)` (payment approval)
- Update canonical paymaster bytecode to push `0x02`

---

## High Findings

### [H-1] MAX_VERIFY_GAS (100K) Is Too Small for the EIP's Own PQ Ambitions

**Severity**: High
**Category**: evm-audit-erc4337, design-review
**Location**: Mempool Constants; Abstract and Motivation sections
**Sources**: Design review, DOS-2

**Description**:
The EIP's primary motivation is post-quantum migration: "a native off-ramp from the elliptic curve based cryptographic system... to post-quantum (PQ) secure systems." But `MAX_VERIFY_GAS = 100,000` is insufficient for most PQ signature schemes:

- **Dilithium (ML-DSA) verification in EVM**: ~200-500K gas (lattice-based, NIST standardized as FIPS 204)
- **SPHINCS+ verification in EVM**: ~500K+ gas (hash-based, stateless)
- **Groth16/fflonk pairing checks for ZK-based authentication**: ~170-220K gas

The only signatures that fit in 100K gas are ECDSA (~3K via ecrecover) and P256 (~3.5K via the secp256r1 precompile) -- the same two curves we already have.

Separately (from the DoS perspective), 100K gas of free validation per sender creates an asymmetric cost: submitting transactions is free on the p2p network, but nodes must spend up to 100K gas of computation per transaction. With 10,000 Sybil accounts, that's 1 billion gas equivalent per flood cycle. The one-tx-per-sender rule raises the cost but doesn't eliminate it.

The gas cap sits in an awkward position: too large from a DoS perspective (nodes do expensive work for free), and too small from a functionality perspective (excludes the PQ schemes that motivate the EIP).

**Proof of Concept**:
A user attempting Dilithium signature verification in the VERIFY frame would need ~300K gas minimum. The validation is rejected by mempool rules before it can even be simulated. The EIP enables signature scheme abstraction but immediately constrains it to the same two curves we already have.

**Recommendation**:
Either raise substantially and pair with stronger anti-DoS measures (per-peer validation gas tracking, minimum sender ETH balance), or reduce to ~50K and be explicit that PQ migration requires future precompiles. The EIP should be honest about this gap.

---

### [H-2] Integer Overflow in sum(frame.gas_limit) with MAX_FRAMES=1000

**Severity**: High
**Category**: evm-audit-precision-math
**Location**: Gas Accounting section
**Source**: PA-01

**Description**:
`tx_gas_limit = FRAME_TX_INTRINSIC_COST + calldata_cost + sum(frame.gas_limit)`. With `MAX_FRAMES = 10^3` and no explicit upper bound on `frame.gas_limit`, two frames with `gas_limit = 2^255` each cause the sum to overflow to 0 in 256-bit arithmetic. The attacker pays near-zero fees while each frame individually claims enormous gas.

**Recommendation**: Constrain `frame.gas_limit` to `uint64`. Add: `assert sum(frame.gas_limit) <= 2^63 - 1`.

---

### [H-3] APPROVE "Collect Total Gas Cost" Is Critically Underspecified

**Severity**: High
**Category**: evm-audit-precision-math, evm-audit-access-control
**Location**: APPROVE Behavior (scope 0x2 and 0x3)
**Sources**: PA-13, GEN-1

**Description**:
Two issues compound:
1. Scope `0x2` says "collect the total gas cost from the account" without specifying which account. Scope `0x3` explicitly says "from `frame.target`." Implementations may incorrectly collect from `tx.sender` instead of the paymaster.
2. It is unspecified whether the collected amount is `tx_gas_limit * max_fee_per_gas` (upfront max) or `tx_gas_limit * effective_gas_price`. Following EIP-1559 precedent, the max should be collected upfront with a refund of the difference -- but no refund mechanism for the fee overpayment is specified.

**Recommendation**: Replace "from the account" with "from `frame.target`" in scope `0x2`. Specify: collect `tx_gas_limit * max_fee_per_gas` upfront, refund `(max_fee - effective_price) * tx_gas_limit` to payer after execution.

---

### [H-4] ECDSA Signature Malleability in Default Code -- No Low-s Enforcement

**Severity**: High
**Category**: evm-audit-signatures
**Location**: Default code, SECP256K1 branch
**Sources**: SIG-1, PA-09

**Description**:
The default code calls `ecrecover(sig_hash, v, r, s)` without enforcing `s <= secp256k1.n / 2`. For any valid `(v, r, s)`, a second valid signature `(v^1, r, n-s)` exists. Since VERIFY frame data is elided from `sig_hash`, any mempool observer can flip the signature, producing a distinct but equally valid transaction. This breaks transaction hash uniqueness and interferes with off-chain tracking systems.

**Recommendation**: Enforce `s <= SECP256K1_N_HALF` and restrict `v` to {27, 28}, consistent with EIP-2.

---

### [H-5] ecrecover Returns address(0) for Invalid Signatures -- Default Code Does Not Check

**Severity**: High
**Category**: evm-audit-signatures
**Location**: Default code, SECP256K1 branch
**Source**: SIG-2

**Description**:
`ecrecover` returns `address(0)` for invalid inputs. The default code checks `frame.target != ecrecover(...)` -- if `tx.sender = address(0)` (with null target resolving to sender), an invalid signature passes verification. The zero address holds substantial burned/stuck ETH on mainnet.

**Recommendation**: Add `if recovered == address(0): revert()`. Reject `tx.sender == address(0)` as a static constraint.

---

### [H-6] Nonce Increment Inside APPROVE Creates Zero-Cost Validation Grinding

**Severity**: High
**Category**: evm-audit-erc4337, evm-audit-dos
**Location**: APPROVE Behavior; Behavior section (nonce check)
**Sources**: AA-1, SIG-7, GEN-6

**Description**:
The nonce is checked at transaction start but only incremented inside APPROVE. If validation fails before APPROVE, the nonce is not consumed, the transaction is not included on-chain, and the sender pays nothing. An attacker deploying many cheap smart accounts can force nodes to repeatedly simulate up to `MAX_VERIFY_GAS` of validation per account without any cost. The one-tx-per-sender rule raises Sybil cost but doesn't eliminate it.

Additionally, the nonce model is purely sequential (single nonce per sender). The mempool rule "at most one pending frame transaction per sender" (line 654) means frame transactions inherit the exact same sequential constraint as EOAs. This contradicts the "realize the original vision of account abstraction" claim. Real AA systems (ERC-4337) support 2D nonces for parallelism. Line 227 hints at this: "0x01 has a possible future extension to allow indices for multidimensional nonces" -- but "possible future extension" is not a design, it's a TODO. A multisig where multiple signers submit independently cannot have concurrent pending transactions.

**Recommendation**: Increment nonce at start of processing (matching legacy transactions), or add explicit rate-limiting: reject re-simulation of the same `(sender, nonce)` pair that previously failed within a cooldown window. For concurrency, define 2D nonce semantics now rather than deferring.

---

### [H-7] TXPARAM(0x06) Max Cost Overflow Breaks Paymaster Balance Reservation

**Severity**: High
**Category**: evm-audit-precision-math
**Location**: TXPARAM opcode, param 0x06
**Source**: PA-02

**Description**:
`TXPARAM(0x06, 0)` returns `tx_gas_limit * max_fee_per_gas + blob_fees`. Without bit-width constraints on fee fields, `2^128 * 2^128 = 2^256 = 0 mod 2^256`. A paymaster sees `max_cost = 0` and approves a transaction it cannot cover. Additionally, "basefee=max" is ambiguous -- max of what range?

**Recommendation**: Constrain `max_fee_per_gas` and `max_fee_per_blob_gas` to `uint64`. Specify that overflow makes the transaction invalid. Clarify the "basefee=max" semantics.

---

### [H-8] Deploy Frame Allows Front-Running of Account Deployment

**Severity**: High
**Category**: evm-audit-erc4337
**Location**: Behavior section (deploy frame); Mempool structural rules
**Source**: AA-8

**Description**:
The deploy frame executes BEFORE authentication. An attacker observing a pending frame transaction with a deploy frame can front-run by calling the deployer with the same initcode/salt, deploying code at the sender's address first. The user's transaction then fails.

**Recommendation**: Add a protocol-level check: if deploy frame is present, `tx.sender` MUST have no code. Add mempool rule verifying the deploy frame's CREATE2 address matches `tx.sender`.

---

### [H-9] Canonical Paymaster Is a Single Point of Failure Disguised as Decentralization

**Severity**: High
**Category**: evm-audit-erc4337, design-review
**Location**: Canonical Paymaster Exception
**Sources**: AA-7, DOS-3, Design review

**Description**:
The spec states: "The canonical paymaster is not a singleton deployment. Many instances may be deployed." But they're all recognized by exact runtime code match -- they're functionally identical. The canonical paymaster is a single codebase that the entire network must trust. The "many instances" framing is misleading: it's one contract design with multiple deployment addresses.

This creates multiple compound risks:
1. **Bug = total failure**: If a vulnerability is found in the canonical paymaster code, every instance is simultaneously vulnerable. There is no upgrade path -- the code hash IS the identity. Changing the code means it's no longer canonical.
2. **Attacker-deployed instances**: Anyone can deploy identical code with their own `owner`, deposit ETH, get many transactions accepted via balance reservation, then drain via delayed withdrawal. All pending transactions become invalid.
3. **Cross-node inconsistency**: `reserved_pending_cost(paymaster)` is maintained locally per node. An attacker submits N transactions against the same paymaster to N different nodes simultaneously. Each passes locally, but the paymaster only has balance for a fraction.
4. **Reentrancy in withdrawal**: The canonical paymaster's `executeWithdrawal()` performs a low-level `call{value: amount}("")` to transfer ETH. A contract recipient could re-enter `requestWithdrawal` during the callback, confusing the node's `pending_withdrawal_amount` tracking.

**Recommendation**: Add `MAX_PENDING_TXS_PER_CANONICAL_PAYMASTER` cap. Mandate balance re-checks at block boundaries. Prohibit canonical paymasters behind proxies. Add reentrancy guard to `executeWithdrawal`. Add `cancelWithdrawal()` mechanism. Acknowledge the centralization vector explicitly in Security Considerations.

---

### [H-10] ENTRY_POINT at address(0xaa) Is a Phantom Address with Opcode Collision

**Severity**: High
**Category**: evm-audit-assembly, design-review
**Location**: Constants table; Behavior section
**Sources**: PA-05, Design review

**Description**:
`ENTRY_POINT = address(0xaa)` is used as the caller for DEFAULT and VERIFY frames, but there is no specification for deploying code at this address or for what happens when contracts interact with it. The `APPROVE` opcode is also `0xaa` -- an odd collision.

This is problematic because:
1. **APPROVE collision**: If a frame sets `target = 0xaa` and code exists at ENTRY_POINT, APPROVE succeeds because `ADDRESS == frame.target`. This could allow unauthorized approval.
2. **Phantom caller**: Contracts that check `msg.sender.code.length > 0` will get `false` for calls from ENTRY_POINT (no code deployed there). Any contract that tries to callback `msg.sender` during a DEFAULT frame will call an empty address.
3. **Tooling confusion**: The same hex value `0xaa` meaning both an opcode and a system address will confuse debuggers, decompilers, and static analysis tools.

**Recommendation**: Use different values. E.g., `ENTRY_POINT = address(0x8141)` or reassign the APPROVE opcode number. Specify whether ENTRY_POINT should have code deployed and what that code does (if anything).

---

### [H-11] Explicit sender Field Enables Nonce Oracle and Mempool State Read Attacks

**Severity**: High
**Category**: evm-audit-dos, design-review
**Location**: Transaction payload (line 51); Behavior section (line 269)
**Source**: Design review

**Description**:
The transaction payload includes `sender` as an explicit field, not derived from a signature. This is fundamentally different from all existing Ethereum transaction types where the sender is recovered from the ECDSA signature. Anyone can set `sender` to any address -- the VERIFY frame is what authenticates it.

This means:
1. **Nonce oracle**: An attacker can learn any account's nonce by submitting a frame transaction with that account as sender and observing whether the nonce check (`tx.nonce == state[tx.sender].nonce`) passes or fails. While nonces are publicly queryable, this forces the receiving node to perform a state read for an attacker-chosen address before confirming the sender is legitimate.
2. **Mempool spam**: A malicious actor can spam transactions claiming to be from high-value accounts. Each requires the node to perform a state lookup for the claimed sender's nonce before it can reject.
3. **State read amplification**: The node does `state[tx.sender].nonce` for every received frame transaction. With rapid submission of transactions claiming different senders, this becomes an I/O amplification attack.

**Proof of Concept**:
Attacker submits 10,000 frame transactions per second, each claiming a different `tx.sender`. Each requires a state read for nonce verification before the node can reject. The node's state trie I/O becomes the bottleneck.

**Recommendation**: Add a lightweight pre-validation step before the nonce lookup -- e.g., require a small proof-of-work on the transaction, or rate-limit per source IP/peer. Require that `tx.sender` is an existing account (has nonce > 0 or code) to prevent enumeration attacks on virgin addresses.

---

### [H-12] Canonical Paymaster Balance Reservation Is Node-Local -- Cross-Node Inconsistency

**Severity**: High
**Category**: evm-audit-dos
**Location**: Canonical Paymaster Exception; Revalidation
**Source**: DOS-3

**Description**:
`reserved_pending_cost(paymaster)` is maintained locally per node. An attacker submits N transactions against the same paymaster to N different nodes simultaneously. Each passes locally, but the paymaster only has balance for a fraction. On inclusion, most become invalid.

**Recommendation**: Add `MAX_PENDING_TXS_PER_CANONICAL_PAYMASTER` cap. Apply safety margin on balance checks.

---

## Medium Findings

### [M-1] ORIGIN Redefinition Is a Backwards Compatibility Bomb

**Sources**: GEN-4, AA-6, PA-16, Design review

**Description**:
Line 286: "The ORIGIN opcode returns frame `caller` throughout all call depths." This means ORIGIN returns `ENTRY_POINT` (0xaa) for DEFAULT/VERIFY frames and `tx.sender` for SENDER frames -- changing mid-transaction as frames execute.

The Backwards Compatibility section acknowledges this "is consistent with the precedent set by EIP-7702" but then says "Contracts that rely on ORIGIN = CALLER for security checks (a discouraged pattern) may behave differently." This understates the impact.

`tx.origin` is used in the wild for:
- Reentrancy guards (`require(tx.origin == msg.sender)`)
- Bot protection / MEV mitigation
- Some access control patterns

Having ORIGIN return different values across frames in the same transaction is **unprecedented**. EIP-7702 did not make ORIGIN vary within the same transaction -- it changed who `tx.origin` refers to, but it remained consistent within one transaction. Under EIP-8141:

- A contract called from a DEFAULT frame sees `ORIGIN = 0xaa` (ENTRY_POINT)
- The same contract called from a SENDER frame sees `ORIGIN = tx.sender`
- Same transaction, same contract, two different `ORIGIN` values

For SENDER frames specifically, `ORIGIN == tx.sender == msg.sender` when the sender calls a contract directly. This means `require(tx.origin == msg.sender)` passes for smart accounts, allowing flash-loan-like atomicity across multiple SENDER frames -- a new dimension of execution context that existing security analyses don't account for.

**Recommendation**: Consider making `ORIGIN` consistently return `tx.sender` across all frames. If per-frame variation is intentional, provide strong rationale, add explicit warnings for contract developers, and elevate from Backwards Compatibility to Security Considerations with concrete examples.

---

### [M-2] VERIFY Frame Data Elision Creates Broad Malleability Surface

**Sources**: SIG-5, AA-2, Design review

**Description**:
ALL data from ALL VERIFY frames is elided from the signature hash and hidden from other frames' introspection. The EIP acknowledges this is intentional (lines 669-673) for aggregation and sponsor workflows. But this creates concrete attack vectors:

1. **Sponsor griefing**: Anyone who intercepts a signed frame transaction can replace the sponsor's VERIFY frame data with garbage. The `frame.target` is still bound (the sender chooses the sponsor address explicitly), but the sponsor's validation will fail, making the entire transaction invalid.

2. **Custom smart account context manipulation**: For the default code (ECDSA/P256), this is fine -- the signature self-authenticates. But for custom smart accounts that use VERIFY data for additional context (e.g., a timestamp, a price feed reference, a session key selector), that context is unsigned and malleable. A man-in-the-middle can substitute different VERIFY data, potentially causing different validation paths.

3. **Paymaster parameter manipulation**: If a paymaster's validation logic uses its frame data to determine payment terms (fee token type, exchange rate), the attacker can substitute more favorable terms. The paymaster's VERIFY frame data is not covered by the sender's signature.

4. **Transaction ID instability**: The tx hash either includes malleable data (unstable) or excludes it (non-unique). Neither is ideal.

**Recommendation**: Explicitly warn paymaster/smart account implementers that their VERIFY frame data is malleable and MUST NOT encode any user-committed parameters. Consider adding a TXPARAM that returns a hash of ALL frame data (including VERIFY) so smart accounts can optionally commit to the full transaction. Document risks in Security Considerations.

---

### [M-3] P256 Signature Malleability -- No Low-s Enforcement
**Source**: SIG-3, AA-13
P256VERIFY does not enforce `s <= P256.n / 2`. Combined with VERIFY data elision, observers can substitute malleable counterparts.

### [M-4] Cross-Chain Replay -- chain_id Enforcement Not Explicit
**Source**: SIG-6, AA-5
`chain_id` is in the signed data but the spec does not explicitly require `tx.chain_id == CHAIN_ID`. Using the canonical sig_hash is "strongly recommended" not mandatory.

### [M-5] Gas Refund Formula Ambiguity -- Potential Underflow
**Source**: PA-03
`refund = sum(frame.gas_limit) - total_gas_used` underflows if `total_gas_used` includes intrinsic/calldata costs not covered by `sum(frame.gas_limit)`.

### [M-6] P256 Public Key Validation Ordering Gap
**Source**: SIG-4, PA-12
Address derived from `keccak(qx||qy)` BEFORE P256VERIFY validates the key is on-curve. Relies on precompile for validation without stating the dependency.

### [M-7] Calldata Cost Calculation Lacks Precise Definition
**Source**: PA-08
`calldata_cost(rlp(tx.frames))` is ambiguous -- which exact bytes? Consensus-splitting risk between client implementations.

### [M-8] Scope Bit Numbering Inconsistency -- Consensus Risk
**Source**: PA-11, GEN-16
Mode Flags table says "bits 9-10" but code uses `(mode >> 8) & 3` (bits 8-9). Implementations following different spec sections compute different values. This is a consensus-splitting risk.

### [M-9] Cross-Frame Data Leakage -- Paymaster Can Read User Operations
**Source**: GEN-7
FRAMEDATALOAD/FRAMEDATACOPY let VERIFY frames read SENDER frame data. A paymaster can inspect user swap parameters (token, amount, slippage) and condition approval on MEV extractability. The paymaster validation runs BEFORE the user's operations, giving it first-mover advantage.

### [M-10] Storage Isolation Between Frames Is Asymmetric -- Transient Storage Breaks Reentrancy Guards

**Sources**: PA-15, GEN-14, DOS-15, AA-12, Design review

**Description**:
The frame interaction model creates an asymmetric isolation boundary:
- Regular storage warmth **persists** across frames (frame N warming a slot benefits frame N+1)
- Transient storage is **reset** between frames

This means patterns that use transient storage for cross-operation communication break across frame boundaries. A smart account that sets a transient reentrancy lock in the VERIFY frame won't have it visible in the SENDER frame. More concretely: a contract that uses EIP-1153 `TSTORE`-based reentrancy locks set in frame N will find them invisible in frame N+1. The contract can be re-entered via a separate frame without the guard firing.

This is an unusual semantic that will surprise developers. EIP-1153 was designed with the assumption that transient storage persists for the full transaction lifetime. Frame transactions break this assumption.

**Recommendation**: Document as a prominent security consideration. Consider preserving transient storage across frames within atomic batches. Alternatively, specify that contracts should use persistent storage for cross-frame guards.

---

### [M-11] Gas Isolation Between Frames Prevents Useful Patterns and Causes Systematic Over-Allocation

**Sources**: DOS-6, Design review

**Description**:
Line 443: "Unused gas from a frame is not available to subsequent frames." Each frame has a fixed `gas_limit`. If frame N uses less gas than allocated, that gas is refunded -- it can't flow to frame N+1.

This creates two problems:

1. **Block space waste**: A transaction can reserve 29M gas from a 30M block while using only 41k across its frames, monopolizing block space. There is no per-frame intrinsic cost to prevent this.

2. **UX friction**: The sender must pre-commit exact gas budgets per frame at signing time. For complex multi-frame transactions (deploy + verify + pay + user_op + post_op), the sender must predict gas usage for each frame independently. Over-allocation wastes gas (the user pays for the full allocation but only uses a portion). Under-allocation causes frame revert. In practice, wallets will over-allocate by 2-3x per frame, making frame transactions substantially more expensive than they need to be.

The alternative (shared gas budget) has DoS implications, but the rigid per-frame allocation creates significant practical friction that the EIP does not acknowledge.

**Recommendation**: Add per-frame intrinsic cost to prevent block stuffing. Consider a hybrid model where a global gas pool supplements per-frame minimums. Document the over-allocation UX tradeoff.

---

### [M-12] Atomic Batching Gas Accounting Is Unspecified for Reverted Frames

**Sources**: DOS-5, GEN-3, Design review

**Description**:
The atomic batch mechanism (lines 291-314) handles state revert but says nothing about gas accounting within a batch. If frame 0 (atomic) uses 50K gas and frame 1 (non-atomic, batch terminator) uses 30K gas, and frame 1 reverts:
- State from frames 0 and 1 is reverted via snapshot restore
- But is the gas from frame 0 refunded?

The spec says "gas refund is calculated as: `sum(frame.gas_limit) - total_gas_used`" but doesn't define whether `total_gas_used` includes gas from reverted atomic batch frames. Two valid interpretations:
- **Gas still "used"**: User pays for computation that was rolled back. Consistent but potentially unfair.
- **Gas refunded**: A malicious contract could waste gas in frame 0 (cheap to the user) then revert frame 1 to get it back -- but the node already did the work.

Additionally, whether `sender_approved`/`payer_approved` survive atomic batch state restore is implicit, not explicit. The spec says approval flags "cannot be reverted," but atomic batch says "restore the state to the snapshot." These could contradict depending on implementation.

Up to 999 frames in one atomic batch can write extensive storage then revert, forcing massive journal rollback with no per-frame intrinsic cost to compensate.

**Recommendation**: Explicitly state: (a) gas from reverted batch frames counts toward `total_gas_used` (not refunded), (b) `sender_approved`/`payer_approved` are transaction-level metadata outside the EVM state snapshot mechanism, (c) warm/cold journal behavior during rollback, (d) add per-frame intrinsic cost.

---

### [M-13] Atomic Batch Snapshot/Restore Interaction with Warm/Cold Journal Unspecified
**Source**: GEN-8, AA-16
Whether warm/cold access journal is part of atomic batch state snapshots is undefined. If journal IS restored, slots warmed in rolled-back frames become cold again. If NOT restored, journal state diverges from actual state. Affects gas costs for post-rollback frames.

### [M-14] Default Code SENDER Mode -- Underspecified RLP Decoding and Subcall Behavior
**Source**: GEN-9, PA-17, DOS-16, AA-4

No specified behavior for: malformed RLP, wrong tuple arity, empty call list, gas allocation per subcall, maximum subcall count. The default code loops over RLP-decoded `calls` with no size limit. A trailing revert rolls back all preceding calls' state changes, consuming gas with zero net effect. Different client implementations might handle edge cases differently, leading to consensus failures.

### [M-15] FRAMEDATACOPY Memory Expansion on VERIFY Mode -- Ambiguous Gas
**Source**: PA-07, GEN-15
FRAMEDATACOPY copies "no data" for VERIFY frames but may still charge memory expansion for large `length` parameters. A single `FRAMEDATACOPY(memOffset=0xFFFFFF, ...)` could trigger quadratic gas costs exceeding MAX_VERIFY_GAS.

### [M-16] Default Code Scope 0 Asymmetry Between Smart Contracts and EOAs
**Source**: GEN-5
Scope constraint 0 means "any scope allowed" for APPROVE but causes default code to revert. EOA users with `mode = 0x01` fail silently.

### [M-17] APPROVE Scope Bits as Confused Deputy -- Sender Constrains Paymaster
**Source**: AA-11
Mode bits 9-10 are set by the transaction creator, not the target contract. A sender can set mode bits that prevent a paymaster from calling its intended APPROVE scope, causing an exceptional halt. The contract's own logic is overridden by external inputs it doesn't control.

### [M-18] Non-Canonical Paymaster Sybil Amplification
**Source**: DOS-4
MAX_PENDING=1 is per-paymaster. Deploying 1,000 minimal paymaster contracts gives 1,000 slots. Mass invalidation by draining all simultaneously.

### [M-19] Revalidation Amplification on Block Reorganizations
**Source**: DOS-7
A canonical paymaster balance change triggers revalidation of all sponsored transactions -- potentially 500M gas equivalent per block.

### [M-20] EIP-7702 Delegation Interaction Is Ambiguous
**Source**: GEN-18
Whether a delegated EOA uses delegated code or default code in frame transactions is unspecified. A delegated account has a delegation designator as its code (not "no code"). If delegated code is used, PQ security goals are potentially undermined since the delegated code likely uses ECDSA.

### [M-21] Null Target + DEFAULT Mode Allows ENTRY_POINT to Call Sender with Arbitrary Data
**Source**: GEN-13
DEFAULT frame with null target calls `tx.sender` with `caller = ENTRY_POINT` and attacker-controlled calldata. Smart accounts should never trust ENTRY_POINT as an authorized caller for privileged operations.

### [M-22] Storage Read Restriction Invalidation Without Staking
**Source**: AA-9
Validation can read `tx.sender`'s storage, and the spec removes staking/reputation entirely (unlike ERC-7562). Smart accounts with configurable storage (session keys, spending limits) are vulnerable to invalidation by any storage write to a tracked slot.

### [M-23] SLOAD Restriction Ambiguity for DELEGATECALL vs CALL Chains
**Source**: DOS-8
"SLOAD only for tx.sender storage, including when reached transitively via CALL*" is ambiguous. A CALL to a helper contract where the helper does SLOAD reads the helper's storage, not tx.sender's. Misimplementation creates mass invalidation vectors.

### [M-24] Default Code RLP SENDER Mode Subcalls Lack Gas Allocation
**Source**: PA-17
The default code for SENDER mode iterates RLP-decoded subcalls with no specified gas allocation per subcall, no maximum subcall count, and no `call_value` bounds check. The first subcall can consume nearly all gas, causing subsequent subcalls to fail.

### [M-25] MAX_FRAMES (1,000) Enables Block Stuffing via Per-Frame Overhead
**Source**: DOS-9
1,000 frames with minimal gas each impose significant fixed overhead (context setup, journal entries, receipt construction) far exceeding what the gas payment covers. No per-frame intrinsic cost exists.

---

## Low Findings

### [L-1] No Signature Expiration/Deadline Mechanism (SIG-9)
No built-in `valid_until` field. Signed transactions remain valid indefinitely while the nonce is unconsumed. The banned opcodes list prevents validation code from checking TIMESTAMP or NUMBER, so even smart accounts cannot enforce time-based expiration in the validation prefix.

### [L-2] P256/ECDSA Address Derivation Collision -- No Domain Separator (SIG-10)
P256 uses the same address derivation as secp256k1: `keccak256(qx || qy)[12:]` with no curve-type domain separator. An adversary who breaks secp256k1 (e.g., quantum computer) could authenticate via `signature_type = 0x0`, defeating the PQ migration goal.

### [L-3] Signature Hash May Not Include Transaction Type Prefix (SIG-8)
The pseudocode shows `keccak(rlp(tx))` which appears to omit the `0x06` type prefix. Without it, a future transaction type with identical field layout would share the same sig_hash.

### [L-4] RLP Encoding Canonicality Not Explicitly Required (SIG-14)
`keccak(rlp(tx))` assumes canonical RLP encoding. Non-canonical encodings would produce different hashes. Consensus-splitting risk.

### [L-5] Default Code Scope 0x1 Creates Permanently Replayable Authorization (SIG-13)
A scope-0x1 signed transaction does not consume the nonce and remains valid indefinitely, replayable by ANY sponsor.

### [L-6] TXPARAM Param ID Gap 0x0A-0x0F Creates Implementation Risk (PA-04)
Param IDs jump from 0x09 to 0x10. Also, param 0x09 says "len(frames) (can be zero)" which contradicts `len(tx.frames) > 0`.

### [L-7] FRAMEDATALOAD Zero-Padding Masks Errors (PA-06)
Out-of-bounds reads return zeros (CALLDATALOAD semantics). Cannot distinguish "data is zero" from "read out of bounds" or "VERIFY mode."

### [L-8] FRAMEDATALOAD/FRAMEDATACOPY Stack Order Ambiguous (PA-14)
No explicit stack table provided (unlike APPROVE). Reversed stack order would silently read wrong frame/offset.

### [L-9] TXPARAM 0x13 Returns Masked Mode, Hiding Scope Flags (PA-10, GEN-10)
Only lower 8 bits returned. Security-sensitive decisions must also check params 0x16 and 0x17 separately.

### [L-10] Receipt payer Field Derivation Not Explicitly Defined (GEN-11)
Should be `frame.target` of the frame that called `APPROVE(0x2)` or `APPROVE(0x3)`, but this is left implicit.

### [L-11] TXPARAM Status (0x15) Harsh Exceptional Halt for Off-by-One Errors (GEN-12)
Accessing current/future frame status consumes all gas. Unusually harsh for a developer mistake.

### [L-12] No Constraint Preventing tx.sender == ENTRY_POINT (GEN-17)
If `tx.sender = 0xaa`, SENDER mode and DEFAULT mode frames both use `caller = ENTRY_POINT`. Contracts cannot distinguish.

### [L-13] Double-APPROVE "Revert" vs. Exceptional Halt Semantics Unspecified (GEN-19)
Whether double-approval causes REVERT (remaining gas returned) or exceptional halt (all gas consumed) is undefined.

### [L-14] APPROVE(0x2) Insufficient Balance Front-Running (DOS-10)
Front-running a sender's balance change can invalidate the entire transaction, not just the payment frame.

---

## Info Findings

### [I-1] Paymaster Nonce Never Incremented -- Must Self-Protect (GEN-20)
Only `state[tx.sender].nonce` is incremented. Paymaster contracts must implement their own replay/abuse prevention.

### [I-2] Non-Canonical Paymaster MAX_PENDING=1 Insufficient for Multi-Account Gas Accounts (AA-10, DOS-4)
With MAX_PENDING=1, a user with multiple accounts using the same gas EOA as paymaster can only have one sponsored transaction pending.

### [I-3] No ERC-1271 Equivalent for Frame Transaction Signature Verification (AA-15)
No equivalent of ERC-4337's `SIG_VALIDATION_FAILED` return value. Smart accounts must maintain separate code paths.

---

## Cross-Cutting Concerns

Several findings interact to create compound risks:

1. **Specification Consistency Crisis (C-1 + M-8 + M-14 + M-7)**: The undefined APPROVE(0x0), bit numbering inconsistencies, underspecified default code behavior, and ambiguous calldata cost definition collectively mean that no two independent implementations would produce identical results. This is a consensus-splitting risk of the highest order.

2. **Gas Accounting Attack Surface (H-2 + H-3 + H-7 + M-5 + M-12)**: Integer overflow in gas limits, underspecified fee collection, overflow in max cost calculation, ambiguous refund formula, and unspecified atomic batch gas accounting create a cluster of issues that could allow attackers to execute transactions at near-zero cost.

3. **Mempool DoS Amplification (H-1 + H-6 + H-11 + H-12 + M-18 + M-19)**: The 100K validation budget is simultaneously too large (DoS) and too small (PQ). Zero-cost validation grinding, explicit sender state reads, cross-node reservation inconsistency, paymaster Sybil amplification, and revalidation cascades combine to create significant node resource exhaustion vectors.

4. **Signature Malleability + VERIFY Elision (H-4 + M-2 + M-3)**: Both ECDSA and P256 signatures are malleable, and ALL VERIFY frame data is unsigned. This creates a broad surface for transaction manipulation by mempool observers -- signature flipping, sponsor data substitution, and paymaster parameter manipulation.

5. **Frame Isolation Model Is Inconsistent (M-10 + M-11 + M-13 + M-9)**: Transient storage is discarded but warm/cold journal is shared. Frame data is cross-readable but VERIFY data returns zeros. Atomic batch restores state but may not restore journals. The inconsistent isolation model creates security assumptions that don't hold and will surprise developers.

6. **Ambition vs. Constraint Gap (H-1 + H-6 + M-11)**: The EIP's stated goals (PQ migration, full AA) are undermined by its own constraints (100K gas cap excludes PQ schemes, single nonce prevents concurrency, rigid per-frame gas causes over-allocation). It currently delivers a more complex version of what EIP-7702 already provides, with the promise that future extensions will unlock the full vision.

---

## Recommendations Summary

### Must Fix Before Finalization
1. Resolve APPROVE scope numbering (C-1) -- the sponsored transaction flow is broken as written
2. Add explicit bit-width constraints on gas_limit, max_fee_per_gas (H-2, H-7)
3. Specify APPROVE gas collection semantics precisely (H-3)
4. Enforce low-s in default code for both ECDSA and P256 (H-4, M-3)
5. Add `ecrecover != address(0)` check (H-5)
6. Resolve APPROVE/ENTRY_POINT 0xaa collision (H-10)
7. Unify bit numbering convention (M-8)
8. Address MAX_VERIFY_GAS vs. PQ motivation gap (H-1) -- either raise cap or adjust motivation

### Should Fix
9. Address nonce increment timing (H-6) -- either move early or add rate-limiting
10. Define atomic batch snapshot scope and gas accounting explicitly (M-12, M-13)
11. Specify default code RLP error handling (M-14)
12. Restrict FRAMEDATALOAD cross-frame visibility for VERIFY frames (M-9)
13. Clarify EIP-7702 delegation interaction (M-20)
14. Add per-frame intrinsic cost to prevent block stuffing (M-11, M-25)
15. Add lightweight pre-validation for explicit sender field (H-11)
16. Define 2D nonce semantics now, not as future extension (H-6)

### Should Document
17. ORIGIN behavioral change with concrete exploit scenarios (M-1)
18. Transient storage reentrancy guard limitations (M-10)
19. VERIFY data malleability risks for paymaster/smart account implementers (M-2)
20. Deploy frame front-running risk (H-8)
21. Canonical paymaster trust model and centralization risk (H-9)
22. Gas isolation over-allocation UX tradeoff (M-11)
23. Explicit sender field state read implications (H-11)

---

## Methodology

Five parallel specialist agents audited the EIP against domain-specific checklists, supplemented by manual architectural design review:

| Agent | Domain | Raw Findings |
|-------|--------|-------------|
| 1 | Account Abstraction (ERC-4337) | 16 |
| 2 | Signatures | 14 |
| 3 | Denial of Service | 16 |
| 4 | General + Access Control | 20 |
| 5 | Precision/Math + Assembly | 17 |
| 6 | Architectural Design Review | 10 |
| **Total Raw** | | **93** |
| **Deduplicated** | | **55** |

Checklists sourced from: Dacian, beirao.xyz, Sigma Prime, RareSkills, Decurity, weird-erc20, Spearbit, Hacken, OpenZeppelin, Cyfrin, MixBytes, and the ERC-4337/ERC-7562 security literature.
