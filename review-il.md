# EIP-8141 "Frame Transaction" -- API Design Review

**Reviewer:** Immo Landwerth (style)
**Date:** 2026-04-12
**EIP Status:** Draft
**Category:** Standards Track / Core

---

## Preface

I am reviewing EIP-8141 as an API design surface. A new transaction type is, for all practical purposes, a public API that every Ethereum client, wallet, library, block explorer, indexer, and developer tool must implement and support *forever*. It cannot be versioned away. It cannot be deprecated without a hard fork. Every field name, every opcode semantic, every mode value becomes a permanent fixture of the protocol. I am reviewing it with that weight in mind.

---

## 1. What I Like

### The Frame Abstraction Itself Is Sound

The core idea -- decomposing a transaction into an ordered list of typed execution frames -- is a genuinely good abstraction. It cleanly separates *validation* from *execution* from *payment* without baking any particular flow into the protocol. This is the Ethereum equivalent of designing an interface rather than a concrete class. The protocol says "here are the building blocks and the rules for composing them," and leaves the policy to account code. That is exactly the right level of abstraction for a Layer 1 primitive.

### Signature Agility via `VERIFY` Frame Elision

Eliding `VERIFY` frame data from the signature hash (`compute_sig_hash`) is an elegant solution to a chicken-and-egg problem: the signature cannot be part of its own hash. Rather than introducing a special "signature" field that forces a particular layout, the spec lets the signature live in any frame marked `VERIFY` and strips it from the hash automatically. This is forward-compatible with signature aggregation and post-quantum schemes. It is the kind of design that avoids painting yourself into a corner.

### Explicit `sender` in the Transaction Envelope

Including `sender` directly in the RLP payload rather than deriving it from a signature is a necessary consequence of signature agility, and the authors have embraced it cleanly. This simplifies deserialization: you do not need to perform cryptographic recovery before you know who the transaction is from. For client implementations (Nethermind, Geth, etc.) this is a meaningful simplification of the hot path.

### The Receipt Contains `payer`

Good. The payer is not statically determinable and consumers need it. Putting it in the receipt is the right call rather than forcing every indexer to simulate execution.

### Default Code for EOAs

Providing "default code" behavior for accounts without code is the right backward-compatibility strategy. It means EOAs can use frame transactions immediately without deploying a contract first. The spec correctly specifies behavior rather than implementation, giving clients freedom in how they achieve it.

### Atomic Batching via a Flag

Using a single bit flag on consecutive `SENDER` frames to form atomic batches is pleasingly minimal. It avoids introducing a new frame mode or a nested structure. The "last frame in the batch is the one *without* the flag" convention is simple enough that implementers will get it right.

### Mempool Validation Prefix Concept

Defining "validation prefix" as the shortest prefix of frames that sets `payer_approved = true` and then limiting mempool enforcement to only that prefix is a clean separation of concerns. Everything after payment approval is the user's business. This is good API boundary design.

---

## 2. What I Do Not Like

### The Name `TXPARAM` Is Poor

`TXPARAM` is a grab-bag opcode that does completely different things depending on its first argument. It returns the nonce (a scalar), the sender (an address), the signature hash (a 32-byte digest), a frame's status (a boolean), and the currently executing frame index (a positional counter). These are semantically unrelated values accessed through a single dispatch table indexed by magic constants.

In .NET API design, we call this the "god method" anti-pattern. If you had a method `object GetParam(int paramId, int subId)` that returned an address, a gas price, or a boolean depending on magic integers, it would be rejected in the first five minutes of review.

The opcode values `0x00` through `0x09` are transaction-level parameters where `in2` must be zero. Values `0x10` through `0x17` are frame-level parameters where `in2` is a frame index. But the numbering jumps from `0x09` to `0x10` -- this is not sequential; it skips `0x0A` through `0x0F`. That gap suggests the authors *intended* a logical grouping but chose not to formalize it into separate opcodes.

**Recommendation:** Split into two opcodes: `TXINFO` for transaction-scoped fields (no frame index needed) and `FRAMEINFO` for frame-scoped fields (frame index required). This eliminates the unused `in2` parameter on half the calls, makes the intent self-documenting, and leaves room for future expansion in each category independently.

### The Name `in2` Is Unacceptable

