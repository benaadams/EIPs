# EIP-8141 "Frame Transaction" -- Comprehensive Combined Review

**Date:** 2026-04-12
**EIP Status:** Draft
**Authors:** Vitalik Buterin, lightclient, Felix Lange, Yoav Weiss, Alex Forshtat, Dror Tirosh, Shahaf Nacson, Derek Chiang

**Sources synthesized:** 17 independent reviews + 1 multi-domain security audit (55 deduplicated findings from 93 raw checklist items across 5 specialist agents)

---

## Reviewer Index

| Code | Reviewer / Perspective | File |
|------|----------------------|------|
| AA | Deep Protocol Review | review-aa.md |
| AF | Atomic Fusion Architect | review-af.md |
| BA | Ben Adams (.NET Performance) | review-ba.md |
| BK | Protocol Review (Systems-Level) | review-bk.md |
| CO | General Protocol Review | review-co.md |
| CX | Codex Adversarial Review | review-cx.md |
| DF | David Fowler (ASP.NET Core Architecture) | review-df.md |
| HK | Hideo Kojima (Thematic/Structural Analysis) | review-hk.md |
| HP | Hilmar Petursson (Sandbox MMO Philosophy) | review-hp.md |
| IL | Immo Landwerth (API Design) | review-il.md |
| KC | Kamil Chodola (QA / Edge Cases) | review-kc.md |
| PA | Protocol Architecture Review | review-pa.md |
| RS | Rory Sutherland (Behavioral Economics) | review-rs.md |
| ST | Stephen Toub (.NET Performance/Async) | review-st.md |
| TIM | Tim Seaward (Cryptography/Security) | review-tim.md |
| TS | Tomasz Stanczak (Client Implementation) | review-ts.md |
| VB | Vitalik Buterin (Mechanism Design) | review-vb.md |
| AUDIT | Multi-Domain Security Audit | audits/eip-8141/AUDIT-REPORT.md |

---

## Grade Summary

| Reviewer | Verdict | Key Sentiment |
|----------|---------|---------------|
| AA | **Ship with adjustments** | "The most architecturally sound AA proposal Ethereum has produced" |
| AF | **Ship with adjustments** | "Deserves to move forward. The right long-term answer to AA on Ethereum" |
| BA | **Ship, tighten constants** | "Architecture is sound. Tighten the constants and clarify the edge cases" |
| BK | **Ship, fix bugs first** | "Fix the bugs, clarify the ambiguities... The rest is solid" |
| CO | **Needs work** | "Not ready as-is. Has spec errors, under-specified edge cases, too much policy bundled" |
| CX | **Needs attention** | "Blocking contradictions in approval semantics; relies on undefined canonical paymaster" |
| DF | **Ship with changes** | "One of the most architecturally significant EIPs I have reviewed" |
| HK | **Ship, sharpen edges** | "Build it. But reduce MAX_FRAMES, add per-frame intrinsic cost, separate mempool policy" |
| HP | **Ship with migration path** | "Ambitious infrastructure for a persistent shared universe" |
| IL | **Ship with naming fixes** | "Largely well-designed. Primary weaknesses are naming consistency and edge cases" |
| KC | **Fix before advancing** | "Architecturally sound but has several inconsistencies causing implementation divergence" |
| PA | **Support moving to Review** | "The most architecturally significant transaction format change since EIP-1559" |
| RS | **Ship, lean into legibility** | "The engineering is sound. The real contribution is structural legibility" |
| ST | **Ship, fix bit numbering** | "Well-factored abstraction. Ship it, but fix the bit numbering first" |
| TIM | **Ship, tighten crypto** | "Deserves to move forward. Ethereum's PQ clock is ticking" |
| TS | **Ship, fix gaps** | "The most architecturally sound AA proposal. Fix APPROVE(0x0) bug, reduce MAX_FRAMES" |
| VB | **Proceed to broad review** | "Ready for serious implementation experimentation on devnets" |
| AUDIT | **55 findings (1C, 12H, 25M, 14L, 3I)** | "Significant gap between ambitions and constraints; blocking contradictions" |

**Overall consensus: Ship, but with mandatory specification fixes and parameter adjustments before advancing beyond Draft.**

---

## Emerging Consensus Points

These observations emerged independently across many or all reviewers, forming the strongest signal in this review.

### Universal Agreement (raised by all or nearly all 18 sources)

1. **The frame abstraction is the correct primitive.** Every reviewer independently concluded that decomposing transactions into typed, ordered frames is the right level of abstraction for account abstraction. This is the first AA proposal that treats the transaction as a *composition* rather than a *modification* of the existing model. [AA, AF, BA, BK, CO, CX, DF, HK, HP, IL, KC, PA, RS, ST, TIM, TS, VB, AUDIT]

2. **MAX_FRAMES = 1000 is too high.** Every reviewer flagged this. Suggested limits range from 16 [TS, DF] to 128 [HP, PA], with most clustering around 32-64 [AA, AF, BA, BK, HK, IL, KC, ST, VB]. No reviewer identified a use case requiring more than ~20 frames. [ALL]

3. **The PQ migration path is genuinely important.** The ability for individual accounts to migrate to post-quantum signature schemes without a protocol-level flag day is recognized as the strongest motivating use case. [ALL]

4. **Signature hash elision for VERIFY frames is elegant and forward-looking.** Simultaneously solves the recursive signing problem, enables future aggregation, and allows sponsor data to be appended after sender signing. [ALL]

5. **Default code for EOAs is the correct migration strategy.** Giving EOAs implicit frame transaction capabilities without requiring smart account deployment is essential for adoption. [ALL]

6. **Atomic batching solves a real problem cleanly.** The approve-then-swap pattern is the most common DeFi footgun, and the bit-flag design on consecutive SENDER frames is minimal and unambiguous. [ALL]

### Strong Agreement (raised by 10+ sources)

7. **Gas isolation between frames is too rigid.** Per-frame gas budgets that cannot share unused gas force systematic over-allocation, creating UX friction and capital inefficiency. Most reviewers suggest an optional gas-forwarding or gas-pool mechanism. [AA, AF, BA, BK, CO, DF, HK, HP, KC, PA, RS, ST, TIM, TS, VB, AUDIT]

8. **The ORIGIN semantic change is more breaking than acknowledged.** Changing ORIGIN to return the frame caller rather than the transaction origin -- and having it vary *between frames within the same transaction* -- will break deployed contracts that use `tx.origin`. The backwards compatibility section understates the impact. [AA, AF, BA, BK, CO, DF, HK, HP, KC, PA, RS, ST, TIM, TS, VB, AUDIT]

9. **The mode bit-packing is fragile and error-prone.** Encoding execution mode (bits 0-7), approval scope (bits 8-9), and atomic batch flag (bit 10/11) into a single integer creates implementation bugs. Most reviewers recommend separate fields: `mode`, `scope`/`flags`. [AA, BK, DF, HK, HP, IL, KC, PA, RS, ST, VB, AUDIT]

10. **Transient storage reset between frames is surprising and needs stronger justification.** EIP-1153 transient storage was designed as transaction-scoped. Resetting it between frames breaks reentrancy guard patterns and prevents useful cross-frame communication. The warm/cold journal being shared while transient storage is not creates an inconsistency. [AA, AF, BA, BK, DF, HK, HP, KC, PA, RS, ST, TIM, TS, VB, AUDIT]

11. **The canonical paymaster is underspecified.** The mempool security model depends on a contract implementation that is not included in the EIP. Nodes must identify canonical paymasters "by runtime code match" but the code to match against is undefined. [AF, CO, CX, DF, HK, HP, KC, PA, ST, TS, VB, AUDIT]

12. **A frame return data channel is needed.** The inability to pass structured data from VERIFY frames to execution frames is the most significant composability gap. Multiple reviewers independently proposed FRAMERETURNDATA opcodes or APPROVE return data semantics. [AA, AF, BA, DF, HK, IL, KC, PA, ST, TS, VB]

### Moderate Agreement (raised by 5-9 sources)

13. **No value field in frames forces ETH transfers through code.** The most common Ethereum operation (sending ETH) requires encoding the transfer in calldata and executing via account code, adding unnecessary bytes and complexity. [AA, AF, BA, BK, HK, HP, IL, KC]

14. **The nonce model is insufficiently specified.** Single-nonce sequential constraint limits throughput. The "possible future extension for multidimensional nonces" is a TODO, not a design. [AA, BA, BK, DF, HK, HP, KC, ST, AUDIT]

15. **The EIP bundles too much into one document.** Consensus transaction semantics, new opcodes, EOA default code, mempool policy, and canonical paymaster framework should be separated for independent evolution. [CO, BK, HK, HP, VB]

16. **The complexity budget is high.** Four new opcodes, a new transaction type, new execution semantics, extensive mempool rules, and default code for EOAs create an enormous implementation surface. [AF, CO, HK, KC, RS, VB]

17. **Non-canonical paymaster limit of 1 is too restrictive.** This effectively forces all serious paymasters to use the canonical implementation, creating centralization pressure and suppressing innovation. [AA, HP, KC, RS, VB]

---

## 1. What We Like (Deduplicated Strengths)

