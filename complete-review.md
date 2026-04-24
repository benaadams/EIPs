# Complete Review: EIP-8141 "Frame Transaction"

## Scope And Method

This document combines:

- `audits/eip-8141/AUDIT-REPORT.md`
- `review-aa.md`
- `review-af.md`
- `review-ba.md`
- `review-bk.md`
- `review-co.md`
- `review-cx.md`
- `review-df.md`
- `review-hk.md`
- `review-hp.md`
- `review-il.md`
- `review-kc.md`
- `review-pa.md`
- `review-rs.md`
- `review-st.md`
- `review-tim.md`
- `review-ts.md`
- `review-vb.md`

Repeated observations are merged and called out as repeated. Source-specific detail that does not cleanly deduplicate is preserved in the relevant themed section or in the appendices. Shorthand below uses the filename suffixes: `audit`, `aa`, `af`, `ba`, `bk`, `co`, `cx`, `df`, `hk`, `hp`, `il`, `kc`, `pa`, `rs`, `st`, `tim`, `ts`, `vb`.

## Overall Verdict

The combined view is unusually consistent on the main point: EIP-8141 is widely seen as the right long-term primitive for native account abstraction, signature agility, gas sponsorship, and atomic multi-step execution. Most reviewers think the frame model is materially better than ad hoc AA layering around legacy transactions.

The same source set is also unusually consistent on readiness: the current draft is not ready to finalize as written. Even the most supportive reviews repeatedly ask for another hardening pass on approval semantics, gas accounting, paymaster specification, default-code cryptography, bit numbering, compatibility notes, and several underspecified execution details. The audit sharpens that verdict into a concrete security posture: 55 deduplicated findings, including 1 critical and 12 high.

The fairest merged summary is:

- Architecture: strong approve.
- Current draft quality: needs hardening before standardization.
- Shipping posture: no-ship as written for strict implementers; broader review and devnet experimentation only after the blocking contradictions are fixed.

## Grade Summary Table

No source used explicit letter grades, so this table normalizes the stance language into a common format.

| Source | Lens | Architecture verdict | Current-draft verdict | Short take |
|---|---|---|---|---|
| `audit` | security audit | promising but constrained | no-ship as written | 55 findings; approval, gas, crypto, paymaster, and mempool blockers |
| `aa` | deep protocol review | strong approve | ship after hardening | strongest praise for the frame model; hardest pressure on parameters and semantics |
| `af` | architectural review | strong approve | phased rollout | right direction, but complexity argues for staged adoption |
| `ba` | implementation / perf | approve | needs hardening | architecture is sound; cache `compute_sig_hash`, simplify gas and limits |
| `bk` | systems-level review | strong approve | needs hardening | serious proposal; fix contradictions and underspecified edges |
| `co` | design review | approve | not ready as-is | split core semantics from mempool and canonical-paymaster policy |
| `cx` | adversarial review | directionally positive | needs-attention / no-ship | blocking contradictions in approval, paymaster, and gas-accounting semantics |
| `df` | architecture / API shape | approve | needs hardening | strong abstraction, but overloaded fields and missing result channels |
| `hk` | conceptual review | approve | needs hardening | very positive on structure; wants sharper semantics and compatibility text |
| `hp` | ecosystem / migration review | approve | needs hardening | generational upgrade if gas, migration, and paymaster details are fixed |
| `il` | API design review | strong approve | needs cleanup before standardizing | permanent public API deserves better names, field boundaries, and exactness |
| `kc` | QA / comprehensive review | strong approve | fix critical inconsistencies first | right primitive, but too many client-divergence traps remain |
| `pa` | protocol architecture review | strong approve | move forward after prerequisites | major protocol step, but not yet ready for promotion beyond Draft |
| `rs` | behavioral / UX review | approve | needs hardening | valuable design that still undersells complexity and user-legibility risks |
| `st` | technical review | strong approve | ship after spec cleanup | "fix the bit numbering first" is representative of the stance |
| `tim` | security / cryptography review | strong approve | proceed after crypto tightening | first credible PQ path, but default-code hygiene needs work |
| `ts` | client / runtime review | strong approve | fix mempool and spec bugs first | strongest on implementation realism and concurrency / revalidation costs |
| `vb` | design philosophy review | strong approve | broader review after underspecified pieces are filled in | probably the most complete L1 AA proposal so far |

## Emerging Consensus Points

### Positive Consensus

| Point | Strength | Repeated by |
|---|---|---|
| The frame abstraction is the right primitive and the right protocol layer. | Near-unanimous | `aa af ba bk co cx df hk hp il kc pa rs st tim ts vb` |
| `VERIFY` plus explicit `APPROVE` is a cleaner authorization model than inferring success from return values. | Strong majority | `aa af ba co df hk hp kc pa st tim ts vb` |
| Excluding `VERIFY` data from the signature hash is elegant and future-proof. | Strong majority | `aa af ba bk co df hk hp il kc pa rs st tim ts` |
| Default code for EOAs, including P256/passkey support, is the correct migration bridge. | Strong majority | `aa af ba bk co hk hp il kc pa rs tim ts vb` |
| Atomic batching is one of the clearest user-security wins. | Near-unanimous | `aa af ba bk co df hk hp il kc pa rs st tim ts vb` |
| Sender approval and payer approval should be separate concerns. | Strong majority | `aa af ba df hk hp kc pa rs st tim ts vb` |
| The mempool section shows genuine ERC-4337 / ERC-7562 operational learning. | Majority, with caveats | `af bk co il kc pa tim ts vb` |

