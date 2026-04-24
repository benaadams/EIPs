# Review of `EIPS/eip-8141.md`

## Bottom line

I think EIP-8141 is directionally very strong. It is one of the cleanest attempts I have seen to make account abstraction a first-class transaction primitive instead of a large amount of convention layered on top of ordinary calls. The proposal has real ambition: arbitrary validation logic, native fee abstraction, a path away from secp256k1, atomic batching, and a credible migration path for EOAs.

The part I like most is the framing choice itself. A transaction as an ordered list of validation / execution / payment frames is a better protocol primitive than "one signature plus a lot of wallet-specific machinery." It makes the protocol more honest about what modern Ethereum transactions already are.

My main reservation is that the draft currently tries to do too much in one place: consensus transaction semantics, new opcodes, EOA default behavior, public mempool policy, and a canonical paymaster model. The idea is good, but the draft needs tightening. There are also a few internal contradictions that should be fixed before broader design debate, because they obscure what the actual intended semantics are.

## What I like

### 1. The primitive is right

The core abstraction is good. `VERIFY`, `SENDER`, and `DEFAULT` frames let the protocol model the real phases of modern wallet execution directly instead of pretending everything is just an ECDSA-signed EOA send. That is much cleaner than hiding validation and payment logic behind one giant entrypoint contract.

### 2. It gets the post-quantum / cryptographic agility story right

The motivation is compelling: the protocol should stop hard-coding one authentication scheme as the only native path. The proposal does not force one replacement. It creates a mechanism for many schemes. That is the right way to open the door for PQ migration.

### 3. The canonical signature hash is a very good idea

The rationale around a protocol-provided canonical signature hash is strong and practical (`EIPS/eip-8141.md:663-677`). Wallet developers should not have to reimplement fragile hashing conventions inside contracts. The decision to elide `VERIFY` frame data is also smart because it keeps room for future aggregation and sponsor-added payloads.

### 4. Atomic batching is simple and useful

The atomic-batch flag is an elegant addition (`EIPS/eip-8141.md:693-697`). It solves a real UX and safety problem without inventing a whole extra execution mode. Approval-plus-action and other multi-step wallet flows need this.

### 5. EOA support is pragmatic

The default-code path is a good political and product choice (`EIPS/eip-8141.md:703-707`). A pure "smart accounts only" design would be cleaner on paper but much weaker in deployment reality. This draft at least gives existing EOA users a migration path into gas abstraction and better auth schemes.

### 6. The mempool section is trying to solve the correct problem

The draft takes mempool safety seriously. That matters. A protocol AA design that ignores shared-state invalidation and public relay DoS is incomplete. Even if I think parts of this section should be separated or softened, the draft is pointing at the right operational constraints.

## What I do not like

### 1. Too much is bundled into one EIP

The proposal mixes:

- consensus transaction semantics
- execution semantics for new opcodes
- EOA emulation / default code
- public mempool policy
- a canonical paymaster framework

Each of those is individually substantial. Bundling them gives a complete story, but it also makes the review surface very large and makes it harder to tell which parts are essential versus provisional. I would strongly consider splitting the document into:

- a core consensus / execution EIP
- a public mempool profile / relay policy EIP
- a canonical paymaster EIP, if that concept remains

That would increase optionality because the core transaction type could stabilize even if the public relay rules evolve.

### 2. The public mempool model feels more rigid than the transaction model

The frame abstraction is general; the mempool section narrows that back down to four recognized validation prefixes and a very specific paymaster story. That may be necessary for safe propagation, but it also risks standardizing today's relay assumptions too early.

I like the idea of a conservative public profile. I do not like binding too much of that policy to the same spec as the transaction format, because relay policy is the part most likely to change as wallets, paymasters, and validators learn what actually works in practice.

### 3. Per-frame gas budgeting is expressive but rigid

Unused gas from one frame cannot flow to later frames (`EIPS/eip-8141.md:439-449`). This makes reasoning predictable, but it also makes transaction authoring brittle. Many real wallet flows have uncertainty concentrated in one late step. A design that supports either:

- per-frame hard caps, or
- an optional shared gas envelope with per-frame maxima

would be more flexible.

### 4. `ORIGIN` semantics become much less intuitive