### 1.1 The Frame Abstraction Itself
*[AA, AF, BA, BK, CO, CX, DF, HK, HP, IL, KC, PA, RS, ST, TIM, TS, VB]*

The core insight -- decomposing a transaction into an ordered sequence of typed frames with distinct modes (VERIFY, SENDER, DEFAULT) -- is a genuinely good abstraction. It cleanly separates validation from execution from payment without baking any particular flow into the protocol. The three modes form a minimal but complete basis.

As DF notes, this is essentially a "transaction middleware pipeline" analogous to ASP.NET Core's middleware architecture. As RS observes, it transforms the transaction from a "black box with a confirm button" into "a sequence of legible, purposeful steps" -- a transformation in perceived control. As PA puts it, the frame model is "topologically superior to ERC-4337's UserOperation struct because it does not force a rigid pipeline."

The frame list as the unit of composition means you can express deploy-then-verify, verify-then-pay-then-execute, atomic multi-calls -- all within a single transaction type. No bundlers, no off-chain infrastructure, no mempool hacks. The transaction IS the bundle [BK].

### 1.2 VERIFY Mode as STATICCALL with Data Elision
*[AA, AF, BA, BK, CO, DF, HK, HP, IL, KC, PA, RS, ST, TIM, TS, VB]*

Two properties that solve two hard problems simultaneously:

1. **STATICCALL semantics** prevent validation-time side effects, eliminating mempool invalidation vectors that ERC-4337 always struggled with. Nodes can safely re-execute validation without worrying about state changes.

2. **Signature hash elision** means (a) the signature cannot cover itself, (b) future signature aggregation is not blocked since VERIFY data is opaque between frames, and (c) sponsor data can be appended after the sender signs.

The fact that `frame.target` for VERIFY frames IS covered means the sender explicitly commits to who can verify on their behalf -- a good balance of flexibility and security [AF]. This is the kind of design choice that "looks trivial but prevents an entire category of future protocol ossification" [PA].

### 1.3 The APPROVE Opcode Design
*[AA, AF, BK, HK, IL, KC, TIM]*

Making approval a new opcode rather than a return value convention avoids ERC-4337's ambiguity where existing contracts could not be retrofitted. Key strengths:
- Semantically unambiguous -- no way to accidentally APPROVE
- Only `frame.target` can call APPROVE, preventing approval delegation attacks
- The scope operand (0x1 execution, 0x2 payment, 0x3 both) cleanly separates authorization questions
- The constraint that sender must approve before payer prevents confused-deputy attacks [DF, BK, PA]

### 1.4 Sender/Payer Separation
*[AA, BK, CO, DF, HK, PA, ST, VB]*

The two-phase approval model (`sender_approved` then `payer_approved`) captures the essential trust relationship: "I authorize actions" is separable from "I pay for gas." This enables the full spectrum of gas sponsorship models without special-casing. The ordering constraint (sender before payer) is the "pit of success" pattern [DF]: you literally cannot build a transaction where someone pays for something the sender didn't approve.

### 1.5 EOA Default Code
*[AA, AF, BA, BK, CO, DF, HK, HP, IL, KC, PA, RS, ST, TIM, TS, VB]*

Every reviewer highlighted this as essential. Rather than forcing users to choose between EOAs and smart accounts, every existing EOA can use frame transactions immediately. Key aspects:
- No migration, no delegation, no deployment required for basic usage
- P256 support enables passkey/WebAuthn authentication from day one
- RLP-encoded multicall in SENDER mode gives EOAs native batching
- The migration path from EOA to smart account is gradual, not cliff-edged

As RS notes: "The user does not choose a new system. The new system simply becomes available around their existing account."

### 1.6 Atomic Batching via Flag
*[AA, AF, BA, BK, CO, DF, HK, HP, IL, KC, PA, RS, ST, TIM, TS, VB]*

The bit-flag design on consecutive SENDER frames is "worse is better" in the best sense [PA]: trivially parseable, requires no recursion, and the batch boundary semantics are unambiguous. The approve-then-swap pattern eliminates an entire class of dangling approval vulnerabilities. As RS observes, "atomic batching transforms a sequence of *hopes* into a single *commitment*."

### 1.7 Warm/Cold State Journal Sharing
*[AA, AF, BA, DF, HK, KC, PA, ST, TIM, TS]*

Sharing the warm/cold access journal across frames avoids double-charging for storage that multiple frames touch. For a typical smart account validation that reads 2-3 storage slots, this saves 4,200-6,300 gas [BA]. This is the kind of detail that reveals the authors have profiled real transaction patterns [AA].

### 1.8 Mempool Rules Are Thorough
*[AF, BA, BK, CO, KC, PA, TIM, TS, VB]*

The mempool section is unusually detailed. The four recognized validation prefixes, the canonical paymaster exception, the banned opcode list, and the revalidation rules form a coherent DoS-resistant policy. The decision to strip out staking and reputation systems entirely (unlike ERC-7562), replacing them with structural rules, is a simplification that makes the system more predictable [AF].

### 1.9 TXPARAM Opcode for Transaction Introspection
*[AF, BK, HP, IL]*

Giving contracts read access to the full transaction structure via TXPARAM is powerful. The signature hash (param 0x08) being directly accessible eliminates the need for expensive reconstruction. The ability to inspect other frames' metadata enables verification contracts to reason about the transaction structure without trust assumptions.

### 1.10 Receipt Structure Includes Payer
*[AF, HK, IL]*

Including `payer` in the receipt and per-frame `[status, gas_used, logs]` is the right level of observability. The payer is determined dynamically at execution time, so there is no other reliable way to surface this information.

### 1.11 Data Efficiency Is Competitive
*[AA, BA]*

At 134 bytes for a basic smart account transaction, frame transactions are within striking distance of legacy EIP-1559 transactions. The comparison with ERC-4337's ABI-encoded UserOperation (which wastes bytes on 32-byte field padding) is stark.

---

## 2. Concerns and Criticisms (Deduplicated)

> **Status as of PR #11521 (branch `frame-compression`):** Items below are annotated with their resolution status.

### 2.1 MAX_FRAMES = 1000 Is Excessively High -- **FIXED**
*[AA, AF, BA, BK, CO, DF, HK, HP, IL, KC, PA, RS, ST, TIM, TS, VB, AUDIT]*

*Reduced to 64. Per-frame intrinsic cost of 475 gas added.*

**Unanimously flagged by all reviewers.** No legitimate use case needs 1000 frames. The examples in the EIP use 2-5. Even the most complex sponsored-deployment-with-post-op transaction needs ~8-10 frames.

Concrete concerns:
- Receipt trie bloat from 1000 frame_receipts [BA]
- 500 independent atomic batch snapshots from alternating batch flags [AA, DF]
- Client implementation code paths only exercised at high frame counts will contain untested bugs [AA]
- No per-frame intrinsic cost means a 1000-frame transaction pays the same base cost as a 2-frame transaction [HK, AUDIT]
- Block stuffing via structural overhead far exceeding gas payment [AUDIT]

**Suggested limits:** 16 [TS, DF], 32 [AA, BK, BA, KC, ST], 64 [AF, HP, PA, VB], 128 [HP alternative]. Consensus clusters around **32-64**.

### 2.2 Gas Isolation Between Frames Is Too Rigid -- **NOT FIXED**
*[AA, AF, BA, BK, CO, DF, HK, HP, KC, PA, RS, ST, TIM, TS, VB, AUDIT]*

"Unused gas from a frame is not available to subsequent frames." This forces transaction constructors to pre-estimate gas for each frame independently. Consequences:
- Systematic over-provisioning wastes gas and increases the upfront cost payers must lock up
- Block gas capacity is reserved but unused, reducing effective throughput [TS]
- Simple operations like ETH transfers now require two frames (VERIFY + SENDER) each with their own gas allocation [AF]
- The refund mechanism mitigates cost but not the UX complexity of setting multiple independent "risk parameters" [RS]

**Suggested solutions:**
- Optional "gas pool" mode via a flag bit [DF, HP, PA, VB]
- Gas cascading/forwarding from one frame to the next, opt-in [BA, ST, HK]
- `gas_limit = 0` sentinel meaning "use remaining gas from shared pool" [DF]

### 2.3 ORIGIN Semantic Change Is More Breaking Than Acknowledged -- **NOT FIXED**
*[AA, AF, BA, BK, CO, DF, HK, HP, KC, PA, RS, ST, TIM, TS, VB, AUDIT]*

The EIP changes ORIGIN to return the frame caller rather than the transaction origin. This creates a **per-frame varying ORIGIN** within a single transaction -- unprecedented behavior:
- DEFAULT/VERIFY frames: ORIGIN = ENTRY_POINT (0xaa)
- SENDER frames: ORIGIN = tx.sender

Contracts that use `tx.origin == msg.sender` as an EOA gate (a common if discouraged pattern) will behave differently under frame transactions. The failure is silent -- ORIGIN returns a valid address, just not the one expected [AA]. The backwards compatibility section treats this as a single paragraph; it deserves a dedicated security analysis with concrete examples of impacted contracts [AUDIT, HK, HP].

### 2.4 Mode Bit-Packing Is Fragile -- **FIXED**
*[AA, BK, DF, HK, HP, IL, KC, PA, RS, ST, VB, AUDIT]*

