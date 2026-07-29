**To:** AI Governance Committee; Legal (AGC)
**Cc:** CISO; Incident Response Lead
**From:** Corporate Security / Incident Response
**Date:** 2026-07-29
**Subject:** For review — proposed dormant "break-glass" AI capability for defensive incident response

---

**BLUF:** We are requesting review and pre-approval (not immediate deployment) of a
**normally-dormant, self-hosted open-weight LLM** to be activated **only during a declared
security incident** for defensive forensic and triage analysis. It fills a specific gap:
commercial AI providers demonstrably **over-refuse legitimate defensive security work**
(published refusal rates ~34% for malware analysis, ~44% for system hardening), and submitting
live incident data — attacker commands, C2 artifacts, exploit payloads, PCAPs — to a
third-party API creates confidentiality and data-egress exposure we cannot accept mid-breach.
We want this vetted and pre-authorized *now* so it is legally clear and technically ready
*before* we need it, not improvised during a crisis.

**Why now.** The July 2026 AI-leveraged intrusion at Hugging Face underscored that adversaries
(and misaligned agents) can operate at machine speed. Responding to that class of threat means
reasoning quickly over large volumes of hostile artifacts — exactly the material commercial
safety filters are tuned to refuse. We would rather not discover the gap for the first time
during an active APT.

**What we're proposing.** A single dedicated GPU host running a vetted open-weight model, kept
**dormant** and activated under strict controls. Our current recommendation is a
permissively-licensed **Qwen3 (Apache 2.0)** general reasoner paired with the US-origin,
security-purpose-built **Cisco Foundation-Sec** model, with **gpt-oss (OpenAI, Apache 2.0,
US-origin)** as a provenance-first runner-up. Model selection will be finalized on the basis of
measured refusal/accuracy against an internal defensive-task test set, not reputation.

**How it stays safe — controls live at the organizational layer, not the model.** We assume the
model has no reliable guardrails and design around that:

- **Air-gapped / tightly segmented** enclave; **no outbound internet** by default (data can go
  in, nothing comes out).
- **Dedicated, hardened host**; encrypted weights at rest; integrity-verified checkpoints.
- **Break-glass mechanics:** dormant by default; activation only on a **declared Sev-1/Sev-2
  incident** with a genuine need commercial tooling can't meet; **two-person authorization**
  by named individuals (IR Lead / CISO or delegate); **time-boxed (e.g., 72h) with automatic
  re-lock**.
- **Mandatory human-in-the-loop** — the model **advises only**; it has no code/tool execution
  and takes no action on live systems. Humans make every decision.
- **Full, tamper-evident logging** of every prompt and output (append-only, hash-chained,
  replicated off-enclave); **MFA + least-privilege** named access.
- **Sensitive incident data** never leaves the enclave, is never used to train the model, and
  is purged at decommission per retention policy.
- **Decommission + mandatory post-incident review** after every activation; periodic dormant-
  readiness drills.

**Key risks we're not hiding.** Residual risks remain and are addressed in the attached summary:
dual-use/misuse potential (mitigated by logging, two-person control, and review), model
error/hallucination in forensics (mitigated by mandatory human corroboration — output is a lead,
never a conclusion), provenance/supply-chain risk for foreign-origin weights (mitigated by
air-gap, checkpoint verification, and a US/EU-origin fallback path), and insider risk (mitigated
by attributable access and least privilege). We treat foreign-model provenance as standard
third-party risk requiring documented acceptance — not an automatic disqualifier, and not a
free pass.

**Framing.** This is intended strictly as a **controlled, auditable, break-glass fallback for
defensive use** — forensic analysis, malware/artifact triage, log analysis, and IR support
during an active incident. It is not an offensive capability and not routine tooling; the hard
activation criteria and time-boxing are designed to keep it that way.

**Call to action.** Please review the attached summary and tell us:

1. Does this design meet our organizational risk requirements as scoped?
2. If **not**, what specific amendments would get it there — e.g., restricting to US/EU-origin
   models only, prohibiting client-PII submission, tightening the activation authority or
   time-box, or a fuller physical air-gap? (These levers are enumerated in §6 of the summary.)
3. Are there licensing, regulatory-notification, or client-contractual obligations Legal needs
   us to build into the policy before any pilot?

We're happy to walk through the details live and to run a controlled proof-of-concept once the
guardrails you require are agreed. Nothing gets built until this group signs off on the
controls.

**Attachment:** *Break-Glass AI Capability for Incident Response — Governance Summary*
(`break-glass-ir-ai-summary.md`)

Thank you,
Corporate Security / Incident Response