### Risk Consensus

| Point | Strength | Repeated by |
|---|---|---|
| The current draft is not implementation-ready as written. | Near-unanimous | `audit aa af ba bk co cx df hk hp il kc pa rs st tim ts vb` |
| Approval semantics are contradictory and currently break sponsored flows. | Near-unanimous | `audit bk co cx df hk hp il kc st ts` |
| Gas accounting is underspecified and economically awkward. | Near-unanimous | `audit aa af ba co cx df hk hp il kc pa rs st tim ts vb` |
| `MAX_FRAMES = 1000` is unjustified and expands attack surface. | Strong majority | `audit aa af ba bk df hk hp kc rs st tim ts` |
| The canonical paymaster is underdefined yet load-bearing. | Strong majority | `audit cx kc pa st tim ts vb co hk hp` |
| Default-code cryptography needs tightening. | Strong majority | `audit tim ts st vb aa bk` |
| The `mode` / scope / flag packing is too clever and inconsistently described. | Strong majority | `audit af ba bk df il kc pa rs st vb` |
| `ORIGIN` changes are a compatibility landmine unless documented much better. | Broad majority, but not unanimous | `audit aa af bk co df hk hp il kc pa rs st ts` |
| Transient storage and frame-isolation semantics are surprising and underexplained. | Broad majority | `audit aa af ba bk kc pa st tim ts vb` |
| `compute_sig_hash` and `TXPARAM` have spec and performance issues. | Broad majority | `ba bk df il st ts vb audit` |
| EIP-7702 interaction is still unclear. | Broad majority | `audit bk st tim ts vb co hp` |

## Merged Observation Register

### 1. Architecture The Sources Consistently Want To Keep

- The strongest shared claim is that a transaction should be an ordered program of validation, payment, deployment, and execution frames rather than a legacy transaction plus wallet-specific ceremony. Repeated by `aa af ba bk co cx df hk hp il kc pa rs st tim ts vb`. `vb` frames this as credible-neutral protocol design, `hk` as turning the transaction into a score/program, `pa` as a general coordination language, and `aa` as a platform for intents, MEV resistance, governance flows, and one-transaction key rotation.
- The `VERIFY` / `APPROVE` split is widely seen as a real improvement over ERC-4337-style `validateUserOp` patterns because consent becomes an explicit opcode-level act instead of a return-value convention. Repeated by `aa af ba df hk hp kc pa st tim ts vb`. `tim` especially likes that this removes an accidental-approval class of bugs.
- Signature-hash elision for `VERIFY` frames is one of the cleanest pieces of the design. Repeated by `aa af ba bk co df hk hp il kc pa rs st tim ts`. Reviewers repeatedly cite the same benefits: it avoids self-reference in signatures, lets sponsor/paymaster signatures be attached after the sender signs, and preserves future room for aggregation or alternate schemes.
- Default code for EOAs is treated as essential, not optional. Repeated by `aa af ba bk co hk hp il kc pa rs tim ts vb`. The repeated rationale is that it lowers adoption friction, keeps existing EOAs relevant, and gives passkeys/P256 a usable bridge without forcing every account into an up-front contract deployment.
- Atomic batching is one of the few features that every source can point to with an immediately legible user-security win. Repeated by `aa af ba bk co df hk hp il kc pa rs st tim ts vb`. The approve-then-swap / dangling-approval example appears in many forms and is close to the document set's canonical positive example.

### 2. Blocking Contradictions And Consensus-Safety Risks

- The clearest blocker is the `APPROVE` scope contradiction. Repeated by `audit bk co cx df hk hp il kc st ts`. The current draft says `only_verify` must call `APPROVE(0x0)` even though `0x0` is invalid, and maps `pay` to `APPROVE(0x1)` even though `0x1` is sender approval. `cx` treats this as an immediate no-ship issue; the audit elevates it to `[C-1]`; `ts` flags it from an implementer/mempool realism perspective; `il` and `kc` want the mapping regenerated from one source of truth and then propagated through examples, structural rules, and bytecode.
- Bit numbering and mode-field semantics are not precise enough for consensus-critical text. Repeated by `audit af ba bk df il kc pa rs st vb`. The recurring complaint is that the lower 8 bits are an execution mode while upper bits carry scope constraints and the atomic flag, but the prose flips between 0-indexed and 1-indexed descriptions. `st` calls this potentially consensus-breaking, `il` recommends explicit named constants or a diagram, and several reviewers prefer separate `mode` and `flags` fields outright.
- The spec still mixes normative behavior, explanatory prose, and implementation hints too casually. Repeated by `bk co hk hp vb`, reinforced by audit findings `M-7`, `M-14`, and `L-6`. Reviewers repeatedly call out `TXPARAM` versus `TXPARAMLOAD`, contradictory comments around frame count and parameter numbering, and informal statements that are doing consensus-critical work.
- The pseudocode style is itself unsafe in places. Repeated by `ba df st ts vb`, with audit reinforcement through the broader consistency cluster. The most repeated example is `compute_sig_hash` mutating the transaction object in place. Multiple reviewers independently ask for pure, copy-based wording so implementers do not literally erase `VERIFY` data.