*Separated into `mode` (uint8) + `flags` (uint8) in the frame tuple. Named constants defined. Explicit 0-indexed bit layout.*

The `mode` field encodes three independent concerns in a single integer:
- Bits 0-7: Execution mode (DEFAULT, VERIFY, SENDER)
- Bits 8-9: Approval scope constraint
- Bit 10/11: Atomic batch flag

This is a "flags enum" anti-pattern [DF]. The spec already has bit-numbering inconsistencies between prose and pseudocode (see Issues section). The TXPARAM opcode then re-exposes these as separate fields (0x13 for mode, 0x16 for scope, 0x17 for atomic_batch), acknowledging the packed representation is not how consumers want the data [ST].

**Recommendation from most reviewers:** Separate into `mode` (uint8) + `flags` (uint16). Cost is 1-2 extra RLP bytes per frame; clarity gain is enormous.

### 2.5 Transient Storage Reset Between Frames -- **NOT FIXED**
*[AA, AF, BA, BK, DF, HK, HP, KC, PA, RS, ST, TIM, TS, VB, AUDIT]*

EIP-1153 defined transient storage as transaction-scoped. Resetting it between frames:
- Breaks reentrancy guards that use TSTORE [AA, KC, AUDIT]
- Prevents VERIFY frames from passing authorization metadata to SENDER frames [AF, PA]
- Creates an inconsistency: warm/cold journal is shared, but transient storage is not [BK, TS, VB]

The rationale for this choice is never stated in the EIP [BK, TS].

**Suggested compromises:**
- Preserve transient storage within atomic batches [BA, BK, HP]
- Allow opt-in sharing between SENDER frames from the same account [DF, VB]
- Add a "validation transient storage" that persists to the immediately following frame only [AF]

### 2.6 Complexity Budget Is Substantial -- **NOT FIXED (complexity increased slightly)**
*[AF, CO, HK, KC, RS, VB]*

*FRAMEPARAM opcode added (now 5 new opcodes instead of 4). Flags field added. However, the separation of mode/flags arguably reduces cognitive complexity even if surface area grows.*

This EIP introduces: a new transaction type, 4 new opcodes, 3 execution modes with bit-encoded flags, a new receipt format, default code semantics, atomic batching logic, and an extensive mempool policy. Each piece may be justified individually, but the aggregate implementation burden on client teams is enormous.

VB pushes on whether FRAMEDATALOAD and FRAMEDATACOPY can be eliminated or deferred. CO recommends splitting into: (a) core consensus/execution EIP, (b) public mempool profile, (c) canonical paymaster EIP. HK recommends separating the mempool policy into a companion document.

RS notes the risk is not that implementers will get it wrong, but that the *perceived complexity* will deter adoption by wallet developers and the tooling ecosystem. The EIP does "very little work to make itself *feel* simple" -- examples are at the bottom after 800 lines of specification.

### 2.7 No Value Field in Frames -- **NOT FIXED (by design)**
*[AA, AF, BA, BK, HK, HP, IL, KC]*

The rationale says "the account code can send value." While true, every ETH transfer now requires encoding the transfer in calldata and executing via SENDER mode. For the most common operation on Ethereum (~40% of transactions [BK]), this adds 22+ bytes of overhead and cognitive complexity. A `value` field defaulting to 0 would close the efficiency gap.

### 2.8 Canonical Paymaster Is Underspecified -- **NOT FIXED**
*[AF, CO, CX, DF, HK, HP, KC, PA, ST, TS, VB, AUDIT]*

*New text acknowledges secp256k1-only and may change, but the paymaster is still not normatively specified. No code hash, no withdrawal state machine, no `pending_withdrawal_amount` computation.*

The EIP references a "canonical paymaster implementation" repeatedly and builds critical mempool rules around it, but the actual code/specification is not included. As VB notes: "If it is not yet specified, this EIP is incomplete." As CX observes: "Public mempool admission is delegated to an implementation the draft never actually specifies."

Specific gaps:
- No defined canonical paymaster bytecode or code hash
- No withdrawal state machine specification
- No `pending_withdrawal_amount` computation definition
- Code-match identification is operationally fragile [AF] -- any bug requires deploying a new instance and updating every node's matching logic
- Anyone can deploy identical code and drain it [AUDIT]

### 2.9 Non-Canonical Paymaster Limit = 1 Is Too Restrictive -- **NOT FIXED**
*[AA, HP, KC, RS, VB]*

This effectively makes non-canonical paymasters useless for any paymaster serving more than one user simultaneously. It forces all serious paymaster operators to use the canonical implementation, creating protocol-level lock-in [VB] and centralization pressure [AA].

**Suggested alternatives:** Staking-based scaling (stake N ETH, get M pending tx slots) [AA], raise to 4-8 with balance reservation [VB].

### 2.10 Explicit Sender Field Is Unusual and Creates Attack Surface -- **PARTIALLY FIXED (documented)**
*[AF, BA, BK, HK, IL, AUDIT]*

*New security section documents state-read amplification and recommends mitigations. `tx.sender != address(0)` added as static constraint. But no structural pre-validation or rate-limiting mechanism specified.*

Including `sender` as an explicit field (rather than deriving it from a signature) means:
- The sender is self-declared, only validated by VERIFY frame code
- If VERIFY validation has a bug, transactions can impersonate arbitrary senders [AF]
- Nonce oracle attacks: attacker learns any account's nonce by submitting transactions with that sender [AUDIT]
- State read amplification: rapid submission of transactions claiming different senders forces I/O [AUDIT]

### 2.11 The Mempool Section Mixes Consensus and Policy -- **NOT FIXED**
*[BK, CO, HK, HP, RS, VB]*

The EIP oscillates between protocol-level behavior and mempool policy. These are different stability levels -- protocol rules are permanent; mempool policy evolves. The language often shifts between "transaction is invalid" and "must not be propagated" without crisp distinction [BK, CO]. The banned opcode list during validation (21 opcodes) is a substantial restriction that will surprise developers [HP].

### 2.12 Nonce Increment Timing Is Fragile -- **NOT FIXED**
*[AA, AF, BA, BK, DF, HK, HP, IL, KC, ST, TS, VB, AUDIT]*

The nonce is checked at transaction start but only incremented inside APPROVE. If validation fails before APPROVE, the nonce is not consumed. This creates:
- Zero-cost validation grinding for attackers [AUDIT]
- A single-pending-transaction-per-account bottleneck [AA]
- No pipeline for concurrent frame transactions from the same account [AA]
- Ambiguity about what happens if two same-nonce transactions hit the same block [ST]

The "possible future extension for multidimensional nonces" is a TODO that does "enormous load-bearing work for future scalability" but is completely unspecified [AA].

### 2.13 Intrinsic Cost of 15,000 Gas Is Questionable -- **PARTIALLY FIXED**
*[BA, HK, AUDIT]*

*Per-frame cost of 475 gas added (`FRAME_TX_PER_FRAME_COST`). But the base 15,000 is unchanged and not justified with concrete measurements.*

The FRAME_TX_INTRINSIC_COST of 15,000 gas does not scale with frame count. A 1000-frame transaction pays the same intrinsic cost as a 2-frame transaction. BA notes the cost should be justified with concrete measurements, and questions whether it double-charges for work already covered by per-frame gas limits.

**Suggestion:** Add per-frame intrinsic cost (e.g., 500-1000 gas per frame beyond the first two) [HK, TS, AUDIT].

### 2.14 EIP-4337 Migration Path Is Missing -- **NOT FIXED**
*[HP]*

EIP-4337 has been live since 2023 with a substantial ecosystem of bundlers, paymasters, smart accounts, and SDKs. The EIP says nothing about migration. What happens to existing 4337 smart accounts? Can they use frame transactions directly? This is strategically important for adoption.

---

## 3. Specification Issues and Bugs

> **Status as of PR #11521 (branch `frame-compression`):** Items below are annotated with their resolution status.

### Critical

#### 3.1 APPROVE(0x0) in Mempool Structural Rules Is Invalid -- **FIXED**
*[BK, CO, CX, IL, KC, TS, AUDIT]*
**Repeated by 7+ sources -- the most consistently flagged bug.**

The mempool structural rules state:
- `self_verify` must call `APPROVE(0x2)`
- `only_verify` must call `APPROVE(0x0)`
- `pay` must call `APPROVE(0x1)`

But the APPROVE spec defines only scopes 0x1, 0x2, 0x3. Scope 0x0 causes an exceptional halt. And `pay` calling `APPROVE(0x1)` means "approval of execution" which requires `frame.target == tx.sender` -- but the paymaster is NOT the sender.

Comparing with the EIP's own examples: Example 1 uses `APPROVE(0x3)` for self-relay; Example 3 uses `APPROVE(0x1)` then `APPROVE(0x2)`. The structural rules are simply wrong.

**Correct values should be:**
- `self_verify`: `APPROVE(0x3)` (both execution and payment)
- `only_verify`: `APPROVE(0x1)` (execution only)
- `pay`: `APPROVE(0x2)` (payment only)

