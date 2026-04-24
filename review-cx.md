# Codex Adversarial Review — EIP-8141 (Frame Transaction)

**Target:** `EIPS/eip-8141.md` (branch diff against master)
**Verdict:** needs-attention

No-ship: the native AA/PQ direction is attractive, but the current `EIPS/eip-8141.md` draft has blocking contradictions in approval semantics, relies on an undefined canonical paymaster, and leaves block-gas accounting ambiguous.

---

## Findings

### [critical] Validation-prefix rules contradict the core `APPROVE` semantics
**Location:** EIPS/eip-8141.md:544-549

This section makes the public mempool depend on `self_verify -> APPROVE(0x2)`, `only_verify -> APPROVE(0x0)`, and `pay -> APPROVE(0x1)`. Earlier in the spec, `APPROVE` is defined as `0x1 = sender approval`, `0x2 = payment approval`, `0x3 = both`, and any other value is invalid. That means the mempool rules reverse sender/payer approval and even require an impossible `APPROVE(0x0)`. Clients that implement the mempool section will reject transactions that follow the opcode semantics and the later examples, so interoperability breaks immediately.

**Recommendation:** Rewrite the validation-prefix taxonomy and structural rules to match the normative `APPROVE` semantics, then regenerate all examples and paymaster rules from that single mapping.

---

### [high] Canonical-paymaster safety is delegated to an implementation the draft never actually specifies
**Location:** EIPS/eip-8141.md:611-625

Public mempool admission is allowed to bypass the generic validation rules when the `pay` frame runtime code 'exactly matches' a canonical paymaster, and solvency depends on `pending_withdrawal_amount(paymaster)`. But the draft never defines the canonical paymaster bytecode/code hash, its withdrawal state machine, or how a client computes that pending-withdrawal amount. This leaves mempool admission and revalidation as client-specific policy instead of a shared protocol rule.

**Recommendation:** Inline the canonical paymaster specification here or split it into a required companion EIP with a fixed runtime code hash, explicit storage/API semantics, and exact revalidation rules.

---

### [high] Gas accounting invents a block-gas refund model without defining consensus behavior
**Location:** EIPS/eip-8141.md:429-449

The draft charges `sum(frame.gas_limit)` up front, says unused frame gas cannot flow to later frames, then refunds the difference and 'adds it back to the block gas pool'. That is a new block-accounting model: builders and validators need to know whether later transactions may consume that re-added gas and how block validity is checked. Without a full consensus rule here, two implementations can pack and validate blocks differently around frame transactions.

**Recommendation:** Either make reserved frame gas permanently count against block gas, or specify a complete consensus-level block gas accounting model for reclaiming unused per-frame gas, including worked block-validity examples.

---

### [high] The default-code rules make third-party EOA paymasters impossible despite the draft claiming they are supported
**Location:** EIPS/eip-8141.md:326-340

In the default-code VERIFY path, an EOA reverts unless `frame.target == tx.sender`. Because pay frames are VERIFY-mode authorization frames, a third-party EOA paymaster cannot approve payment under these rules; only the sender's own EOA can use default-code verification. That directly conflicts with the later claim that 'any EOA' can be a paymaster and breaks the gas-account / sponsored-transaction optionality the draft advertises.

**Recommendation:** If third-party EOA paymasters are in scope, extend default-code verification to authorize non-sender paymasters explicitly; otherwise remove the claim and any examples that rely on it.

---

## Next Steps

- Normalize the sender/payer approval model first; the current draft, examples, and mempool section are not describing the same transaction semantics.
- Define the canonical paymaster as a real normative artifact, or drop the carve-out and keep all paymasters under one generic validation model.
- If increasing optionality is still a goal after the blockers are fixed, make third-party paymaster authorization a first-class interface instead of relying on sender-only default code.