### 3. Gas Accounting And The Economic Model

- The largest thematic cluster after approval semantics is gas. Repeated by `audit aa af ba co cx df hk hp il kc pa rs st tim ts vb`. The common complaint is not just "gas is hard"; it is that the draft simultaneously changes execution accounting, paymaster accounting, refund behavior, and sometimes even implicit block-gas accounting without pinning those changes down tightly enough.
- The audit's harshest gas points are structural: `H-2` on overflow in `sum(frame.gas_limit)`, `H-3` on undefined fee collection semantics, `H-7` on `TXPARAM(0x06)` max-cost overflow, `M-5` on refund underflow ambiguity, and `M-12` / `M-13` on reverted-batch charging and journal restoration. `cx` independently reaches the same block-gas concern and treats the "add unused gas back to the block gas pool" idea as an undefined consensus rule.
- Per-frame gas isolation is widely viewed as defensible in theory and painful in practice. Repeated by `audit aa af ba co df hk hp kc pa rs st tim ts vb`. The repeated problem statement is that every frame must be over-provisioned independently, unused gas cannot help later frames, and the payer has to reserve capital against the worst case for each frame. `vb` explores a gas-pool alternative, `aa` and `ba` ask for gas forwarding or donation, and the audit recommends at least a per-frame intrinsic charge plus explicit documentation of the over-allocation tradeoff.
- `MAX_VERIFY_GAS = 100,000` is seen as self-contradictory relative to the PQ story. Repeated by `audit tim ts` and echoed conceptually by several architecture reviews. The audit frames it as both too high for free mempool validation and too low for most PQ or ZK-based verification paths. `tim` makes the same point from a cryptography lens and explicitly notes that some realistic PQ verification paths would not fit.
- `MAX_FRAMES = 1000` is one of the most repeated single-parameter complaints in the entire source set. Repeated by `audit aa af ba bk df hk hp kc rs st tim ts`. The shared critique is that examples rarely need anywhere near 1000 frames, but the higher cap expands parsing, journaling, receipt, snapshot, and test surface dramatically. Suggested replacements cluster around `16`, `32`, or `64`, or at least "justify it and make it future-adjustable."
- Several sources add non-audit economic details that are worth preserving: `hp` flags ERC-20 paymaster refund economics where the paymaster may both charge the user in tokens and keep the ETH refund; `ba` questions whether the intrinsic fee already double-charges costs paid inside frames; `co` and `cx` want a much more formal description of what reclaimable reserved gas means for block validity and packing.

### 4. Cryptography, Signature Semantics, And Default Code

- The review set strongly likes the protocol's signature agility while simultaneously arguing that the current default-code cryptography is not production-tight. Repeated by `audit tim ts st vb aa bk`.
- The audit's most concrete crypto findings are `H-4` secp256k1 low-`s` malleability, `H-5` missing zero-address rejection from `ecrecover`, `M-3` P256 malleability, `M-6` ordering gaps around P256 point validation, `L-2` missing curve domain separation in address derivation, `L-3` possible omission of the typed-transaction prefix from the signed hash, and `M-4` weakly specified chain-id enforcement.
- `tim` adds the richest crypto-specific detail: invalid-curve / point-validation risk for P256, missing point compression, cross-scheme replay concerns if signature type is fully elided from the signed domain, and the canonical paymaster's brittle use of verbatim assembly. `ts` reinforces low-`s`, point validation, and broader replay/domain-separation concerns. `st` and `vb` mostly arrive from a spec-quality angle but support the same hardening direction.
- Several implementation reviews connect cryptography back to performance. `ba` and `bk` both single out `TXPARAM(0x08)` and `compute_sig_hash` as likely DoS or latency hotspots unless the value is cached, repriced, or both. That concern is not just theoretical; it is tied to repeated frame introspection during validation and execution.

### 5. Paymasters, Mempool Design, And DoS

- A major theme is that the mempool and paymaster model is sophisticated, but too much of it is bundled into the core EIP and too much of it depends on a "canonical paymaster" that the draft does not actually specify tightly enough. Repeated by `audit co cx hk hp kc pa st tim ts vb`.
- `cx` is the sharpest adversarial statement of the problem: approval semantics are wrong, the canonical paymaster is undefined, and block-gas reclaim semantics are underspecified. `kc`, `st`, `vb`, and the audit all independently argue that the canonical paymaster is load-bearing enough that it either needs to be fully normative here or moved into a required companion spec/EIP.
- The audit adds concrete operational risk details that other reviews mostly imply: `H-9` centralization and identical-code shared-bug risk, `H-12` node-local balance reservations, `M-18` non-canonical-paymaster Sybil amplification, `M-19` reorg-driven revalidation amplification, and `I-2` the practical uselessness of `MAX_PENDING=1` for shared gas accounts.
- Review-specific additions deepen the same theme. `aa` proposes stake-based scaling instead of a hard one-transaction ceiling for non-canonical paymasters. `hp` wants a clearer ERC-4337 migration path and flags the economics of paymasters charging users in one asset and settling/refunding in another. `rs` says the mempool section is effectively a security spec and should be presented with that gravity, not as soft policy text.
- The explicit `sender` field changes the threat model in ways several sources dislike. Repeated by `audit af ba bk`, with related concerns in `tim`. The combined complaint is that a peer can force arbitrary sender nonce/state reads before cryptographic legitimacy is known, creating a nonce-oracle and trie-I/O amplification surface. The audit adds `H-11` on explicit sender state-read abuse and `H-6` on late nonce increment enabling repeated failed validation without consuming the nonce.
- Multiple sources argue that the public mempool policy should be at least partially split from the transaction core. Repeated by `af co cx hk hp`. The usual reason is governance and evolution: core transaction semantics should ossify more slowly than relay policy, banned-opcode rules, and canonical-paymaster heuristics.