Having `ORIGIN` return the frame caller throughout all call depths (`EIPS/eip-8141.md:286`, `EIPS/eip-8141.md:858-860`) is understandable from the frame model, but it is still a very sharp semantic change. It is more dynamic than the traditional transaction-scoped meaning of `ORIGIN`. Even though contracts should not rely on `tx.origin`, many still do. This deserves more emphasis and probably more concrete examples of surprising behavior.

## Concrete issues in the current draft

### 1. The `APPROVE` scope semantics are internally contradictory in the mempool section

The core spec defines:

- `0x1` = execution approval (`EIPS/eip-8141.md:163-164`)
- `0x2` = payment approval (`EIPS/eip-8141.md:165`)
- `0x3` = both (`EIPS/eip-8141.md:166`)

But the public mempool structural rules later say:

- `self_verify` must call `APPROVE(0x2)` (`EIPS/eip-8141.md:545`)
- `only_verify` must call `APPROVE(0x0)` (`EIPS/eip-8141.md:546`)
- `pay` must call `APPROVE(0x1)` (`EIPS/eip-8141.md:547`)

That is not compatible with the earlier definition, and `0x0` is explicitly invalid (`EIPS/eip-8141.md:168`). This is the most obvious spec bug in the draft.

### 2. Null-target semantics are not fully normalized

The execution rules say that if `target` is null, the call target becomes `tx.sender` (`EIPS/eip-8141.md:278-285`). But several later checks still compare against `frame.target` directly:

- `APPROVE` requires `ADDRESS == frame.target` (`EIPS/eip-8141.md:182`)
- default EOA verification checks `frame.target != tx.sender` and reverts (`EIPS/eip-8141.md:326`, `EIPS/eip-8141.md:340`)

The examples repeatedly use null-as-sender targets. The spec needs an explicit notion of "resolved target" and then use that term consistently. As written, many example flows appear to revert.

### 3. `TXPARAM` / `TXPARAMLOAD` naming is inconsistent, and one parameter reference is wrong

The opcode is introduced as `TXPARAM` (`EIPS/eip-8141.md:196`), but the default-code section repeatedly refers to `TXPARAMLOAD` (`EIPS/eip-8141.md:324`, `EIPS/eip-8141.md:362`, `EIPS/eip-8141.md:369`, `EIPS/eip-8141.md:665`).

Worse, the Python comment says:

- `TXPARAMLOAD(0x14, TXPARAMLOAD(0x10))` for `mode` (`EIPS/eip-8141.md:362`)

But `0x14` is `len(data)` and `0x13` is `mode` (`EIPS/eip-8141.md:217-221`).

This is partly editorial, but it matters because these are exactly the kinds of details implementers copy.

### 4. Receipt semantics for skipped frames are underspecified

Receipts include one `frame_receipt` per frame with `[status, gas_used, logs]` (`EIPS/eip-8141.md:122-129`), and the spec also allows later frames to be skipped because of atomic batching (`EIPS/eip-8141.md:287`, `EIPS/eip-8141.md:298-300`). But it never says what receipt values a skipped frame should have.

Questions that need an answer:

- Does skipped mean `status = 0`, or a third status?
- Is `gas_used` zero for skipped frames?
- Are skipped frames visible to `TXPARAM(..., frameIndex)` status lookups as failure, or something else?

Right now "success" and "failure" are defined, but "not executed" is not.

### 5. Dependency tracking / revalidation is narrower than the allowed validation surface

The policy summary explicitly allows validation to depend on the code of helper contracts reached during validation (`EIPS/eip-8141.md:474-477`). But the acceptance algorithm records only sender storage slots (`EIPS/eip-8141.md:652`), and the revalidation section mentions sender state plus canonical paymaster changes (`EIPS/eip-8141.md:659`).

If helper contract code is part of the allowed dependency set, code-hash changes to those contracts should also be tracked for revalidation. Otherwise the revalidation story is incomplete.

### 6. The spec should be clearer about which parts are consensus and which are policy

The draft often shifts between "transaction is invalid" and "must not be propagated through the public mempool." Those are very different classes of rule. The distinction mostly exists, but not always crisply enough. This is particularly important in the validation prefix and paymaster sections, where policy language is dense and implementers may accidentally read it as consensus behavior.

## Usage scenarios I like

### 1. Passkeys / WebAuthn / hardware-backed wallets without bespoke protocol work