The second stack operand for `TXPARAM` is called `in2`. This name carries zero semantic content. It is a frame index when `param >= 0x11`, and must be zero otherwise. The spec itself cannot decide what to call it -- the table header says `in2`, but the prose says "names a frame index."

**Recommendation:** Call it `frameIndex` in all documentation and spec text. For transaction-level params, specify that the opcode takes only one stack operand (which is another argument for splitting into two opcodes).

### `APPROVE` Overloads `scope` in a Confusing Way

`APPROVE` uses `scope` values `0x1`, `0x2`, and `0x3` where `0x3` means "both `0x1` and `0x2`." This is a bitmask disguised as an enumeration. The values happen to work as flags (`0x1 | 0x2 == 0x3`), but this is never stated explicitly. The spec lists them as discrete cases with subtly different behavior for `0x3` vs. applying `0x1` then `0x2` sequentially.

Then, separately, bits 9-10 of `frame.mode` also encode an "approval scope" that *constrains* which `scope` values `APPROVE` may use. The spec describes this with the formula `(frame.mode >> 8) & 3`, but the mode flag table says "bits 9-10." Bit 9 of a number shifted right by 8 is bit 1 of the result. This is confusing because the shift is by 8, not 9, and the reader must reconcile "bits 9-10" with `>> 8`. (It appears the spec means bits 8-9 counting from 0, or bits 9-10 counting from 1. This ambiguity should not exist in a protocol specification.)

**Recommendation:** Explicitly state that `scope` is a bitmask with `APPROVE_EXECUTION = 0x1` and `APPROVE_PAYMENT = 0x2`. Define named constants. Clarify the bit numbering with an explicit diagram or use consistent 0-indexed bit references throughout.

### `mode` Packs Too Much Into One Field

The `mode` field in each frame is simultaneously:
- An execution mode enum (bits 0-7, values 0-2 defined, 3-255 reserved)
- An approval scope (bits 8-9)
- An atomic batch flag (bit 10)

This is three logically distinct concerns packed into a single integer. The execution mode is the *identity* of the frame. The approval scope is a *constraint* on what the frame's code may do. The atomic batch flag is a *composition directive* for how frames relate to each other.

Packing these together means the "mode" of a frame is not really its mode -- it is its mode plus configuration flags. The spec even has to say things like "the lower 8 bits of `frame.mode`" and "bits 9/10 from `frame.mode`" throughout, which is a sign that one field is doing too much.

**Recommendation:** Separate `mode` into `mode` (uint8) and `flags` (uint16 or varint). This makes the frame tuple `[mode, flags, target, gas_limit, data]`. It is one more RLP element but dramatically improves clarity and leaves clean room for future flags without overloading the mode field further.

### `ENTRY_POINT` at `address(0xaa)` Is Never Explained

The constant `ENTRY_POINT = address(0xaa)` appears in the constants table and is used as the caller for `DEFAULT` and `VERIFY` mode frames. But the spec never explains *what* is at this address. Is it a contract? A precompile? An empty account? Is it the same `0xaa` as the `APPROVE` opcode value? If that collision is intentional, it should be stated. If accidental, it should be fixed.

The caller address for `DEFAULT`/`VERIFY` frames is `ENTRY_POINT`, which means any contract receiving a call can check `msg.sender == 0xaa` to know it was called from a frame transaction. This is a meaningful protocol detail that deserves more than a single row in a constants table.

### `TXPARAMLOAD` vs. `TXPARAM` Inconsistency

The spec defines the opcode as `TXPARAM` in the opcodes table and throughout the "New Opcodes" section. But the Rationale section refers to `TXPARAMLOAD` (line 664: "The canonical signature hash is provided in `TXPARAMLOAD`"), and the Default Code section also says "Retrieve the `mode` with `TXPARAMLOAD`" (line 324). The Python code comment references `TXPARAMLOAD(0x14, TXPARAMLOAD(0x10))` and `TXPARAMLOAD(0x08)`. This is a straight-up naming inconsistency in the spec. Pick one name.

### The `FRAMEDATALOAD` / `FRAMEDATACOPY` Naming Is Inconsistent with `TXPARAM`

If the transaction-level opcode is `TXPARAM`, why are the frame-data opcodes `FRAMEDATALOAD` and `FRAMEDATACOPY` rather than `FRAMEPARAM` or `TXDATALOAD`? The naming convention is inconsistent:

- `TXPARAM` -- transaction parameters, returns data on the stack
- `FRAMEDATALOAD` -- frame data, returns data on the stack
- `FRAMEDATACOPY` -- frame data, copies to memory

These three opcodes form a family, but their naming suggests otherwise. `CALLDATALOAD`/`CALLDATACOPY` are the EVM precedent and `FRAMEDATALOAD`/`FRAMEDATACOPY` follow that pattern well. But then `TXPARAM` should be `TXPARAMLOAD` (or better, the split I recommended above).

### `0x0` (Zero) Is Not a Valid `scope` for `APPROVE`, But It Is for Mode Bits

The mode bits 8-9 being `0` means "any scope can be used" in `APPROVE`. But `scope = 0` in the `APPROVE` opcode itself is invalid (only `0x1`, `0x2`, `0x3` are valid). So "any scope" really means "any of the three valid scopes." This asymmetry where zero means "unrestricted" in the mode bits but "invalid" as an actual scope value is a subtle trap.

### No `value` Field in Frames Is a Footgun

The Rationale says "No value in frame: It is not required because the account code can send value." This is technically true but creates a poor developer experience. Every other call-like construct in the EVM (`CALL`, `DELEGATECALL`, transaction types 0/1/2/3) has a native way to transfer value. Frame transactions are the only construct where sending ETH requires encoding the value transfer into calldata and having the account code execute it.

For the common case of "send 1 ETH to Bob," you must: create a VERIFY frame for signature verification, then create a SENDER frame whose data is an RLP-encoded call list containing the target, value, and empty data. The data efficiency table in the spec confirms this costs 134 bytes for the simplest possible transfer, versus ~110 bytes for an EIP-1559 transaction. The extra complexity is not just in bytes but in the cognitive overhead of understanding that ETH transfers are "just calls from account code."

### The Spec Says "Two Reasons" Then Lists Three

In the Rationale section on the canonical signature hash (around line 669), the text says "This is done for two reasons:" and then lists items 1, 2, and 3. Minor, but it erodes trust in the precision of the spec.

---

## 3. Issues

### Nonce Increment Timing Creates a Subtle Ordering Dependency

The nonce is incremented inside `APPROVE(0x2)` or `APPROVE(0x3)`. This means the nonce is not incremented until the payer approval frame executes. If validation fails before reaching the payer frame, the nonce is never incremented. This is probably intentional (invalid transactions should not consume nonces), but it means:

1. A malicious actor can spam the network with transactions that pass structural validation but fail during `VERIFY` frame execution, consuming node compute without ever incrementing the nonce.
2. The mempool rules attempt to address this, but the spec should explicitly state that nonce increment is deferred until `APPROVE` and discuss the implications.

### `compute_sig_hash` Mutates the Input

The Python pseudocode for `compute_sig_hash` modifies `tx.frames[i].data` in place:

```python
def compute_sig_hash(tx: FrameTx) -> Hash:
    for i, frame in enumerate(tx.frames):
        if (frame.mode & 0xFF) == VERIFY:
            tx.frames[i].data = Bytes()
    return keccak(rlp(tx))
```

This is a destructive operation. Any implementation that calls this function naively will lose the frame data. The spec should either note that this operates on a copy or rewrite it to be non-destructive.

### What Happens If No Frame Has Mode `VERIFY`?

The spec requires `payer_approved == true` after all frames execute, and `payer_approved` can only be set by `APPROVE(0x2)` or `APPROVE(0x3)`, which can only be called from `frame.target` in a frame where `ADDRESS == frame.target`. But nothing in the constraints section requires that at least one frame has mode `VERIFY`. The mempool rules require it for public propagation, but the *protocol-level* constraints do not.

Can a block builder include a frame transaction with no `VERIFY` frame? If some frame in `DEFAULT` mode manages to call `APPROVE`, would that work? The spec says `APPROVE` reverts if `ADDRESS != frame.target`, but it does not restrict `APPROVE` to `VERIFY` mode only. This ambiguity should be resolved.

### `ORIGIN` Redefinition Is More Breaking Than Acknowledged