### 6. Compatibility, Ergonomics, And API Design

- `ORIGIN` is one of the few issues where the source set shows real disagreement rather than just differences in emphasis. `tim` and `vb` are positive on the semantic change and think it reduces the long-standing `tx.origin` footgun. A larger set - `audit aa af bk co df hk hp il kc pa rs st ts` - argues the change is materially under-documented and will silently break contracts, analytics, bot logic, and heuristics that still assume the old meaning. The merged takeaway is not "do not do it"; it is "if you do it, promote it to a first-class compatibility section with worked examples and tests."
- Transient storage and isolation semantics are another major ergonomics complaint. Repeated by `audit aa af ba bk kc pa st tim ts vb`. The recurring point is that warm/cold access state is shared, transient storage is not, `VERIFY` frame data is hidden, other frame data is visible, and atomic rollback may or may not restore journals. `pa` says this destroys the most natural cross-frame communication channel; `tim` says it will surprise developers who expect EIP-1153 transaction-scoped behavior; the audit treats the overall isolation model as internally inconsistent.
- API-surface criticism is deepest in `il`, but many others reinforce it. `il` argues that `TXPARAM` is effectively a "god method", that `in2` is a meaningless name, that the opcode family naming is inconsistent, and that `mode` packs identity, permissioning, and composition into one field. `df`, `st`, `ts`, `vb`, `ba`, and the audit add related complaints about numbering gaps, harsh exceptional halts, hidden flags, and naming drift (`TXPARAM` versus `TXPARAMLOAD`).
- Several reviewers want an explicit cross-frame return-data or result channel. Repeated by `af ba df il kc pa st ts`, with adjacent concerns in `aa`. The common complaint is that the frame structure is powerful, but the protocol gives weak native tools for passing structured results, revert data, or error codes between frames. Suggestions include a `FRAMERETURNDATA`-style surface, richer status and gas introspection, frame-level error codes, and explicit revert data in frame receipts.
- Default SENDER-mode behavior and receipt semantics need more exact text. The audit contributes `M-14`, `M-24`, `L-10`, and related receipt findings; `st` and `vb` ask about skipped-frame statuses and missing bloom semantics; `aa` asks for revert data and standardized frame events; several reviews also want a frame-level `value` field because ETH transfers feel awkward and byte-heavy without one.
- The source set praises both default-code EOA multicall and multi-frame `SENDER` execution, but rarely compares them directly. The spec clearly allows EOAs to pack many calls into one default-code `SENDER` frame via RLP-decoded `[[target, value, data]]`, while also allowing the same high-level workflow to be expressed as multiple `SENDER` frames. The important point, mostly implicit in the reviews, is that these are not protocol-equivalent encodings: `VERIFY` can distinguish them via `TXPARAM` / `FRAMEDATALOAD`, multi-frame execution has per-frame gas/status/receipt/atomic-batch structure, and default-code multicall has one outer frame with under-specified inner subcall semantics. More importantly, under the current design any policy-bearing `VERIFY` or paymaster logic that needs to reason about the inner calls of the packed EOA form would have to parse that RLP itself inside the validation gas budget, which is in direct tension with the EIP's bounded-validation goal. If the packed form is intended to be equally usable for validation-sensitive flows, the protocol likely needs to expose those inner calls as semantic objects rather than opaque bytes; otherwise the spec should say explicitly that per-operation introspection requires explicit multiple `SENDER` frames. This distinction is supported indirectly by `aa`, `ba`, `bk`, `pa`, `vb`, `hk`, `rs`, `st`, `ts`, and by audit findings `M-14` and `M-24`, but it should be documented explicitly in the EIP.
- EIP-7702 interaction remains unclear enough that it now qualifies as a repeated omission rather than a niche question. Repeated by `audit bk st tim ts vb`, with supporting mentions in `co` and `hp`. The common issue is whether delegated EOAs execute delegated code or default code in frame transactions, and whether that undermines the clean PQ / signature-agility story the EIP is trying to tell.
- A small but important compatibility cluster revolves around `ENTRY_POINT = 0xaa`. `audit` raises `H-10`, `M-21`, and `L-12`; `hk` and `tim` independently call the collision between opcode value `0xaa` and caller address `0xaa` needlessly confusing and error-prone.

### 7. Unique Expansions Worth Preserving Even Where Consensus Is Weaker