The obvious win is moving beyond secp256k1. P256 is already in the default-code path, and the general model lets wallets adopt future schemes without waiting for a brand new transaction type.

### 2. Solver and sponsor composition

The `VERIFY`-data elision is especially good for flows where the user signs the core intent and a sponsor or solver appends its own authorization later. That is a meaningful capability increase over designs where every participant's data must be fixed before the user signs.

### 3. Gas accounts and treasury accounts

The non-canonical paymaster discussion highlights a useful pattern (`EIPS/eip-8141.md:709-713`): a user or organization can centralize ETH in one gas-paying account and let many operating accounts stay ETH-light. That is a good fit for consumer wallets, gaming, enterprise ops, and family / team wallet setups.

### 4. Safer multi-step wallet actions

Atomic `SENDER` batches enable wallet-native workflows like:

- approve + swap
- wrap + bridge + stake
- revoke old allowance + set new allowance + execute downstream action

without forcing every wallet to invent custom batching contracts.

### 5. First-class account deployment flows

The `deploy -> verify -> user_op` path is important. It lets an account's first action be its real action, not just "deploy yourself first and come back later."

## Additional possibilities the draft enables, but does not emphasize enough

### 1. Signature aggregation as an eventual optimization path

The rationale mentions aggregation briefly (`EIPS/eip-8141.md:669-673`), but I think this is bigger than the draft presents. By making verification frames opaque to later frames and excluded from the canonical signed payload, the design preserves room for future aggregated validation schemes at the protocol or block-builder layer.

### 2. Policy-based smart accounts

Frames are a good substrate for policy engines:

- spending limits
- target allowlists / denylists
- time-delayed approvals
- guardian co-signing
- hardware key + recovery key combinations

A smart account can express these policies as validation logic without changing the transaction primitive.

### 3. Compliance / attestation-style authorization

The abstraction is broad enough for verification frames that check more than signatures:

- attestation from a trusted enclave
- proof of device possession
- application-issued capability tokens
- jurisdiction / KYC gating, if someone wants that

I am not endorsing all of those use cases, but the important point is that the transaction type enables them.

### 4. More credible EOA migration

Because EOAs can participate through default code, wallets can move users incrementally:

- start with ordinary secp256k1 EOA auth
- add sponsorship
- upgrade to passkeys or smart-account code later

That is a much better migration story than forcing a single cutover event.

## Changes I would consider to increase optionality further

### 1. Split the core transaction format from the public relay profile

This is my strongest recommendation. Keep the frame transaction primitive general. Put the recognized validation prefixes, canonical paymaster rules, and public mempool constraints into a companion spec or clearly labeled relay profile. That gives the ecosystem more room to evolve relay rules without reopening the consensus transaction format.

### 2. Define a resolved-target abstraction explicitly

The spec should say, once and clearly, whether `null` is normalized to `tx.sender`:

- before execution
- for `ADDRESS == frame.target` checks
- for signature hashing
- for frame introspection
- for receipt reporting

That would both fix current ambiguities and make future extensions easier.

### 3. Add an optional shared-gas mode or "use remaining gas" sentinel

Many useful wallet flows cannot know in advance which one frame will consume the slack. An optional transaction-wide gas envelope, while preserving per-frame caps where desired, would make the design more ergonomic and future-proof.

### 4. Preserve some committed visibility into `VERIFY` payloads

I agree with eliding `VERIFY` frame data from direct introspection and from the canonical signature hash. But there is room for an intermediate option, such as exposing:

- a commitment hash of the hidden payload, or
- a second data field where one part is committed / introspectable and one part is opaque

That would let users bind to sponsor terms or other verifier-side parameters more precisely without giving up the aggregation-friendly design.

### 5. Consider whether one execution principal is enough long term

Right now the model has one `sender` and a payer. That is enough for a lot of cases. But there may eventually be demand for richer "approved principal" semantics, where later frames can execute as one of several explicitly authorized identities, not only `tx.sender`. I would at least keep that extension path in mind when reserving mode bits and thinking about future opcode surface.

## Final view

I like this EIP a lot more than I dislike it. The core idea feels correct. It is more principled than ad hoc AA schemes, more future-proof on cryptography, and more honest about real wallet behavior.

The current draft is not ready as-is, though. It has a few plain spec errors, some under-specified edge cases, and too much policy bundled into the main document. If those are cleaned up, I think this could become one of the more important AA-related core proposals.