#### 3.2 Bit Numbering Inconsistency -- Consensus-Splitting Risk -- **FIXED**
*[BA, BK, KC, PA, ST, VB, AUDIT]*
**Repeated by 7+ sources.**

*Mode and flags separated into distinct fields. Named constants and explicit 0-indexed bit positions.*

The mode flags table says "bits 9-10" for approval scope and "bit 11" for atomic batch. But the code uses `(frame.mode >> 8) & 3` to extract scope (which gives bits 8-9 in 0-indexed) and `(frame.mode >> 10) & 1` for atomic batch (which gives bit 10 in 0-indexed). Three conflicting descriptions of the bit layout exist in the spec. Implementations following different sections will compute different values.

**Fix:** Include a definitive bit layout diagram with consistent 0-indexed numbering, and align all pseudocode.

#### 3.3 compute_sig_hash Mutates the Transaction Object -- **FIXED**
*[BK, DF, IL, KC, ST, TS, VB]*
**Repeated by 7 sources.**

*Now explicitly operates on `deep_copy(tx)` with comment: "the original transaction object is not modified."*

The Python pseudocode modifies `tx.frames[i].data` in place. Any implementation that calls this function naively will destroy the VERIFY frame data. The spec must either explicitly state it operates on a copy or rewrite it as a pure function.

### High

#### 3.4 TXPARAM/TXPARAMLOAD Naming Inconsistency -- **FIXED**
*[BK, CO, IL, KC, TS, VB]*

*Unified on `TXPARAM`. Frame-level params moved to new `FRAMEPARAM` opcode (0xb3). All `TXPARAMLOAD` references removed.*

The opcode is defined as `TXPARAM` in the opcodes table but the Rationale section and Default Code section repeatedly reference `TXPARAMLOAD`. The Python code comments use `TXPARAMLOAD`. Implementers will be confused about whether there are one or two opcodes.

#### 3.5 VERIFY Mode Is STATICCALL-Like but APPROVE Modifies State -- **FIXED**
*[AF, BA, BK, DF, HK, KC, RS, ST]*

*Now explicitly states: "The execution behaves the same as STATICCALL for user code: state cannot otherwise be modified. The APPROVE opcode is the only exception and applies its protocol-defined effects, including approval updates and, for payment scopes, nonce increment and gas-charge collection."*

VERIFY mode "behaves the same as STATICCALL -- state cannot be modified." But APPROVE increments nonces and deducts balances. This exception to the static-call rule is never stated. Client implementations enforcing STATICCALL at the EVM level will need a special carve-out for APPROVE, which is a bug-prone pattern [AF].

**Fix:** Explicitly state: "VERIFY mode frames execute as STATICCALL, except that the APPROVE opcode may modify transaction-scoped state (nonce increment and balance transfer for payment)."

#### 3.6 APPROVE Return Data Is Unspecified -- **NOT FIXED**
*[BA, DF, IL, KC, ST, VB]*

APPROVE takes offset and length from the stack (like RETURN) but the spec never describes what happens with this return data. Is it available to subsequent frames? Is it discarded? Can VERIFY frames communicate information to later frames via APPROVE's return data? If discarded, say so. If accessible, define how.

#### 3.7 EIP-7702 Delegation Interaction Is Unspecified -- **FIXED**
*[BK, DF, HP, KC, ST, TS, VB, AUDIT]*

*Frame txs do not include 7702 authorization lists. Default code only for accounts with neither code nor 7702 delegation indicator. 7702 delegated accounts use delegated-code semantics. Added `requires: 7997`.*

What happens when `tx.sender` has an EIP-7702 delegation? The default code applies to "accounts with no code," but a delegated account has a delegation designator. Two plausible answers:
1. Delegated code runs (consistent with 7702) -- but may not understand APPROVE
2. Default code runs regardless (special case for frame transactions)

BK recommends: "Either answer is defensible. Not specifying the answer is not."

#### 3.8 Skipped Frame Receipt Semantics Are Undefined -- **NOT FIXED**
*[BK, CO, DF, IL, KC, ST]*

When frames are skipped due to atomic batch revert, the spec never defines:
- Does skipped mean `status = 0`? Or a third status?
- Is `gas_used` zero for skipped frames?
- Are skipped frames visible to `TXPARAM(0x15)` as failure or something else?
- Do skipped frames consume their gas allocation?

#### 3.9 Atomic Batch Gas Accounting Is Ambiguous -- **NOT FIXED**
*[AF, BA, BK, DF, HK, IL, KC, ST, AUDIT]*

When an atomic batch reverts, state is restored. But gas is not state. The spec does not clarify:
- Is gas consumed by reverted frames charged or refunded?
- Do `sender_approved`/`payer_approved` survive batch state restore?
- Is the warm/cold journal restored or preserved?

**Consensus recommendation:** Gas from reverted batch frames counts as consumed (not refunded), consistent with how the EVM handles reverted internal calls. But this must be stated explicitly.

#### 3.10 TXPARAM Param Numbering Gap (0x09 to 0x10) -- **FIXED**
*[BA, BK, HK, HP, IL, KC, PA, ST, TS, VB]*

*Numbering gaps eliminated. TX params contiguous 0x00-0x0A. Frame params moved to FRAMEPARAM 0x00-0x07. "Can be zero" contradiction for `len(frames)` removed.*

Parameters jump from 0x09 to 0x10, skipping 0x0A through 0x0F. This appears intentional (grouping TX-level params in 0x00-0x0F and frame-level params in 0x10-0x1F) but is not documented. Invalid values in the gap cause exceptional halts, making the reserved space unusable without a hard fork.

Additionally, TXPARAM(0x09) says `len(frames)` "can be zero" but the constraints require `len(tx.frames) > 0` -- a direct contradiction [BA, BK, KC, ST].

#### 3.11 Default Code SENDER Mode All-or-Nothing Is Too Strict -- **NOT FIXED**
*[KC, ST]*

The default code for SENDER mode reverts the entire frame if any single sub-call reverts. An EOA wanting partial-success batching must use separate SENDER frames, each with its own gas allocation. This defeats the purpose of multicall within a single frame.

### Medium

#### 3.12 TXPARAM(0x08) Gas Cost Is Mispriced -- **FIXED**
*[BA, BK, ST, TS]*

*Spec now mandates: "This value MUST be computed at most once per transaction and cached." Gas cost of 2 is appropriate for a cached lookup.*

TXPARAM with param 0x08 returns `compute_sig_hash(tx)`, requiring hashing the entire transaction. The gas cost is 2, which is wildly insufficient. At 6 gas per word for keccak256, a 128KB transaction would cost ~24,000 gas just for the hash. Either the signature hash must be cached (and the spec must mandate this) or the gas cost must be variable.

#### 3.13 Null Target Semantics Are Inconsistent -- **FIXED**
*[BK, CO]*

*Explicit "resolved target" concept introduced (`resolved_target = frame.target if not None else tx.sender`). Used consistently throughout all checks.*

The execution rules say null target resolves to `tx.sender`, but later checks still compare against `frame.target` directly (APPROVE requires `ADDRESS == frame.target`; default code checks `frame.target != tx.sender`). The spec needs an explicit "resolved target" concept used consistently. As written, many example flows appear to revert [CO].

#### 3.14 Default Code P256 Address Derivation Issues -- **PARTIALLY FIXED**
*[AF, BA, TIM, TS, AUDIT]*

- **No point validation specified.** The EIP does not mandate that `(qx, qy)` is a valid P256 curve point before use [TIM] -- **PARTIALLY FIXED** *Notes now state "P256VERIFY must reject invalid public keys, including points that are not on the P256 curve." But this is in notes, not in the normative pseudocode flow.*
- **No point compression.** Requiring both qx and qy adds 32 bytes per P256 transaction [TIM] -- **NOT FIXED**
- **No domain separation** from secp256k1 derivation scheme -- both use `keccak256(qx || qy)[12:]` [TIM, AUDIT] -- **FIXED** *`P256_ADDRESS_DOMAIN = b"\x01"` added. P256 address is now `keccak(P256_ADDRESS_DOMAIN|qx|qy)[12:]`.*
- **Key rotation impossible** for P256 EOAs without migrating to a new address [BA] -- **NOT FIXED**
- **160-bit collision resistance** reduced to 80 bits by Grover's algorithm in the PQ threat model [AF] -- **NOT FIXED**

#### 3.15 ENTRY_POINT Address 0xaa Collides with APPROVE Opcode 0xaa -- **PARTIALLY FIXED (documented only)**
*[HK, IL, KC, TIM, AUDIT]*

*ENTRY_POINT behavior now explicitly documented with a full paragraph. States "its numeric equality with the APPROVE opcode value has no semantic significance." But the collision itself remains -- values not changed.*

Both the ENTRY_POINT address and the APPROVE opcode share value `0xaa`. While in different namespaces, this creates debugging confusion and potential issues if code exists at the ENTRY_POINT address. The spec should use different values.

#### 3.16 Explicit APPROVE Gas Cost Not Specified -- **NOT FIXED**
*[IL, ST]*

APPROVE performs significant work (nonce increment, balance check, balance deduction) but has no explicit gas cost. If intended to match RETURN, state that explicitly.

#### 3.17 FRAMEDATACOPY Gas Cost Is Underspecified -- **PARTIALLY FIXED**
*[KC, ST]*