- `aa` contributes several forward-looking capability ideas that are not fully duplicated elsewhere: native intents, programmable MEV protection, governance without wrapper governor contracts, single-transaction key rotation, and a warning about delegatecall-based `APPROVE` authority inheritance. It also adds concrete asks for revert data, frame-level indexing, and better receipt surfaces.
- `af` is the strongest advocate for phased rollout. It argues for shipping core frame machinery first and layering more aggressive canonical-paymaster / mempool machinery later. It also contributes optionality ideas such as a `SKIP` mode, frame priority hints, micro-frames, cross-transaction context, and validation-scoped transient storage.
- `pa` broadens the use-case story into AI agents, identity proofs, institutional / mechanism-design style transactions, and cross-rollup orchestration. Even if these are not near-term design drivers, they help explain why multiple reviewers think the frame abstraction belongs at L1 rather than in account-specific middleware.
- `rs` adds the best UX / legibility argument in the set: frames create narrative structure, but that value is lost unless wallets and tooling can present them as named templates, annotated steps, and human-legible simulations instead of raw frame arrays.
- `vb` is the most aggressive about complexity budget and extension discipline. It explicitly asks whether `FRAMEDATALOAD` / `FRAMEDATACOPY` should exist at all, whether unknown `TXPARAM` values should return zero instead of halting, and whether the design is spending too much permanent protocol complexity at once.
- `hp` presses hardest on migration and operational realism: explicit ERC-4337 migration text, proxy / `DELEGATECALL` interaction with the `ADDRESS == frame.target` rule, and ERC-20 paymaster refund asymmetries.
- `tim` contributes the richest capability catalog outside the main critique path: HSM integration, threshold signatures, social recovery, programmable transaction introspection, intent architectures, and lighter-weight validation for light clients. Those examples support the larger pro-architecture case even while the document remains tough on crypto hygiene.

## Prioritized Merged Recommendations

> **Status as of PR #11521 (branch `frame-compression`):** Items below are annotated with their resolution status.

### Must Fix Before Broader Review Or Devnet

1. **FIXED** -- Normalize `APPROVE` semantics from one source of truth, then regenerate structural rules, examples, and canonical-paymaster bytecode from that mapping. *Scopes are now named constants (`APPROVE_PAYMENT=0x1`, `APPROVE_EXECUTION=0x2`, `APPROVE_PAYMENT_AND_EXECUTION=0x3`). All mempool rules, examples, and pseudocode regenerated from the new mapping.*
2. **PARTIALLY FIXED** -- Make the transaction accounting model exact: overflow bounds, fee collection source, refund math, skipped/reverted batch charging, and whether unused reserved gas is reclaimable at block-accounting level. *Overflow bounds fixed (`gas_limit <= 2^63-1`, running sum checked). Fee collection source clarified (`TXPARAM(0x06)` from `resolved_target`). But skipped/reverted batch gas charging and journal restoration remain unspecified.*
3. **NOT FIXED** -- Either normatively specify the canonical paymaster here or split it into a required companion specification with exact code-hash and state semantics. *New text acknowledges it is secp256k1-only and may change, but the paymaster is still not normatively specified.*
4. **MOSTLY FIXED** -- Tighten default-code cryptography: low-`s` checks for secp256k1 and P256, explicit zero-address rejection, explicit P256 point validation expectations, and better replay/domain separation. *Low-s check added. Zero-address rejection added. P256 domain separator added (`P256_ADDRESS_DOMAIN`). P256 point validation noted ("P256VERIFY must reject invalid public keys"). Cross-scheme replay domain separator in the signature hash itself is still absent.*
5. **FIXED** -- Resolve `mode` / scope / atomic-flag numbering and decide whether one packed field is still worth the long-term complexity. *Separated into `mode` (uint8) + `flags` (uint8) in the frame tuple. Explicit 0-indexed bit layout with named constants.*
6. **FIXED** -- Rewrite `compute_sig_hash` as a pure function and address `TXPARAM(0x08)` caching / repricing. *Now uses `deep_copy(tx)`. Spec mandates: "This value MUST be computed at most once per transaction and cached."*
7. **PARTIALLY FIXED** -- Decide on realistic limits for `MAX_VERIFY_GAS` and `MAX_FRAMES`, and align them with the stated PQ and batching ambitions. *`MAX_FRAMES` reduced from 1000 to 64. Per-frame intrinsic cost added (475 gas). But `MAX_VERIFY_GAS` (100,000) is unchanged and the PQ tension is unresolved.*
8. **NOT FIXED** -- Clarify nonce timing, sender pre-validation, and whether 2D nonce semantics should be defined now rather than deferred. *Nonce is still incremented inside APPROVE. Zero-cost validation grinding remains. 2D nonces remain a "possible future extension."*

### Should Fix For Implementability And Ergonomics