The spec says `ORIGIN` returns the frame's `caller` throughout all call depths. This means `ORIGIN` returns `ENTRY_POINT` (address `0xaa`) during `DEFAULT` and `VERIFY` frames, and `tx.sender` during `SENDER` frames. The Backwards Compatibility section says this is "consistent with the precedent set by EIP-7702." But EIP-7702 is an opt-in delegation mechanism, while frame transactions change `ORIGIN` semantics for *any contract called within a frame*. A DeFi protocol that uses `tx.origin` (yes, it is discouraged, but it exists in production) will see `0xaa` as the origin during validation frames. This is not merely "may behave differently" -- it *will* break specific contracts.

### Atomic Batch Revert Semantics Are Ambiguous for Gas Accounting

When an atomic batch reverts, the spec says "restore the state to the snapshot taken before the batch." But what about gas? Is the gas consumed by reverted frames within the batch still charged? The spec says "unused gas from a frame is not available to subsequent frames" and the refund is calculated after all frames execute. But if frames 0-2 form an atomic batch and frame 2 reverts, are the gas costs of frames 0-2 all charged to the payer? Presumably yes, but this should be stated explicitly.

### Skipped Frames in Atomic Batches -- Gas Accounting

When a frame in an atomic batch reverts and subsequent frames are "skipped," do the skipped frames consume their `gas_limit`? Or is their gas returned to the pool? The spec does not say. This is a meaningful economic question: if I construct a batch with a large `gas_limit` on the last frame and an earlier frame reverts, do I still pay for the skipped frame's gas allocation?

### `APPROVE` Can Be Called from Nested Calls, But Should It?

The spec says `APPROVE` reverts if `ADDRESS != frame.target`. In a scenario where `frame.target` calls another contract via `DELEGATECALL`, the `ADDRESS` would still be `frame.target`. So a contract could `DELEGATECALL` to a library that executes `APPROVE`. Is this intentional? It means the "only `frame.target` can approve" invariant is weaker than it appears -- any contract reachable via `DELEGATECALL` chain from `frame.target` can approve.

### Default Code References `TXPARAMLOAD` Which Does Not Exist

As noted in section 2, the default code section references an opcode `TXPARAMLOAD` that is not defined in the opcodes table. The defined opcode is `TXPARAM`. This is not just a naming issue -- it could confuse implementers about whether there are one or two opcodes.

### `only_verify` Calls `APPROVE(0x0)` Which Is Invalid

In the Structural Rules (line 546), the spec says: "`only_verify` must call `APPROVE(0x0)`." But earlier (line 169), the spec says `APPROVE` scope must be `0x1`, `0x2`, or `0x3`, and "any other value results in an exceptional halt." So `APPROVE(0x0)` would cause an exceptional halt. This appears to be a bug in the mempool spec -- `only_verify` should call `APPROVE(0x1)` (approve execution only).

### No Mechanism to Query `blob_versioned_hashes[i]`

`TXPARAM(0x07, 0)` returns `len(blob_versioned_hashes)`, but there is no `TXPARAM` value to retrieve individual blob versioned hashes by index. If a smart account needs to verify or reason about specific blobs, it cannot access them. This seems like an oversight given that the spec explicitly supports blob transactions (`max_fee_per_blob_gas` and `blob_versioned_hashes` are in the envelope).

### `MAX_FRAMES = 10^3` Is Generous

One thousand frames per transaction is a lot. The practical examples show 2-5 frames. At 1000 frames, the gas accounting, atomic batch tracking, and frame interaction semantics become complex for client implementations. Was this limit chosen to "never be the bottleneck"? If so, that is a reasonable philosophy, but it should be stated.

---

## 4. Usage Scenarios I Find Compelling

### Post-Quantum Migration Path

The primary motivation is sound. ECDSA will eventually fall to quantum computers, and Ethereum needs a migration path that does not require every user to move their assets to new addresses. Frame transactions let existing accounts adopt new signature schemes by deploying account code that verifies the new scheme in a `VERIFY` frame. This is the most important use case and the spec handles it well.

### Gas Sponsorship Without Trusted Relayers

The canonical paymaster pattern (Examples 3 and 4) enables trustless gas sponsorship. A user can pay for gas in ERC-20 tokens without relying on a centralized relayer. The frame structure makes the payment flow explicit and auditable: verify sender, verify payer, transfer tokens, execute operation, post-operation cleanup. Each step is a separate frame with clear semantics.

### Atomic Approve-and-Swap