*Text improved: "calculated exactly as for CALLDATACOPY, including the fixed cost of 3, the per-word copy cost, and the standard EVM memory expansion cost." But the exact formula is still not spelled out.*

"The gas cost matches CALLDATACOPY" is insufficient. CALLDATACOPY has a complex gas formula (3 base + 3 * ceil(length/32) + memory expansion). The exact formula must be specified.

#### 3.18 "Known Deterministic Deployer" Is Not Defined -- **FIXED**
*[DF, HK]*

*All references now point to the specific "EIP-7997 deterministic factory predeploy." Added `requires: 7997`.*

The mempool rules reference "a known deterministic deployer" but don't specify which deployers are known. This leads to divergent mempool behavior across client implementations.

#### 3.19 Rationale Lists Three Reasons Under "Two Reasons" -- **FIXED**
*[IL, KC]*

*Now says "three reasons."*

Minor editorial error in the canonical signature hash rationale section.

#### 3.20 Behavior Section Numbering Starts at 2 -- **FIXED**
*[KC]*

*Steps now start at "1."*

The numbered steps start at "2. Execute a call..." with no step 1. May indicate missing content.

#### 3.21 Receipt Format Missing Bloom Filter -- **NOT FIXED**
*[VB]*

The receipt format does not include the logs bloom filter that existing receipt types include. This may break tooling.

#### 3.22 No Maximum Size for frame.data -- **NOT FIXED**
*[BK, KC, ST]*

*MAX_FRAMES reduced to 64 which limits total frame count, but no per-frame data size limit added.*

No explicit limit on individual frame data size. A transaction with 1000 frames, each with 128KB of data, would be ~128MB.

---

## 4. Security Findings

> **Status as of PR #11521 (branch `frame-compression`):** Items below are annotated with their resolution status.

### Critical

#### 4.1 [AUDIT C-1] APPROVE Scope Self-Contradiction Breaks Sponsored Transaction Flow -- **FIXED**
The mempool structural rules require `APPROVE(0x0)` which is invalid and `APPROVE(0x1)` for paymaster which requires `frame.target == tx.sender`. The entire sponsored transaction flow is broken as specified. (See 3.1 above)

*Scopes remapped to named constants. All rules, examples, and flows regenerated.*

### High

#### 4.2 [AUDIT H-4, TIM] ECDSA Signature Malleability -- No Low-s Enforcement -- **FIXED**
*[BA (notes CanonicalPaymaster gets it right), TIM, AUDIT]*

*Default code now checks `s > secp256k1n / 2` and reverts. Constant `SECP256K1N_DIV_2` added.*

The default code calls `ecrecover(sig_hash, v, r, s)` without enforcing `s <= secp256k1.n / 2`. For any valid `(v, r, s)`, a second valid signature exists. Since VERIFY frame data is elided from sig_hash, mempool observers can flip the signature, producing a distinct but equally valid transaction. This breaks transaction hash uniqueness. The CanonicalPaymaster correctly enforces this; the default code should match.

#### 4.3 [AUDIT H-5, TIM] ecrecover Returns address(0) for Invalid Signatures -- **FIXED**
The default code checks `frame.target != ecrecover(...)`. If ecrecover fails (returns address(0)) and `frame.target` is also address(0), the check passes. The zero address holds substantial burned ETH on mainnet.

**Fix:** Add `if recovered == address(0): revert()`. Reject `tx.sender == address(0)` as a static constraint.

*Both checks added: `recovered == bytes(20)` revert and `tx.sender != bytes(20)` static constraint.*

#### 4.4 [AUDIT H-2] Integer Overflow in sum(frame.gas_limit) -- **FIXED**
With MAX_FRAMES=1000 and no explicit upper bound on `frame.gas_limit`, two frames with `gas_limit = 2^255` each cause the sum to overflow to 0 in 256-bit arithmetic. The attacker pays near-zero fees.

**Fix:** Constrain `frame.gas_limit` to uint64. Add: `assert sum(frame.gas_limit) <= 2^63 - 1`.

*`gas_limit <= 2^63-1` per frame. Running `total_frame_gas` sum checked against `<= 2^63-1`. MAX_FRAMES reduced to 64.*

#### 4.5 [AUDIT H-1] MAX_VERIFY_GAS (100K) vs. PQ Ambitions -- **NOT FIXED**
The EIP's primary motivation is PQ migration, but 100K gas is insufficient for most PQ schemes:
- ML-DSA verification: ~200-500K gas
- SPHINCS+: ~500K+ gas
- ZK-based auth: ~170-220K gas

The only signatures fitting 100K are ECDSA and P256 -- the same curves we already have. The gas cap simultaneously creates DoS risk (100K free computation per sender) and blocks the EIP's stated purpose.

#### 4.6 [AUDIT H-3] APPROVE "Collect Total Gas Cost" Is Underspecified -- **PARTIALLY FIXED**
Scope 0x2 says "collect the total gas cost from the account" without specifying which account. It's also unspecified whether the collected amount is `tx_gas_limit * max_fee_per_gas` (upfront max) or `tx_gas_limit * effective_gas_price`.

*Now specifies: "collect the transaction's maximum cost (TXPARAM(0x06)) from resolved_target." Which account and amount are now clear. Skipped/reverted batch charging still unspecified.*

#### 4.7 [AUDIT H-7] TXPARAM(0x06) Max Cost Overflow -- **PARTIALLY FIXED**
`max_fee_per_gas` and `max_fee_per_blob_gas` have no bit-width constraints. `2^128 * 2^128 = 0 mod 2^256`. A paymaster sees `max_cost = 0` and approves a transaction it cannot cover.

*`gas_limit` bounded to uint63, which limits one factor of the product. But `max_fee_per_gas` and `max_fee_per_blob_gas` still have no explicit bit-width constraints.*

#### 4.8 [AUDIT H-8] Deploy Frame Front-Running -- **FIXED (documented)**
The deploy frame executes BEFORE authentication. An attacker observing a pending frame transaction can front-run by deploying code at the sender's address first.

*New "Deploy Frame Front-Running" security section documents the risk and mitigations. Wallets advised to expect resubmission without the deploy frame.*

#### 4.9 [AUDIT H-9] Canonical Paymaster Single Point of Failure -- **NOT FIXED**
All canonical paymasters are identified by exact runtime code match -- functionally identical code. A bug in the canonical paymaster simultaneously affects every instance. There is no upgrade path (changing code breaks the code-match identity). Attacker-deployed instances can drain via delayed withdrawal, invalidating all pending transactions.

*New text acknowledges secp256k1-only and "may change in later specifications, in which case a new canonical implementation version would be required." But the structural risks are not addressed.*

#### 4.10 [AUDIT H-11] Explicit Sender Enables Nonce Oracle Attacks -- **FIXED (documented)**
An attacker can learn any account's nonce and force nodes to perform state reads by submitting frame transactions claiming arbitrary sender addresses. 10,000 transactions per second with different senders becomes an I/O amplification attack.

*New security section "Explicit Sender State-Read Amplification" with mitigation guidance (structural/stateless checks before state access, peer-level rate limiting). `tx.sender != address(0)` added as static constraint.*

#### 4.11 [TIM] No Replay Protection Across Signature Schemes -- **NOT FIXED**
The signature hash contains no domain separator for the signature scheme being used. Since signature type is in the elided VERIFY data, the same hash is presented to both secp256k1 and P256 verification paths. A future scheme addition could create cross-scheme replay.

*P256 address derivation got a domain tag (`P256_ADDRESS_DOMAIN`), but the signed hash itself still contains no scheme domain separator.*

#### 4.12 [AA] APPROVE Reentrancy via Delegatecall Chains -- **FIXED**
If frame.target DELEGATECALLs to a library that executes APPROVE, `ADDRESS` is still frame.target (delegatecall preserves context), so the check passes. If that library is compromised, the attacker can APPROVE arbitrary transactions. The EIP should explicitly state that APPROVE is valid in delegatecall context and discuss implications.

*Rationale now states: "Because DELEGATECALL preserves ADDRESS, code executed via DELEGATECALL from the resolved target may also execute APPROVE successfully. Contracts that rely on APPROVE should therefore treat delegatecalled libraries as fully trusted."*

### Medium

#### 4.13 [AUDIT M-2] VERIFY Frame Data Malleability -- **FIXED (documented)**
ALL data from ALL VERIFY frames is elided and malleable. Concrete vectors:
- Sponsor griefing: replace sponsor's VERIFY data with garbage
- Custom smart account context manipulation via unsigned VERIFY data
- Paymaster parameter manipulation (fee terms, exchange rates)
- Transaction ID instability

#### 4.14 [AUDIT M-9] Cross-Frame Data Leakage to Paymasters -- **FIXED (documented)**
FRAMEDATALOAD/FRAMEDATACOPY let VERIFY frames read SENDER frame data. A paymaster can inspect user swap parameters (token, amount, slippage) and condition approval on MEV extractability. The paymaster validation runs BEFORE user operations, giving it first-mover advantage.

*New security section "Cross-Frame Data Visibility During Validation" warns users that paymasters can observe SENDER frame data. Recommends treating non-VERIFY frame data as visible to validation logic.*