1. **NOT FIXED** -- Specify default SENDER-mode RLP decoding, gas allocation, malformed-input behavior, and subcall rules. *Default code SENDER mode still reverts the entire frame on any sub-call revert. No per-subcall gas allocation or malformed-input behavior specified.*
2. **NOT FIXED** -- Add or clearly reject a cross-frame return-data / result channel, and define frame-level revert-data and status semantics. *No FRAMERETURNDATA opcode. APPROVE return data handling unspecified. Skipped frame receipt semantics undefined.*
3. **PARTIALLY FIXED** -- Add a per-frame intrinsic charge and revisit optional gas sharing / pooling / forwarding. *Per-frame intrinsic charge added (475 gas). Gas sharing/pooling/forwarding not addressed; gas remains rigidly per-frame.*
4. **FIXED** -- Clarify or redesign `TXPARAM` naming, gaps, exceptional-halt behavior, and future-extensibility story. *Unified on `TXPARAM`. Frame-level params moved to new `FRAMEPARAM` opcode. Numbering gaps eliminated (TX params 0x00-0x0A contiguous, frame params 0x00-0x07 contiguous).*
5. **FIXED** -- Make EIP-7702 interaction explicit. *Frame txs do not include 7702 authorization lists. Default code only for accounts without code or 7702 delegation. 7702 delegated accounts use delegated-code semantics.*
6. **NOT FIXED** -- Revisit whether a frame-level `value` field belongs in the model. *Rationale still says "not required because the account code can send value."*
7. **PARTIALLY FIXED** -- Restrict or more clearly justify VERIFY-frame access to later-frame data. *New security section documents that paymasters can observe SENDER frame data. But the access itself is not restricted.*

### Must Be Documented Explicitly Even If The Core Design Stays The Same

1. **NOT FIXED** -- `ORIGIN` behavior changes, with real compatibility examples. *No new compatibility section or worked examples added.*
2. **NOT FIXED** -- Transient-storage limitations and reentrancy-guard implications. *No rationale or documentation for transient storage reset behavior.*
3. **FIXED** -- Deploy-frame front-running and sender-address occupancy risks. *New "Deploy Frame Front-Running" security section added.*
4. **PARTIALLY FIXED** -- Canonical-paymaster trust assumptions, centralization, and reservation inconsistency. *Secp256k1-only acknowledged. But centralization, shared-bug risk, node-local reservation overcommit, and upgrade-path concerns not documented.*
5. **FIXED** -- Explicit-sender state-read implications. *New "Explicit Sender State-Read Amplification" security section added.*
6. **NOT FIXED** -- Gas-isolation UX costs and paymaster refund economics. *Not documented.*
7. **NOT FIXED** -- The relationship between one default-code `SENDER` multicall frame and multiple `SENDER` frames: what `VERIFY` can distinguish, which semantics differ, and when wallets should prefer each encoding. *Not documented.*
8. **NOT FIXED** -- Wallet and indexer guidance for rendering frames legibly. *Not documented.*

## Appendix A. Audit Finding Index

This appendix preserves the audit's full finding inventory so that the detailed issue set is not lost in the higher-level synthesis.

### Critical And High Findings

- `[C-1]` **FIXED** -- `APPROVE(0x0)` is invalid, yet the mempool rules require it for `only_verify`, and pair `pay` with `APPROVE(0x1)`; the audit recommends remapping to `only_verify -> 0x1` and `pay -> 0x2`. *Scopes remapped to named constants; all rules, examples, and bytecode regenerated.*
- `[H-1]` **NOT FIXED** -- `MAX_VERIFY_GAS = 100,000` is too small for most PQ / ZK verification paths and too large for free mempool validation; either raise it with stronger anti-DoS measures or lower it and be honest that PQ needs future precompiles.
- `[H-2]` **FIXED** -- `sum(frame.gas_limit)` can overflow without explicit per-frame and aggregate bounds; the audit recommends bounding `frame.gas_limit` and the total sum. *`gas_limit <= 2^63-1` per frame and running sum checked.*
- `[H-3]` **PARTIALLY FIXED** -- `APPROVE` gas collection is underspecified: which account pays, whether collection is at max fee or effective price, and how refunding works all need exact wording. *Now specifies "collect the transaction's maximum cost (TXPARAM(0x06)) from resolved_target." Refund section updated. Skipped/reverted batch charging still unspecified.*
- `[H-4]` **FIXED** -- The default secp256k1 path is malleable because it omits low-`s` enforcement; the audit recommends canonical low-`s` checks and strict `v`. *Low-s check added with `SECP256K1N_DIV_2` constant.*
- `[H-5]` **FIXED** -- `ecrecover` can return `address(0)` and the default code does not reject it; the audit recommends explicit zero-address rejection and a static ban on `tx.sender == address(0)`. *Both checks added.*
- `[H-6]` **NOT FIXED** -- The nonce is incremented inside `APPROVE`, enabling repeated failed validation attempts that consume no nonce and preserving a single sequential-nonce bottleneck; the audit recommends earlier increment or explicit rate-limiting, plus 2D nonces.
- `[H-7]` **PARTIALLY FIXED** -- `TXPARAM(0x06)` max-cost computation can overflow and mislead paymasters; the audit recommends explicit bit-width bounds and invalidating overflow. *`gas_limit` bounded to uint63. But `max_fee_per_gas` and `max_fee_per_blob_gas` still have no explicit bit-width constraints.*
- `[H-8]` **FIXED (documented)** -- Deploy frames can be front-run so the intended address gets occupied first; the audit recommends binding the deploy frame's target address to `tx.sender` and checking that it is still undeployed. *New security section documents the risk and mitigations.*
- `[H-9]` **NOT FIXED** -- The canonical paymaster is effectively centralized by exact-code matching, bringing shared-bug, reservation, and withdrawal risks; the audit recommends balance caps, boundary rechecks, proxy bans, withdrawal hardening, and explicit documentation of the trust model.
- `[H-10]` **PARTIALLY FIXED** -- `ENTRY_POINT = address(0xaa)` collides numerically with opcode `0xaa` and is not normatively explained; the audit recommends separating the values and defining ENTRY_POINT behavior explicitly. *ENTRY_POINT behavior now documented with explicit paragraph. Collision documented as semantically insignificant. Values not separated.*
- `[H-11]` **FIXED (documented)** -- The explicit `sender` field lets peers trigger nonce/state reads on arbitrary accounts before legitimacy is known; the audit recommends lightweight pre-validation and sender existence checks. *New security section with mitigation guidance. `tx.sender != address(0)` added as static constraint.*
- `[H-12]` **NOT FIXED** -- Canonical paymaster reservations are node-local, so multiple nodes can simultaneously overcommit the same balance; the audit recommends a pending-tx cap and safety margins.