Example 2 (atomic approve + swap) solves a real and persistent UX problem. Today, users must submit two separate transactions (approve + swap), and if the swap fails, they are left with a dangling approval. Atomic batching eliminates this by reverting the approval if the swap fails. This alone would justify the atomic batching feature.

### First-Transaction Account Deployment

Example 1b shows deploying account code in the first frame and then using it in subsequent frames within the same transaction. This is a dramatically better onboarding experience than the current "deploy contract in one transaction, use it in another" flow. New users can receive funds at a counterfactual address and deploy + use their account in a single transaction.

### Separation of Validation and Execution for Wallet UX

The clear separation between `VERIFY` (static, no state changes) and `SENDER` (stateful execution) frames gives wallets a clean boundary for simulation. A wallet can simulate the `SENDER` frames to show users what will happen, confident that the `VERIFY` frames will not change state. This is a meaningful improvement for transaction preview UX.

---

## 5. Additional Possibilities Enabled

### Multi-Signature Transactions Without Wrapper Contracts

Frame transactions can natively represent multi-sig flows. Multiple `VERIFY` frames can verify different signers, and a smart account contract can require that N-of-M `VERIFY` frames succeed before calling `APPROVE`. This moves multi-sig logic from application-layer wrapper contracts into the transaction structure itself, reducing gas costs and improving auditability.

### Intent-Based Transaction Frameworks

The frame structure is a natural fit for "intent" architectures. A user can express an intent in a `VERIFY` frame (e.g., "I want to swap X for at least Y"), and a solver can fill in the `SENDER` frames with the actual execution path. The solver cannot forge the intent because it is covered by the signature hash (via `frame.target` of the `VERIFY` frame), but they have freedom in how they fulfill it.

### Session Keys and Delegated Execution

A smart account could implement session key logic where the `VERIFY` frame checks a session key signature and the `SENDER` frames are constrained by the session key's permissions. This enables "approve once, execute many" patterns without the security risks of blanket approvals.

### Cross-Frame Data Passing for Composable Protocols

`FRAMEDATALOAD` and `FRAMEDATACOPY` allow frames to read data from other frames (except `VERIFY` frames). Combined with `TXPARAM` for reading frame status, this enables a form of structured inter-frame communication. A post-operation frame can check whether execution frames succeeded and adjust behavior accordingly -- for example, a paymaster's post-op frame can check execution status and adjust refunds.

### Upgradeable Account Logic Without Address Changes

Because validation is defined by account code rather than by a fixed cryptographic scheme, users can upgrade their account logic (e.g., from ECDSA to a PQ scheme, or from a single key to a multi-sig) without changing their address. This is the "account abstraction" promise delivered at the protocol level rather than through ERC-4337's higher-level infrastructure.

### Programmable Nonce Schemes

The spec notes that `TXPARAM(0x01, ...)` has "a possible future extension to allow indices for multidimensional nonces." This hints at support for parallel transaction execution from a single account. Smart accounts could implement 2D nonces (channel + sequence) to allow independent transaction streams that do not block each other.

---

## 6. Changes to Increase Optionality

### Add a Version Field to the Frame Structure

The frame tuple is currently `[mode, target, gas_limit, data]`. Adding a leading `version` field (`[version, mode, target, gas_limit, data]`) would allow future hard forks to extend the frame structure without changing the transaction type. Version 0 would be the current spec. Future versions could add fields (e.g., `value`, `access_list`) without introducing a new transaction type or overloading existing fields. The cost is one byte per frame in the common case (version 0 encodes as a single RLP byte).

### Define an Explicit Frame Result Channel

Frames can currently check prior frames' `status` via `TXPARAM(0x15, frameIndex)`, but this is a single bit (success/failure). Consider adding a mechanism for frames to return structured data that subsequent frames can read -- for example, a return data area per frame accessible via a `FRAMERETURNDATA` opcode. This would enable richer inter-frame communication patterns like a `VERIFY` frame returning the verified signer address for consumption by a `SENDER` frame.

### Reserve `TXPARAM` Slots for Future EIP Fields

The `TXPARAM` table has a gap from `0x0A` to `0x0F`. Explicitly reserve these for future transaction-level fields (e.g., access lists if they return, multidimensional gas parameters, priority ordering hints). Documenting the reservation intent makes the numbering scheme intentional rather than accidental.

### Allow `APPROVE` to Return Data