#### 4.15 [AA] Signature Hash Malleability Window -- **PARTIALLY FIXED (documented)**
During key rotation, different valid signatures could produce the same signature hash. For exotic schemes that embed authorization metadata in signatures (e.g., "this signature is valid but rate-limited"), this malleability could be exploitable.

*New text: "Verification logic MUST NOT assign policy meaning to signature encodings or adjacent VERIFY frame data unless that meaning is independently authenticated." Structural fix not applied, but the risk is now documented.*

#### 4.16 [PA] Approval Scope Bits Create Confused Deputy Risk -- **FIXED (documented)**
The approval scope bits are set by the transaction submitter, not by the smart account. A malicious bundler could construct a transaction with scope bits granting broader approval than the sender intended. Paymaster code must validate scope bits itself before calling APPROVE.

*New text: "allowed_scope is caller-supplied policy input and may restrict a verifier's intended APPROVE call. Verification logic SHOULD authenticate any approval scope it relies on and MUST NOT treat allowed_scope as trusted unless it is covered by that logic."*

#### 4.17 [TIM] Warm/Cold Journal Sharing Leaks Information -- **NOT FIXED**
A VERIFY frame can probe whether storage slots are warm (via gas costs) to learn what previous frames accessed. In the adversarial paymaster model, a malicious paymaster VERIFY frame could infer information about the sender's verification logic.

#### 4.18 [AUDIT M-17] APPROVE Scope Bits as Confused Deputy -- **FIXED**
Mode bits 9-10 are set by the transaction creator, not the target contract. A sender can set mode bits preventing a paymaster from calling its intended APPROVE scope.

*Scope bits moved to separate `flags` field. Confused-deputy risk now explicitly documented with normative guidance.*

#### 4.19 [CX] Default Code Rules Make Third-Party EOA Paymasters Impossible -- **PARTIALLY FIXED**
Default code VERIFY reverts unless `frame.target == tx.sender`. A third-party EOA paymaster cannot approve payment under these rules. This conflicts with the claim that "any EOA" can be a paymaster.

*Default code now checks: `if allowed_scope & APPROVE_EXECUTION != 0 and resolved_target != tx.sender: revert()`. A payment-only scope (`APPROVE_PAYMENT`) no longer requires `resolved_target == tx.sender`, so a third-party EOA can approve payment. But the default code for payment-only scope relies on the flags field allowing it, which must be set by the transaction creator.*

---

## 5. Compelling Usage Scenarios

### 5.1 Post-Quantum Migration Without Flag Day
*[ALL]*

Users deploy smart accounts with PQ verification logic. The default code already supports P256. The signature scheme is account-specific. No hard fork needed for new schemes. This is the first proposal making PQ migration a user-space decision rather than a protocol-level event.

### 5.2 Trustless ERC-20 Gas Payment
*[AA, AF, BA, BK, HK, HP, IL, KC, RS, ST, TIM, TS, VB]*

A user holding only USDC can transact without acquiring ETH. The sponsor validates token balance, pays ETH, receives token compensation, all within a single atomic transaction. No trusted relayers. This eliminates one of the biggest UX barriers.

### 5.3 Atomic Approve-and-Swap
*[AA, AF, BA, BK, DF, HK, HP, IL, KC, PA, RS, ST, TIM, TS, VB]*

The approve-then-swap pattern becomes a single atomic operation. If the swap fails, the approval is reverted. This eliminates dangling approvals -- a class of vulnerabilities that has cost users hundreds of millions of dollars [TS].

### 5.4 First-Transaction Account Deployment
*[AF, BA, BK, DF, HK, HP, IL, KC, PA, ST, TS, VB]*

Deploy a smart account and execute the first operation in a single transaction. The user receives funds at a counterfactual address, and the first transaction they send deploys the account. "Create a character and start playing" [HP].

### 5.5 Passkey/WebAuthn Wallets Without Smart Account Deployment
*[BA, KC, TIM, TS]*

P256 in the default code enables browser-native passkey authentication and mobile secure enclave signing without any contract deployment. ~100M+ existing EOAs gain hardware-backed authentication immediately.

### 5.6 Social Recovery Without Intermediary Contracts
*[AA, AF, HK, HP, PA, TIM, VB]*

Smart accounts implement social recovery directly in validation logic. Multiple VERIFY frames from different guardian addresses, composable threshold requirements. The recovery flow is a single transaction.

### 5.7 Multi-Operation Batching as First-Class
*[AA, AF, BA, HP, PA, TS]*

Approve + swap + stake in one transaction. Claim rewards + restake. Withdraw from multiple positions + deposit into a new one, all atomic. This eliminates entire classes of wrapper contracts (multicall, batch executor, etc.).

### 5.8 Session Keys and Delegated Authorization
*[BA, DF, HP, KC, PA, VB]*

Smart accounts validate session key signatures with limited scope -- time-bounded, operation-bounded, value-bounded. Gaming sessions, DApp-specific keys, automated strategies -- all without persistent private key exposure. This is the "log in once, play for hours" UX.

---

## 6. Additional Possibilities (Things People May Not Be Thinking About)

### 6.1 Protocol-Native Intent System
*[AA, AF, BA, BK, HP, KC, PA, RS, TIM, VB]*

VERIFY frames can introspect subsequent frames via TXPARAM and FRAMEDATALOAD. This means VERIFY can act as an *intent verifier*: the user signs an intent, and the VERIFY frame checks that execution frames satisfy it before calling APPROVE. No off-chain solvers, no trusted matchers. The smart account IS the intent verifier.

### 6.2 Programmable MEV Resistance
*[AA, BA, BK, HP, PA, TIM, TS]*

Smart accounts enforce frame ordering constraints: "the swap frame must immediately follow the approval frame," "no DEFAULT frames between my SENDER frames," "the last frame must be a balance check that reverts if my position decreased." User-level MEV protection enforced by the protocol.

### 6.3 AI Agent Transaction Architecture
*[PA]*

Frame transactions provide a native protocol-level capability system for autonomous agents. VERIFY frames check agent actions against policy smart accounts (spending limits, target whitelists, time constraints). Circuit-breaker contracts check cumulative risk limits. Human overseers attach approval frames afterward for high-value actions.

### 6.4 Composable Authentication Standards as a Market
*[HP]*

Frame transactions create a market for verification modules. "Passkey verifier v2," "Multi-sig verifier with social recovery," "Biometric bridge verifier." Accounts switch between these by updating verification logic. An entirely new category of smart contract.

### 6.5 Governance Without Governor Contracts
*[AA, HP]*

A DAO's smart account implements governance logic: VERIFY collects member signatures (multi-VERIFY frames or aggregated proof), SENDER executes the proposal atomically. Governance reduced to its essential form: collective authorization of a specific action.

### 6.6 Programmable Transaction Policies
*[AA, BK, HP, PA, RS, TS]*

Smart accounts enforce arbitrary policies during VERIFY: spending limits, destination whitelists, time-based restrictions, value-based key tiers. These policies compose, execute before execution (not after), and cannot be bypassed. The "wallet" ceases to be a key management tool and becomes a policy engine [RS].

### 6.7 Cross-Frame Conditional Execution
*[AF, BK, PA, ST, VB]*

TXPARAM(0x15) lets later frames check prior frame status. Frame 0 tries swap on DEX A; frame 1 checks if frame 0 succeeded; if not, tries DEX B. In-transaction routing without a router contract.

### 6.8 Transactions as Legible Narratives
*[RS]*

The frame decomposition provides semantic scaffolding for machine-generated transaction narratives: "Your account was verified using your passkey. GasDAO sponsored the fee. You swapped 100 USDC for 0.04 ETH. GasDAO reclaimed their fee." Transaction history that reads like a story instead of an audit log.

### 6.9 Privacy-Preserving Sponsorship
*[AF, PA]*

VERIFY frame data is elided and invisible to execution frames. A ZK proof that the sender is an authorized user is invisible to on-chain observers. The paymaster approves payment without revealing the relationship between sponsor and sender.

### 6.10 Dead Man's Switch / Estate Planning
*[HP, VB]*

Smart account validation logic includes time-based fallback: "if primary key unused for 12 months, accept recovery key." Estate planning for digital assets as a protocol-level capability rather than a trusted-third-party service.

### 6.11 Account-Level MEV Capture
*[HP]*

A smart account's VERIFY frame implements an auction: accepts any valid signature but only APPROVEs if the frame list includes a payment to the account owner. The account itself becomes an MEV capture mechanism.

### 6.12 Cross-Rollup Transaction Orchestration
*[PA]*

Frame transactions on L1 can coordinate L2 actions: send message to Rollup A, send message to Rollup B, lock collateral on L1 -- all atomic via batching.

### 6.13 Programmable Fee Markets
*[AF, BA, DF, HP, KC, PA, TS]*

The payer abstraction enables application-specific fee pricing: charging more for high-contention state access, less for delayable transactions, subscription-based gas, gas futures, cross-chain gas payment.

### 6.14 Composable Validation Pipelines
*[VB]*

Multiple VERIFY frames chain modular, reusable validators: Frame 0 checks PQ signature, Frame 1 checks rate-limiting oracle, Frame 2 checks spending policy contract. "Lego" composability for authentication.