### Medium Findings

- `[M-1]` **NOT FIXED** -- `ORIGIN` becomes frame-relative, which can break contracts and heuristics that still assume transaction-origin semantics. *No new compatibility section or examples added.*
- `[M-2]` **FIXED (documented)** -- Eliding all `VERIFY` data creates a broad malleability surface for sponsor data, paymaster terms, and transaction identity. *New normative text: "Implementations MUST NOT treat VERIFY frame data as sender-authenticated." Verifiers MUST authenticate independently.*
- `[M-3]` **NOT FIXED** -- P256 signatures are malleable without low-`s` enforcement. *secp256k1 low-s added but P256 low-s not explicitly added in the default code.*
- `[M-4]` **NOT FIXED** -- Cross-chain replay protection is only implicit because the spec does not firmly require `tx.chain_id == CHAIN_ID`.
- `[M-5]` **NOT FIXED** -- The refund formula can underflow depending on how intrinsic and calldata costs are counted against frame-level gas totals.
- `[M-6]` **PARTIALLY FIXED** -- P256 address derivation appears to rely on key validation that is not normatively ordered in the text. *Notes now state "P256VERIFY must reject invalid public keys, including points that are not on the P256 curve." But this is in notes, not in the normative pseudocode flow.*
- `[M-7]` **NOT FIXED** -- `calldata_cost(rlp(tx.frames))` is not precise enough for consensus use.
- `[M-8]` **FIXED** -- Approval-scope bit numbering is inconsistent across prose and formulas. *Mode and flags separated into distinct fields with named constants and explicit 0-indexed bit positions.*
- `[M-9]` **FIXED (documented)** -- Paymasters can inspect later SENDER-frame data and use that visibility to make approval decisions based on extractable value. *New security section "Cross-Frame Data Visibility During Validation" warns users.*
- `[M-10]` **NOT FIXED** -- Warm state persists but transient storage resets, which breaks EIP-1153-style developer assumptions and some cross-frame reentrancy guards.
- `[M-11]` **NOT FIXED** -- Rigid gas isolation wastes block space and forces systematic over-allocation.
- `[M-12]` **NOT FIXED** -- Atomic batching specifies state rollback but not reverted-batch gas charging or approval-flag survival.
- `[M-13]` **NOT FIXED** -- The draft does not say whether warm/cold access-journal state is part of an atomic-batch snapshot.
- `[M-14]` **NOT FIXED** -- Default SENDER-mode RLP decoding and subcall behavior are underspecified enough to create client-divergence risk.
- `[M-15]` **NOT FIXED** -- `FRAMEDATACOPY` on VERIFY frames may still incur meaningful memory-expansion gas despite "no data" semantics.
- `[M-16]` **FIXED** -- Scope constraint `0` means "any scope allowed" at the mode level but behaves asymmetrically in default code. *Flags field now separate. `APPROVE_SCOPE_NONE` means no APPROVE scope is allowed. VERIFY frames must have non-zero scope in flags. Confused-deputy documented.*
- `[M-17]` **FIXED (documented)** -- Transaction-specified approval-scope bits create a confused-deputy risk by constraining what the callee contract may legally approve. *New text: "allowed_scope is caller-supplied policy input... Verification logic SHOULD authenticate any approval scope it relies on and MUST NOT treat allowed_scope as trusted."*
- `[M-18]` **NOT FIXED** -- The one-pending-tx rule for non-canonical paymasters is Sybil-amplifiable by deploying many tiny paymasters.
- `[M-19]` **NOT FIXED** -- Canonical-paymaster balance changes can trigger expensive revalidation cascades after block changes or reorgs.
- `[M-20]` **FIXED** -- EIP-7702 delegation behavior in frame transactions is ambiguous. *Frame txs don't include 7702 auth lists. Default code only for accounts without code or 7702 delegation. Delegated accounts use 7702 semantics.*
- `[M-21]` **FIXED** -- A DEFAULT frame with null target can call the sender from `ENTRY_POINT` with attacker-controlled calldata. *Explicit "resolved target" concept introduced and used consistently throughout.*
- `[M-22]` **NOT FIXED** -- Validation can read sender storage without ERC-7562-style staking protections, so mutable config state can invalidate transactions.
- `[M-23]` **NOT FIXED** -- The draft is ambiguous about how SLOAD restrictions propagate through helper-contract `CALL` and `DELEGATECALL` chains.
- `[M-24]` **NOT FIXED** -- Default SENDER-mode subcalls lack a precise gas-allocation and starvation model.
- `[M-25]` **FIXED** -- `MAX_FRAMES = 1000` plus no per-frame intrinsic charge enables block stuffing through frame overhead. *MAX_FRAMES reduced to 64. Per-frame cost of 475 gas added.*