`APPROVE` currently takes `offset` and `length` from the stack (like `RETURN`) but the spec does not describe what happens to that return data. If this data were made accessible to subsequent frames (e.g., via `TXPARAM` or a dedicated opcode), it could serve as a channel for the validation phase to communicate with the execution phase. For example, a `VERIFY` frame could return the validated signer identity for use by downstream frames.

### Consider a `CANCEL` or `REJECT` Opcode for Explicit Failure

Currently, a `VERIFY` frame that wants to reject a transaction simply reverts. But reverts are also used for "something went wrong unexpectedly." A dedicated `REJECT` opcode (analogous to `APPROVE` but for explicit rejection) would let smart accounts distinguish between "I examined this transaction and it is not authorized" vs. "I encountered an unexpected error during validation." This distinction matters for debugging, error reporting, and potentially for mempool penalty logic.

### Make Atomic Batch Boundaries First-Class

Currently, atomic batches are inferred from consecutive `SENDER` frames with bit 11 set. Consider making atomic batches explicit in the frame structure -- for example, a `batch_id` field where frames sharing the same `batch_id` form an atomic group. This would allow non-consecutive atomic batches and batches that span non-`SENDER` frames in the future. The current flag-based approach is simpler but less extensible.

### Add `FRAMEDATASIZE` as a Dedicated Opcode

Currently, getting a frame's data length requires `TXPARAM(0x14, frameIndex)`. But `FRAMEDATALOAD` and `FRAMEDATACOPY` are dedicated opcodes. The EVM has `CALLDATALOAD`, `CALLDATACOPY`, and `CALLDATASIZE` as a trio. For consistency, there should be a `FRAMEDATASIZE` opcode to complete the trio alongside `FRAMEDATALOAD` and `FRAMEDATACOPY`, rather than forcing developers to context-switch to `TXPARAM` for the size query.

### Define Behavior for `APPROVE` Return Data Explicitly

The `APPROVE` opcode takes `offset` and `length` parameters like `RETURN`, suggesting it produces return data. But the spec never says what happens to this data. Does the frame caller receive it? Is it discarded? Can subsequent frames access it? This needs to be specified. If the return data is intentionally discarded, say so. If it is accessible, define how. Leaving it unspecified will lead to divergent client implementations.

### Consider Making `VERIFY` Frames Explicitly Non-Reentrant

The spec says `VERIFY` frames behave like `STATICCALL` (no state modifications). But `STATICCALL` still allows calls to other contracts that can read state. If a `VERIFY` frame calls a contract that calls back into the verifying contract (reentrance via read-only calls), the behavior is defined but potentially confusing. Consider explicitly stating that `VERIFY` frames have the same reentrancy properties as `STATICCALL` to avoid implementer confusion.

### Explicitly Specify Gas Cost of `APPROVE`

The spec says `TXPARAM` costs 2 gas, `FRAMEDATALOAD` costs 3 gas, and `FRAMEDATACOPY` matches `CALLDATACOPY`. But I do not see an explicit gas cost for `APPROVE`. Since `APPROVE` performs significant work (nonce increment, balance check, balance deduction for scope `0x2`/`0x3`), its gas cost is non-trivial and should be specified. If it is intended to be the same as `RETURN`, state that explicitly.

---

## Summary Assessment

EIP-8141 is an ambitious and largely well-designed proposal. The frame abstraction is the right primitive for account abstraction at the protocol level. The separation of validation, execution, and payment into composable frames is sound. The mempool safety analysis is thorough.

The primary weaknesses are in naming consistency (`TXPARAM` vs. `TXPARAMLOAD`, `in2`, mode-flag overloading) and in under-specification of edge cases (gas accounting for atomic batch reverts, `APPROVE` return data, `APPROVE` gas cost, the `APPROVE(0x0)` bug in structural rules). These are fixable issues that do not undermine the core design.

The most impactful changes would be:
1. Fix the `APPROVE(0x0)` bug in the mempool structural rules
2. Split `mode` into `mode` + `flags`
3. Split `TXPARAM` into transaction-level and frame-level opcodes
4. Add a frame version field for future extensibility
5. Resolve the `TXPARAM` vs. `TXPARAMLOAD` naming inconsistency

This EIP deserves to move forward. The design is at the right level of abstraction, the use cases are compelling, and the problems are tractable.