---

## 7. Recommendations for Changes (Deduplicated)

> **Status as of PR #11521 (branch `frame-compression`):** Items below are annotated with their resolution status.

### Must Fix Before Finalization

| # | Recommendation | Sources | Status |
|---|---------------|---------|--------|
| 1 | **Fix APPROVE scope values in mempool structural rules** (C-1: 0x0 is invalid) | BK, CO, CX, IL, KC, TS, AUDIT | **FIXED** |
| 2 | **Resolve bit numbering inconsistency** with definitive 0-indexed bit layout diagram | BA, BK, KC, PA, ST, VB, AUDIT | **FIXED** |
| 3 | **Fix compute_sig_hash** to operate on a copy (pure function) | BK, DF, IL, KC, ST, TS, VB | **FIXED** |
| 4 | **Enforce low-s for ECDSA** in default code (`s <= secp256k1.n / 2`) | BA, TIM, AUDIT | **FIXED** |
| 5 | **Add ecrecover != address(0) check** in default code | TIM, AUDIT | **FIXED** |
| 6 | **Resolve TXPARAM/TXPARAMLOAD naming** -- pick one name | BK, CO, IL, KC, TS, VB | **FIXED** |
| 7 | **Add bit-width constraints** on gas_limit (`uint64`), fee fields | AUDIT | **PARTIALLY FIXED** -- gas_limit bounded; fee fields not |
| 8 | **Specify APPROVE gas collection semantics** precisely ("from frame.target", upfront max, refund mechanism) | AUDIT | **PARTIALLY FIXED** -- which account and amount now clear; batch charging still unspecified |
| 9 | **Resolve ENTRY_POINT/APPROVE 0xaa collision** -- use different values | IL, KC, TIM, AUDIT | **PARTIALLY FIXED** -- documented as insignificant; values not changed |

### Should Fix

| # | Recommendation | Sources | Status |
|---|---------------|---------|--------|
| 10 | **Reduce MAX_FRAMES to 32-64** and add per-frame intrinsic cost | ALL | **FIXED** -- reduced to 64, per-frame cost 475 gas |
| 11 | **Separate mode into mode + flags fields** in the frame tuple | BK, DF, HK, IL, PA, ST, VB | **FIXED** |
| 12 | **Add frame return data channel** (FRAMERETURNDATA opcode or APPROVE return data) | AA, AF, BA, DF, HK, IL, KC, PA, ST, TS, VB | **NOT FIXED** |
| 13 | **Add optional gas forwarding** between frames (opt-in via flag bit) | BA, DF, HK, HP, PA, ST, TS, VB | **NOT FIXED** |
| 14 | **Specify EIP-7702 interaction** explicitly | BK, DF, HP, KC, ST, TS, VB, AUDIT | **FIXED** |
| 15 | **Define skipped frame receipt semantics** (status, gas_used, TXPARAM visibility) | BK, CO, DF, IL, KC, ST | **NOT FIXED** |
| 16 | **Specify atomic batch gas accounting** (gas consumed by reverted frames is charged; approval flags survive restore) | AF, BA, BK, DF, HK, IL, KC, ST, AUDIT | **NOT FIXED** |
| 17 | **Add domain separator** for signature schemes in default code | TIM, AUDIT | **PARTIALLY FIXED** -- P256 address domain added; sig hash scheme separator not |
| 18 | **Add P256 point validation** before address derivation | TIM, AUDIT | **PARTIALLY FIXED** -- noted in text; not in normative pseudocode |
| 19 | **Specify APPROVE return data** handling explicitly | BA, DF, IL, KC, ST, VB | **NOT FIXED** |
| 20 | **Define canonical paymaster** as normative artifact or companion EIP | AF, CO, CX, DF, HK, HP, KC, PA, ST, TS, VB, AUDIT | **NOT FIXED** |
| 21 | **Add TXPARAM for individual blob versioned hash access** | BA, BK, HK, IL, ST | **NOT FIXED** -- but note added that `BLOBHASH(index)` opcode already provides this |
| 22 | **Mandate TXPARAM(0x08) caching** or add variable gas cost | BA, BK, ST, TS | **FIXED** -- "MUST be computed at most once per transaction and cached" |
| 23 | **Define "known deterministic deployer"** as explicit list | DF, HK | **FIXED** -- now EIP-7997 deterministic factory predeploy |
| 24 | **Add `tx.sender != address(0)` constraint** | TIM, AUDIT | **FIXED** |
| 25 | **Explicitly state VERIFY/APPROVE static exception** | AF, BA, BK, DF, HK, KC, RS, ST | **FIXED** |
| 26 | **Address default code multicall vs. multi-frame transparency asymmetry** -- document that VERIFY policy enforcement requires multi-frame patterns, or add introspection for default code sub-calls | Gap analysis | **NOT FIXED** |

### Should Consider

| # | Recommendation | Sources | Status |
|---|---------------|---------|--------|
| 27 | **Add frame-level value field** (optional, default 0) | AA, AF, BA, BK, HK, HP, IL, KC | **NOT FIXED** (by design) |
| 28 | **Preserve transient storage within atomic batches** | BA, BK, HP, VB | **NOT FIXED** |
| 29 | **Separate mempool policy** into companion document | BK, CO, HK, HP, VB | **NOT FIXED** |
| 30 | **Add frame version field** for future extensibility | DF, IL, ST, TS | **NOT FIXED** |
| 31 | **Relax non-canonical paymaster limit** with staking or raise to 4-8 | AA, HP, KC, RS, VB | **NOT FIXED** |
| 32 | **Define named transaction templates** (SimpleTransaction, SponsoredTransaction, etc.) for legibility | RS | **NOT FIXED** |
| 33 | **Include revert data** in frame receipts (first N bytes of revert reason) | AA, DF | **NOT FIXED** |
| 34 | **Add per-frame intrinsic cost** (e.g., 500-1000 gas per frame) | HK, TS, AUDIT | **FIXED** -- 475 gas per frame |
| 35 | **Add frame annotation field** for human-readable step descriptions | RS | **NOT FIXED** |
| 36 | **Reserve DELEGATE mode** (mode 3) for future delegatecall-style frames | ST, TS, VB | **NOT FIXED** |
| 37 | **Define multidimensional nonce semantics** now, not as future extension | AA, AUDIT | **NOT FIXED** |
| 38 | **Add FRAMEDATASIZE opcode** to complete the trio with FRAMEDATALOAD/FRAMEDATACOPY | IL | **NOT FIXED** |
| 39 | **Add explicit frame dependency declarations** for future parallel execution | HK, PA, VB | **NOT FIXED** |
| 40 | **Consider making TXPARAM return 0 for unknown params** instead of exceptional halt (forward-compatible) | DF, VB | **NOT FIXED** |
| 41 | **Add lightweight pre-validation** for explicit sender field (rate-limit, require existing account) | AUDIT | **PARTIALLY FIXED** -- documented with mitigation guidance; no structural mechanism |

---

## Audit Findings Summary

From the multi-domain security audit (55 deduplicated findings):

| Severity | Count | Key Clusters |
|----------|-------|-------------|
| Critical | 1 | APPROVE(0x0) specification self-contradiction |
| High | 12 | Gas overflow, signature malleability, nonce grinding, canonical paymaster risks, explicit sender DoS |
| Medium | 25 | ORIGIN change, VERIFY malleability, transient storage, gas isolation, atomic batch ambiguity, EIP-7702 interaction |
| Low | 14 | No expiration, address collision, RLP canonicality, param gaps |
| Info | 3 | Paymaster nonce, non-canonical limit, no ERC-1271 equivalent |

### Cross-Cutting Risk Clusters (from audit)

1. **Specification Consistency Crisis** [C-1 + M-8 + M-14 + M-7]: No two independent implementations would produce identical results from the current spec. -- **MOSTLY FIXED** *C-1 and M-8 resolved. M-14 (default SENDER-mode subcalls) and M-7 (calldata_cost precision) still open.*

2. **Gas Accounting Attack Surface** [H-2 + H-3 + H-7 + M-5 + M-12]: Integer overflow, underspecified fee collection, overflow in max cost, ambiguous refund, unspecified atomic batch gas accounting. -- **PARTIALLY FIXED** *H-2 fixed (overflow bounds). H-3 partially fixed. H-7 partially fixed. M-5 and M-12 still open.*

3. **Mempool DoS Amplification** [H-1 + H-6 + H-11 + H-12 + M-18 + M-19]: 100K validation budget (too large for DoS, too small for PQ), zero-cost grinding, sender state reads, paymaster Sybil amplification. -- **PARTIALLY FIXED** *H-11 documented with mitigations. H-1, H-6, H-12, M-18, M-19 still open.*

4. **Signature Malleability Surface** [H-4 + M-2 + M-3]: Both ECDSA and P256 signatures are malleable, and ALL VERIFY frame data is unsigned. -- **MOSTLY FIXED** *H-4 (ECDSA low-s) fixed. M-2 (VERIFY malleability) documented. M-3 (P256 low-s) not addressed.*