### Low And Informational Findings

- `[L-1]` **NOT FIXED** -- No built-in deadline or expiration mechanism means signatures can remain valid indefinitely while the nonce is unconsumed.
- `[L-2]` **FIXED** -- P256 and secp256k1 address derivation lacks a curve domain separator. *`P256_ADDRESS_DOMAIN = b"\x01"` added for P256 derivation.*
- `[L-3]` **NOT FIXED** -- The signature-hash pseudocode may omit the typed-transaction prefix.
- `[L-4]` **NOT FIXED** -- Canonical RLP encoding is assumed but not made explicit.
- `[L-5]` **NOT FIXED** -- Scope-`0x1` authorization can be replayed forever because it does not consume the nonce. *Note: scope values have been remapped, but the replay concern for execution-only approval remains.*
- `[L-6]` **FIXED** -- `TXPARAM` skips IDs `0x0A-0x0F`, and the frame-count parameter contradicts the non-empty-frames rule. *Numbering gaps eliminated. TX params contiguous 0x00-0x0A. Frame params moved to FRAMEPARAM 0x00-0x07. "Can be zero" contradiction removed.*
- `[L-7]` **NOT FIXED** -- `FRAMEDATALOAD` zero-padding can hide out-of-bounds and VERIFY-mode mistakes.
- `[L-8]` **NOT FIXED** -- The stack ordering of `FRAMEDATALOAD` / `FRAMEDATACOPY` is not stated as clearly as `APPROVE`.
- `[L-9]` **FIXED** -- `TXPARAM(0x13)` hides upper mode bits and can mislead security-sensitive logic. *Mode is now a pure uint8 in its own field. FRAMEPARAM(0x02) returns the full mode, FRAMEPARAM(0x03) returns the full flags. No hidden bits.*
- `[L-10]` **NOT FIXED** -- Receipt `payer` derivation is implicit rather than explicit.
- `[L-11]` **FIXED** -- `TXPARAM(0x15)` punishes indexing mistakes with an exceptional halt. *TXPARAM no longer has frame-level params. FRAMEPARAM(0x05) handles status with the same exceptional halt for current/future frames, but this is now in a dedicated opcode with clearer semantics.*
- `[L-12]` **NOT FIXED** -- The draft does not forbid `tx.sender == ENTRY_POINT`.
- `[L-13]` **NOT FIXED** -- Double-`APPROVE` failure semantics are unspecified. *The "if already set, revert the frame" behavior exists for each scope, but the interaction between two separate APPROVE calls in the same frame (e.g. APPROVE(0x1) then APPROVE(0x2)) is not clarified.*
- `[L-14]` **NOT FIXED** -- `APPROVE(0x2)` can be invalidated by front-running balance changes. *Note: now `APPROVE(APPROVE_PAYMENT)` but the same concern applies.*
- `[I-1]` **NOT FIXED** -- Paymaster nonces do not increment automatically, so paymasters must defend themselves.
- `[I-2]` **NOT FIXED** -- `MAX_PENDING=1` is too restrictive for shared gas accounts.
- `[I-3]` **NOT FIXED** -- There is no ERC-1271-style equivalent surface for frame-transaction signature validation results.

### Audit Cross-Cutting Concern Groups

- Specification consistency crisis: `C-1`, `M-7`, `M-8`, and `M-14` together suggest two independent clients could disagree on core behavior.
- Gas-accounting attack surface: `H-2`, `H-3`, `H-7`, `M-5`, and `M-12` together create overflow, underflow, and ambiguous-charging paths.
- Mempool DoS amplification: `H-1`, `H-6`, `H-11`, `H-12`, `M-18`, and `M-19` combine into an expensive validation and revalidation surface.
- Signature malleability plus VERIFY elision: `H-4`, `M-2`, and `M-3` make transaction identity and sponsor data more malleable than many implementers will expect.
- Inconsistent frame isolation: `M-9`, `M-10`, `M-11`, and `M-13` collectively describe an isolation model that is powerful but not internally intuitive.
- Ambition versus constraint gap: `H-1`, `H-6`, and `M-11` are used by the audit to argue that the EIP promises PQ migration and "full AA" while still constraining itself around a sequential nonce and rigid gas budgets.

## Appendix B. Source Coverage

All requested source files are represented in this synthesis:

- `audits/eip-8141/AUDIT-REPORT.md`
- `review-aa.md`
- `review-af.md`
- `review-ba.md`
- `review-bk.md`
- `review-co.md`
- `review-cx.md`
- `review-df.md`
- `review-hk.md`
- `review-hp.md`
- `review-il.md`
- `review-kc.md`
- `review-pa.md`
- `review-rs.md`
- `review-st.md`
- `review-tim.md`
- `review-ts.md`
- `review-vb.md`