5. **Inconsistent Isolation Model** [M-10 + M-11 + M-13 + M-9]: Transient storage discarded but warm/cold journal shared. Frame data cross-readable but VERIFY data returns zeros. Atomic batch restores state but may not restore journals. -- **PARTIALLY FIXED** *M-9 documented. M-10, M-11, M-13 still open.*

6. **Ambition vs. Constraint Gap** [H-1 + H-6 + M-11]: PQ migration motivation undermined by 100K gas cap; full AA undermined by single nonce; gas efficiency undermined by rigid per-frame allocation. -- **NOT FIXED**

---

## Gap in Review Coverage: Default Code Multicall vs. Multi-Frame Transparency to VERIFY -- **NOT ADDRESSED in PR #11521**

*This issue was not explicitly covered by any of the 18 review sources. Several reviewers touched adjacent pieces -- KC and ST flagged the default code's all-or-nothing revert behavior; AA, BK, HP, and PA described VERIFY-based policy enforcement assuming multi-frame patterns -- but none identified the transparency asymmetry between the two approaches.*

### The Two Patterns

EOA users have two ways to express multi-operation transactions:

**Pattern A: Multiple SENDER frames** (each operation is a separate frame)
```
Frame 0: VERIFY  -> APPROVE(0x3)
Frame 1: SENDER  -> target=TokenContract, data=approve(DEX, 100)
Frame 2: SENDER  -> target=DEX, data=swap(Token, 100, minOut)
```

**Pattern B: Single SENDER frame with default code multicall** (operations packed as RLP sub-calls)
```
Frame 0: VERIFY  -> APPROVE(0x3)
Frame 1: SENDER  -> target=null(→sender), data=RLP([[TokenContract, 0, approve(...)],
                                                      [DEX, 0, swap(...)]])
```

### How They Look to VERIFY

**Pattern A is fully transparent.** VERIFY can inspect each operation structurally:
- `TXPARAM(0x11, 1)` returns `TokenContract` directly
- `TXPARAM(0x11, 2)` returns `DEX` directly
- `TXPARAM(0x09, 0)` shows 3 frames (the actual operation count)
- `FRAMEDATALOAD(1, 0)` / `FRAMEDATALOAD(2, 0)` give per-operation calldata

This is the basis for all the policy/intent/MEV-protection patterns described by multiple reviewers [AA, BK, HP, PA]: "all SENDER frames must target contracts in my whitelist," "total value across all frames must not exceed my daily limit," "the swap frame must immediately follow the approval frame."

**Pattern B is opaque at the structured level.** VERIFY sees:
- `TXPARAM(0x11, 1)` returns `tx.sender` (the multicall wrapper, not the actual targets)
- `TXPARAM(0x09, 0)` shows 2 frames (hiding that there are 2+ sub-operations)
- `FRAMEDATALOAD(1, 0)` returns raw RLP bytes -- the actual call targets and calldata are buried inside

To discover what Pattern B actually does, the VERIFY frame must RLP-decode the frame data. **There is no native RLP parsing in the EVM.** This means the VERIFY frame would need to implement byte-by-byte RLP decoding in EVM bytecode: reading length prefixes, handling variable-length encoding of addresses/values/data, iterating over a list of unknown length. This is:

- **Prohibitively expensive.** RLP parsing of a multi-call list with variable-length calldata fields can easily consume tens of thousands of gas. Within the 100K MAX_VERIFY_GAS budget, this competes directly with the signature verification itself.
- **Extremely fragile.** Hand-rolled RLP parsing in EVM assembly is a rich source of bugs -- off-by-one errors in length prefix handling, incorrect short-vs-long form detection, missing bounds checks on nested structures.
- **Tightly coupled to the default code format.** The VERIFY logic must hardcode knowledge of the default code's specific `[[target, value, data], ...]` encoding. If the default code encoding ever changes (new fields, different tuple arity), every smart account's VERIFY logic that parses it breaks.
- **Not what TXPARAM was designed for.** The entire point of TXPARAM and FRAMEDATALOAD is structured introspection -- giving VERIFY clean, typed access to frame metadata. Forcing VERIFY to fall back to raw byte parsing defeats that design.

In practice, no reasonable VERIFY implementation would attempt this. The result is that default code multicall operations are effectively **uninspectable by VERIFY**.

### Why This Matters

1. **Policy enforcement is bypassed.** A smart account enforcing "only calls to whitelisted contracts" via `TXPARAM(0x11, i)` is trivially circumvented by packing operations into a single default-code multicall frame. The structural inspection that makes programmable transaction policies powerful [AA, HP, PA, RS, TS] only works on multi-frame patterns.

2. **Intent verification breaks down.** The intent-verifier pattern [AA, AF, BA, BK, PA] -- where VERIFY checks that execution frames satisfy user constraints before calling APPROVE -- assumes each operation is visible as a separate frame. A single opaque multicall frame defeats this.

3. **The patterns have opposing incentives.** Multiple SENDER frames are more transparent but more expensive (per-frame gas overhead, no gas sharing, separate estimation per frame). A single multicall frame is cheaper and simpler but sacrifices the structured inspectability that makes the frame model valuable. Users are economically incentivized toward the less-inspectable pattern.

4. **Smart accounts don't have this problem.** Custom smart account SENDER code can expose sub-operations to its own VERIFY logic however it chooses. The asymmetry only affects EOAs using default code -- the vast majority of accounts.

### The Question: Should They Look the Same to VERIFY?

The EIP should address this explicitly. Three possible approaches:

**Option 1: Make default code multicall inspectable.** Add TXPARAM parameters that return sub-call metadata from the default code's RLP decoding (sub-call count, their targets, values). This would require the protocol to "understand" the default code's encoding format at the introspection level, blurring the line between protocol and default code.

**Option 2: Recommend multi-frame for policy enforcement.** Document that the single-frame multicall is a convenience/efficiency mode that trades VERIFY inspectability for lower gas cost. Make the tradeoff explicit so wallet developers and smart account designers can make informed choices. Policy-enforcing VERIFY logic should require multi-frame patterns.

**Option 3: Normalize introspection.** Add a TXPARAM that exposes "resolved sub-call count and targets" regardless of whether operations are expressed as multiple frames or as a single multicall frame. This preserves the efficiency of the multicall path while maintaining the transparency of the multi-frame path.

Option 2 is the lightest touch. Option 3 is the most complete but adds complexity. Option 1 is a middle ground. At minimum, the EIP should acknowledge the asymmetry and provide guidance, since multiple reviewers built their most compelling use cases (programmable policies, intent verification, MEV resistance) on the assumption that VERIFY can see each operation as a first-class frame.

---

## Overall Assessment

EIP-8141 is the most architecturally significant account abstraction proposal Ethereum has produced. The frame abstraction is the correct primitive -- it decomposes transactions into their natural phases and makes each phase independently programmable. The design is credibly neutral: it does not privilege any signature scheme, gas payment model, or account architecture.

**What it gets right:**
- The frame model is the correct level of abstraction
- VERIFY-as-STATICCALL with data elision is elegant
- The APPROVE opcode with scope separation is well-designed
- Default code for EOAs is the correct migration strategy
- The mempool rules are thorough and battle-informed
- Atomic batching solves a real problem cleanly

**What needs work before advancing (updated per PR #11521):**
- ~~Specification bugs (APPROVE scopes, bit numbering, naming inconsistencies, compute_sig_hash mutation)~~ **Mostly fixed.** APPROVE scopes, bit numbering, naming, and compute_sig_hash all resolved. Remaining: APPROVE return data, skipped frame receipts, FRAMEDATACOPY formula.
- ~~Parameter tuning (MAX_FRAMES too high, MAX_VERIFY_GAS vs. PQ ambitions)~~ **Partially fixed.** MAX_FRAMES reduced to 64 with per-frame cost. MAX_VERIFY_GAS (100K) vs. PQ still unresolved.
- ~~Missing specifications (canonical paymaster, EIP-7702 interaction, skipped frame semantics, atomic batch gas accounting)~~ **Partially fixed.** EIP-7702 interaction fully specified. Canonical paymaster, skipped frame semantics, and atomic batch gas accounting still missing.
- ~~Cryptographic hygiene (canonical S values, zero-address checks, P256 point validation, domain separators)~~ **Mostly fixed.** Low-s, zero-address, P256 address domain all added. Cross-scheme replay domain separator in sig hash and P256 low-s still absent.
- Design gaps (cross-frame communication channel, gas forwarding mechanism, frame-level value field) **Not fixed.** These remain open.

**The consensus across all 18 sources:** The core design is sound. The frame transaction is the right long-term answer to account abstraction on Ethereum. It should proceed, but with the mandatory specification fixes and parameter adjustments identified here. The foundation is solid. The edges need sharpening.

As one reviewer put it: "The most exciting thing about this EIP is not what it enables today. It is that it creates a substrate for authentication and authorization innovation that we cannot fully predict. The best infrastructure is infrastructure that serves use cases its designers never imagined." [VB]

---

*This combined review synthesizes 17 independent reviews and 1 multi-domain security audit, plus post-synthesis gap analysis of the default code multicall vs. multi-frame transparency asymmetry. All findings are attributed to their original sources. Where observations were made independently by multiple reviewers, the convergence is noted as a measure of confidence.*
