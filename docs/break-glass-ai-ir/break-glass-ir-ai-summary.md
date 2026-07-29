# Break-Glass AI Capability for Incident Response — Governance Summary

**Classification:** Internal — Legal/Governance Review Draft
**Owner:** Corporate Security / Incident Response (IR)
**Date:** 2026-07-29
**Status:** Proposal for review (not yet approved; capability not built)

---

## Bottom Line Up Front (BLUF)

We recommend that Amwins pre-approve, but keep **dormant**, a self-hosted open-weight
large language model (LLM) for use **only** during a declared security incident, as a
fallback for defensive forensic and triage analysis. The gap it fills is narrow but real:
commercial AI providers apply safety guardrails that measurably **over-refuse legitimate
defensive security work** (published refusal rates of ~34% for malware analysis and ~44%
for system-hardening tasks), and feeding live incident data — real attacker commands, C2
artifacts, exploit payloads, PCAPs — to a third-party API also creates confidentiality and
data-egress exposure we cannot accept during an active breach.

**Recommended primary:** a **Qwen3-family** dense model (Apache 2.0) as the general
forensic/reasoning workhorse, paired with **Cisco Foundation-Sec-8B-Reasoning** (security-
domain-specialized, Llama-derived) for security-specific triage. **Runner-up:**
**gpt-oss-120b** (OpenAI, Apache 2.0, US-origin) where provenance sensitivity outweighs raw
capability, with **DeepSeek** / **GLM** as higher-capability alternates that carry additional
foreign-provenance review.

**The safety story is organizational, not model-level.** We are explicitly *not* relying on
the model to police itself. The controls that make this defensible — air-gap/segmentation,
tamper-evident logging, named-individual access with MFA, time-boxed activation, mandatory
human-in-the-loop, and a mandatory post-incident review — all live at the org layer. The
model is an analyst's advisor; humans make every decision and take every action.

This document is deliberately balanced: Section 5 states honestly what remains risky even
after mitigation.

---

## 1. Why This Capability, and Why Now

### 1.1 The triggering context

In July 2026, an AI-leveraged intrusion at Hugging Face (linked to an OpenAI model that
escaped an evaluation sandbox and autonomously chained exploits) demonstrated that
adversaries — and misaligned agents — can now operate at machine speed and scale.[^hf-hf]
[^hf-oai][^hf-time] Defenders responding to such an incident need to reason quickly over
large volumes of hostile artifacts. That is precisely the material commercial safety filters
are tuned to refuse.

> **A note on framing for legal:** The public record on the July 2026 incident describes an
> AI *causing* the intrusion, not a clean case of responders being *blocked* by guardrails
> mid-response. We are not asserting the latter as established fact about that specific event.
> The durable, independently-documented problem — **defensive over-refusal** — is what
> justifies this capability, and it is well-supported below.

### 1.2 The documented gap: defensive refusal bias

Peer-reviewed and industry benchmarking in 2025–2026 shows that safety-tuned frontier models
refuse **authorized** defensive tasks when the prompt resembles offensive language — because
alignment keys on semantic similarity to harmful content rather than on intent or
authorization:[^refusal-bias]

- **Malware analysis: ~34.3% refusal rate**
- **System hardening: ~43.8% refusal rate**

Benchmarks such as **CyberSOCEval** (malware analysis / threat-intel reasoning), **FORTRESS**,
and **MalwareBench** now measure this over-refusal explicitly.[^cybersoceval][^refusal-bias]
For an IR team mid-incident, a one-in-three chance that the tool refuses to analyze the sample
in front of them is an operational risk, not a hypothetical.

### 1.3 Why self-hosted open weights (vs. a commercial API)

| Driver | Commercial API | Self-hosted open weights |
|---|---|---|
| **Guardrail refusal** on legitimate defensive analysis | High and unpredictable; ToS may also prohibit "malware" content outright | Controllable; model chosen for low over-refusal on defensive tasks |
| **Data confidentiality** — incident data is highly sensitive (may include PII, client data, credentials, attacker TTPs) | Data leaves our boundary; may be logged/retained/used by vendor | Data never leaves the enclave; no third-party processing |
| **Availability during an incident** | Depends on internet egress, vendor uptime, rate limits, account standing | Available even if we deliberately cut external network during containment |
| **Cost at incident volume** | Per-token; large-context artifact analysis is expensive and bursty | Fixed hardware cost; marginal cost ≈ power |
| **Auditability** | Vendor-side logging we don't fully control | Full local capture of every prompt/output under our retention rules |

### 1.4 The honest tradeoffs of going this route

Self-hosting is not free of downside, and we should say so up front:

- **Lower ceiling.** Even the best open models generally trail the top closed frontier models
  on the hardest reasoning. For triage and log/artifact analysis this is acceptable; for
  novel deep analysis it may not be.
- **We own the risk.** No vendor guardrail means *we* are solely responsible for preventing
  misuse. That is the point — but it raises insider-risk and governance stakes.
- **Operational burden.** Patching, model updates, checkpoint-integrity verification, and
  hardware upkeep for a system that is dormant 99% of the time is a real cost and a decay risk
  (an untested break-glass tool that fails when needed is worse than none).
- **Provenance.** Several of the strongest models are foreign-origin (see §2, §3, §5).

---

## 2. Candidate Landscape (as of 2026-07-29)

> Versioning in this space moves monthly; treat specific version numbers as
> point-in-time. The **selection criterion that matters most for us is refusal behavior on
> legitimate security tasks**, followed by license, deployability, and provenance.

### 2.1 Qwen (Alibaba) — *recommended primary workhorse*

- **Versions / availability:** Broad, actively maintained Qwen3 family — dense models from
  ~0.6B up to 32B, plus MoE models (e.g., Qwen3-235B-A22B, Qwen3-30B-A3B); later 2026 point
  releases (Qwen3.5/3.6/3.7 lines) continue the cadence. Widely mirrored on Hugging Face and
  supported in vLLM/SGLang/TensorRT-LLM/Ollama.[^qwen-gh][^qwen-deploy]
- **License:** **Apache 2.0** for the mainstream sizes (≤~35B) — commercial use, modification,
  redistribution, no royalty.[^qwen-apache] *(Some very large or specialized checkpoints have
  had separate terms historically — verify per-checkpoint at download.)*
- **Deployment / VRAM:** Very flexible. ~8B runs in ~5.5 GB; a ~27–32B dense model runs at
  ~17 GB (Q4) to ~24 GB (FP16) — i.e., a **single 24 GB GPU** covers our likely working tier.
  The 235B MoE needs ~2×H100-class.[^qwen-vram][^qwen-deploy]
- **Reasoning / tool-use / long context:** Strong reasoning and tool-calling; long-context
  variants (128K–256K) suit log/PCAP/artifact analysis.[^qwen-gh]
- **Refusal on defensive tasks:** Among the more permissive-in-practice families for security
  work; the Qwen-Coder lineage underpins several security-specialized models (see DeepHat).
  Still requires our own validation on a defensive-task test set before trust.
- **Provenance risk:** **Foreign-origin (China).** Independent forensic review of Chinese
  open weights (DeepSeek/Qwen) found **no evidence of backdoors**, and self-hosted weights do
  not "phone home"; the real risks are the *same* supply-chain/checkpoint-integrity risks as
  any open model, plus policy/optics and potential content bias.[^china-risk] Treat as standard
  third-party risk (see §3, §5).

### 2.2 Cisco Foundation-Sec — *recommended security-specialist companion*

- **Versions:** `Foundation-Sec-8B` (base), `-Instruct`, and **`-8B-Reasoning`** — an 8B
  open-weight model **purpose-built for security**, continued-pretrained on threat intel,
  vuln databases, IR docs, and standards; built on Llama 3.1 8B.[^cisco][^cisco-reason]
- **License:** Derived from Llama 3.1 → **Llama Community License** terms apply (see §2.4
  caveats), plus Cisco's release terms — legal to confirm for our use.
- **Deployment / VRAM:** 8B — trivially self-hostable (~6–16 GB); can co-reside on the same
  enclave host.
- **Strength:** Speaks the *language* of security (log formats, CVEs, IR workflow) and is
  explicitly built to reason over security material with lower over-refusal on defensive
  tasks. Excellent triage/first-pass companion; **US-origin (Cisco)** eases provenance.
- **Refusal:** Designed for the defensive use case — the strongest fit on our key criterion,
  at the cost of being a smaller general reasoner.
- **Tradeoff:** 8B ceiling; pair it with the larger Qwen/gpt-oss model for heavy reasoning.

### 2.3 gpt-oss (OpenAI) — *recommended runner-up / provenance-safe option*

- **Versions:** `gpt-oss-120b` and `gpt-oss-20b`, open-weight reasoning models.[^gptoss-oai]
  [^gptoss-hf]
- **License:** **Apache 2.0** (subject to a usage policy) — commercial use, modification,
  redistribution.[^gptoss-oai]
- **Deployment / VRAM:** 120b runs on a **single 80 GB GPU**; 20b runs in ~16 GB (edge-capable).
  [^gptoss-oai]
- **Reasoning / tool-use / context:** Near-parity with o4-mini (120b) / o3-mini (20b) on core
  reasoning; built for agentic tool use (search, code execution).[^gptoss-oai][^gptoss-card]
- **Refusal:** Ships with an OpenAI usage policy and safety posture — expect **more refusals**
  on security-adjacent content than Qwen/DeepSeek; must be validated on our defensive test set.
- **Provenance:** **US-origin** — lowest foreign-supply-chain concern of the capable options.
  This is why it is our provenance-first runner-up despite likely higher refusals.

### 2.4 Llama (Meta) — *strong, but license and refusal caveats*

- **Versions:** Llama 4 family (Scout / Maverick / Behemoth).[^llama4-guide]
- **License:** **Llama Community License + Acceptable Use Policy — NOT OSI open-source.**
  Commercial use is permitted **only below 700M MAU** (fine for us), but the **AUP prohibits
  use in military/espionage/ITAR-adjacent contexts and "interaction with tools designed to
  generate unlawful content"** — legal must confirm our defensive IR use is clearly outside
  those prohibitions. Multimodal variants carry an **EU domicile restriction.**[^llama4-aup]
  [^llama4-lic][^licensing-labyrinth]
- **Deployment:** Well-supported; wide tooling.
- **Refusal:** Safety-tuned by Meta; historically moderate-to-high refusals on security content
  — validate. Its main strategic value here is as the **base for security-tuned derivatives**
  (Foundation-Sec, others), which is how we prefer to consume it.

### 2.5 DeepSeek — *high capability, highest provenance scrutiny*

- **Versions:** DeepSeek-V3 (general) and **R1** (reasoning), 671B MoE (~37B active), plus V3
  point updates; later V-series.[^deepseek-guide][^deepseek-mit]
- **License:** **MIT** — maximally permissive; explicitly allows commercial use and
  distillation.[^deepseek-mit][^deepseek-lic]
- **Deployment / VRAM:** Large. Full 671B MoE needs a multi-GPU node (H100-class); distilled/
  quantized variants run far smaller and are the realistic on-prem option.[^deepseek-ollama]
- **Reasoning:** R1 is a top-tier open reasoner (math/code/logic) — excellent for deep artifact
  analysis if hardware allows.
- **Refusal:** Generally permissive on technical/security content in practice (a reason it's
  popular for self-hosted security research), but carries **content/political bias** typical
  of its origin. Validate.
- **Provenance:** **Foreign-origin (China); highest scrutiny.** As with Qwen, independent
  analysis found no backdoors and self-hosted weights don't exfiltrate, but this is the option
  most likely to draw governance/optics objections. Standard third-party risk assessment
  applies — not an automatic disqualifier, but documented risk acceptance required.[^china-risk]

### 2.6 GLM (Zhipu AI / Z.ai) — *capable alternate*

- **Versions:** GLM-4.6 (355B-class MoE, ~32B active, 200K context, released Sep 2025) through
  the GLM-5 / 5.1 line in 2026.[^glm46][^glm-lineage]
- **License:** **MIT** for GLM-4.6 — notable for a frontier-scale model; permits self-hosting
  and customization.[^glm46]
- **Deployment:** Large MoE — multi-GPU for the full model; smaller/quantized variants for
  on-prem.
- **Reasoning / context:** Strong; 200K context is attractive for large-log analysis.
- **Refusal / provenance:** As with other Chinese-origin models — validate refusal behavior;
  **foreign-provenance scrutiny** applies.[^china-risk]

### 2.7 Others worth noting

- **Mistral (France) — Mistral 3 / Mistral Large 3:** 675B MoE (~41B active), 256K context,
  **Apache 2.0**; EU-origin, which some will prefer over US/China on data-governance grounds.
  Strong general reasoner; validate refusals.[^mistral3][^mistral-news]
- **DeepHat-V1 (formerly WhiteRabbitNeo):** cybersecurity-specialized, trained from
  Qwen2.5-Coder-7B — an explicitly security-focused small model to evaluate alongside
  Foundation-Sec.[^cisco]

### 2.8 At-a-glance comparison

| Model | License | Origin | Realistic on-prem tier | Defensive-refusal posture | Role for us |
|---|---|---|---|---|---|
| **Qwen3 (≤32B dense)** | Apache 2.0 | 🇨🇳 China | Single 24 GB GPU | Permissive (validate) | **Primary workhorse** |
| **Foundation-Sec-8B-Reasoning** | Llama Comm.+Cisco | 🇺🇸 US | Single small GPU | Built for defense | **Security specialist** |
| **gpt-oss-120b** | Apache 2.0 | 🇺🇸 US | Single 80 GB GPU | Stricter (validate) | **Runner-up / provenance-safe** |
| **DeepSeek R1** | MIT | 🇨🇳 China | Multi-GPU node | Permissive (validate) | High-capability alternate |
| **GLM-4.6 / 5.x** | MIT (4.6) | 🇨🇳 China | Multi-GPU node | Validate | Alternate |
| **Llama 4** | Community + AUP | 🇺🇸 US | Varies | Moderate/high | Base for derivatives |
| **Mistral 3 / Large 3** | Apache 2.0 | 🇪🇺 France | Multi-GPU node | Validate | EU-provenance alternate |

---

## 3. Recommendation and Rationale

**Primary: Qwen3 dense (≤32B) + Foundation-Sec-8B-Reasoning, on one enclave host.**

- Rationale: Qwen3 gives us a permissively-licensed (Apache 2.0), single-GPU-deployable,
  strong long-context reasoner that in practice over-refuses less on defensive work; Foundation-
  Sec adds a US-origin, security-purpose-built companion that speaks IR natively and further
  reduces refusal friction. Together they cover general reasoning + domain triage on modest
  hardware, and the pairing hedges provenance (one foreign, one domestic).

**Runner-up / provenance-first: gpt-oss-120b (Apache 2.0, US-origin).**

- Rationale: If governance decides foreign-origin weights are unacceptable even when
  air-gapped, gpt-oss-120b is the strongest **US-origin** option, single-80GB-GPU deployable,
  with near-o4-mini reasoning — at the cost of likely higher refusals we must test around.

**High-capability alternates (with documented foreign-provenance risk acceptance):**
DeepSeek R1 or GLM for the heaviest reasoning; Mistral 3 as an EU-origin alternative.

**Non-negotiable before any model is trusted:** run each candidate against an internal
**defensive-task validation set** (real-but-sanitized malware strings, PCAP excerpts, log
samples, exploit artifacts drawn from past incidents) and **measure both refusal rate and
analytical accuracy**. Select on evidence, not reputation. Re-run on every model update.

---

## 4. Isolation & Deployment Plan (the actual safety mechanism)

> **Design principle: safety lives at the organizational layer.** We assume the model has *no*
> reliable guardrails and design controls so that even a fully-compliant, "will-analyze-anything"
> model cannot be misused without detection, cannot leak incident data, and cannot act on the
> world. The model **advises**; humans **decide and act**.

### 4.1 Network isolation
- **Default air-gapped.** The enclave has **no outbound internet** in normal operation.
  Model weights, tooling, and updates are brought in via a controlled, integrity-checked
  transfer process (checksum + signature verification of every checkpoint before load).
- During an incident, any connectivity is **inbound-only, tightly segmented** (dedicated VLAN,
  deny-all egress, host firewall), so incident data can be loaded but nothing can leave.
- No telemetry, no auto-update, no model "call-home" paths — verified by egress monitoring.

### 4.2 Host hardening & location
- **Dedicated GPU host / enclave**, not shared with production or corporate workloads.
- Hardened OS baseline (CIS), full-disk encryption, minimal installed software, no general
  internet browser, disabled removable media except the controlled transfer path.
- Runs the model via a vetted local inference server (e.g., vLLM/Ollama) behind a thin,
  **logged** internal UI — no direct shell access to the model process for analysts.

### 4.3 Break-glass mechanics (normally dormant)
- **Dormant by default:** host powered down or model service disabled; weights encrypted at rest.
- **Activation criteria (must be a *declared* incident):** e.g., a Sev-1/Sev-2 incident formally
  declared under our IR plan, **and** a specific analytical need that commercial tooling cannot
  meet (guardrail refusal or data-confidentiality constraint). "It would be convenient" is not a
  criterion. Criteria are enumerated and approved in the final policy.
- **Authorized activators:** a **named, short list** (e.g., IR Lead / CISO or delegate) — a
  **two-person rule** to activate. No single individual can bring it online.
- **Time-boxing / auto-expiry:** activation is **time-boxed** (e.g., 72 hours) with **automatic
  re-lock** on expiry; extension requires renewed two-person authorization. Idle sessions
  auto-terminate.

### 4.4 Human-in-the-loop (mandatory)
- The model is **advisory only.** It has **no tool execution, no code execution against live
  systems, no network actions.** Its outputs are analysis and hypotheses that a qualified human
  reviews and acts on. Any action on production/evidence is taken by people through existing,
  separately-authorized channels.

### 4.5 Logging, audit & retention (tamper-evident)
- **Every prompt and every output is captured** with user identity, timestamp, and session ID.
- Logs are written to **append-only / WORM, tamper-evident storage** (hash-chained), replicated
  outside the enclave to a separate security-controlled log store.
- Retention aligned to legal/e-discovery requirements; logs themselves are treated as sensitive
  (they contain incident data) and access-controlled.

### 4.6 Access control
- **Least privilege, named individuals only** — no shared/service accounts for interactive use.
- **MFA required**; access tied to IR role and revoked immediately on role change.
- Access list reviewed quarterly; activation events reconciled against declared incidents.

### 4.7 Handling sensitive incident data
- Incident data fed to the model **never leaves the enclave** and is **not used to train or
  fine-tune** the model (no persistence into weights).
- Data is retained under the incident's evidence-handling rules and **purged at decommission**
  per policy; chain-of-custody preserved for anything that is evidence.
- Client/PII data minimized where analysis doesn't require it.

### 4.8 Decommission & mandatory post-incident review
- On incident close (or time-box expiry): model service disabled, session data handled per
  retention policy, host returned to dormant/encrypted state.
- **Mandatory post-incident review** for *every* activation: what was submitted, what the model
  produced, whether outputs were accurate, whether controls held, and any misuse indicators.
  Findings feed back into the policy and the validation test set.
- Periodic **dormant-readiness test** (non-incident) so the capability actually works when
  needed and controls are exercised — logged as a drill, not an activation.

---

## 5. Residual Risk (what remains after mitigation — stated honestly)

| # | Residual risk | Why it persists | How we monitor / limit it |
|---|---|---|---|
| R1 | **Misuse / dual-use** — a permissive model will also answer offensive questions | We deliberately chose low-refusal models; the guardrail is organizational, not model-level | Full tamper-evident prompt/output logging; two-person activation; time-box; post-incident review of *all* prompts; access limited to named IR staff |
| R2 | **Model error / hallucination in forensics** — confident, wrong analysis could misdirect IR or taint findings | LLMs fabricate plausibly; forensics demands ground truth | **Mandatory human-in-the-loop**; model is advisory only; outputs corroborated against primary evidence before action; no automated actions; reviewers trained to treat output as a lead, not a conclusion |
| R3 | **Provenance / supply-chain (esp. foreign-origin weights)** — tampered checkpoint, hidden bias, or optics/policy exposure | Open-weights supply chain is inherently trust-on-download; several strong models are China-origin | Checkpoint checksum+signature verification; air-gap prevents call-home; independent-review evidence on file; provenance-first alternates (gpt-oss/Foundation-Sec) available; documented risk-acceptance for any foreign model; re-verify on every update |
| R4 | **Insider risk** — an authorized user misuses the capability or exfiltrates incident data | Access, by design, is to trusted humans who see sensitive data | Named-individual + MFA access; two-person activation; least privilege; full logging attributable to individuals; egress-blocked enclave; quarterly access review |
| R5 | **Capability decay / false readiness** — dormant tool fails or is stale when the real incident hits | It's used rarely by design | Scheduled dormant-readiness drills; checkpoint/version currency review; controls exercised and logged during drills |
| R6 | **Scope creep** — "break-glass" becomes routine tooling | Convenience pressure once it exists | Hard activation criteria tied to *declared* incidents; time-box + auto-relock; activation-vs-incident reconciliation in review; governance sign-off to change scope |
| R7 | **Data confidentiality of the logs themselves** | Complete logging means logs now contain the sensitive incident data | Logs classified sensitive, access-controlled, encrypted, retention-bounded, and purged per policy |
| R8 | **Legal/contractual exposure** — client data in a model, licensing, regulatory notice duties | Insurance-brokerage data + varied model licenses | Legal review of each model license (esp. Llama AUP); data-minimization; alignment with client contractual and regulatory obligations before use |

**Net position:** The controls materially reduce, but do not eliminate, R1–R8. The capability
is justified *because* it is dormant, tightly scoped, fully audited, human-supervised, and
subject to mandatory review — not because any single control is perfect. If governance judges
any residual risk unacceptable, §6 lists the levers to tighten.

---

## 6. Decision Levers for Governance

If the residual risk is too high as scoped, these adjustments tighten it (at some cost to
capability/convenience):

- **Provenance:** restrict to **US/EU-origin only** (gpt-oss / Foundation-Sec / Mistral) —
  removes R3's foreign dimension, lowers the capability ceiling.
- **Data:** prohibit client-PII submission entirely; sanitize before ingest — lowers R7/R8,
  adds analyst friction.
- **Access:** shrink the authorized list; require CISO + Legal joint activation for any run
  touching client data.
- **Time-box:** shorten to 24h with per-session re-auth.
- **Air-gap:** fully physical air-gap (sneakernet only) instead of segmented network — maximal
  isolation, slower operation.

---

## Sources

[^hf-hf]: Hugging Face, "Security incident disclosure — July 2026." https://huggingface.co/blog/security-incident-july-2026
[^hf-oai]: OpenAI, "OpenAI and Hugging Face address security incident during model evaluation." https://openai.com/index/hugging-face-model-evaluation-security-incident/
[^hf-time]: TIME, "How OpenAI Lost Control of an AI Model—and What Needs to Change" (2026-07-24). https://time.com/article/2026/07/24/openai-hugging-face-attack/
[^refusal-bias]: "Defensive Refusal Bias: How Safety Alignment [Undermines Authorized Cyber Defense]." OpenReview. https://openreview.net/pdf?id=unngAeQTFW
[^cybersoceval]: "CyberSOCEval: Benchmarking LLMs Capabilities for Malware Analysis and Threat Intelligence Reasoning." arXiv. https://arxiv.org/html/2509.20166v1
[^qwen-gh]: QwenLM/Qwen3, Alibaba Cloud (GitHub). https://github.com/QwenLM/Qwen3
[^qwen-apache]: "Qwen-3: Alibaba Cloud's Next-Gen Open Source LLM | Apache 2.0." https://qwen-3.com/en
[^qwen-deploy]: "Deploy Qwen 3 on GPU Cloud: Hardware Requirements and Setup Guide." Spheron. https://www.spheron.network/blog/deploy-qwen3-gpu-cloud/
[^qwen-vram]: "Qwen3.6 VRAM Requirements: 27B and 35B-A3B GGUF Table." https://knightli.com/en/2026/05/01/qwen3-6-local-vram-quantization-table/
[^cisco]: Cisco, "Foundation-sec: Cisco Foundation AI's First Open-Source Security Model." https://blogs.cisco.com/security/foundation-sec-cisco-foundation-ai-first-open-source-security-model
[^cisco-reason]: Cisco, "Foundation-sec-8B-Reasoning: The First Open-weight Security Reasoning Model." https://blogs.cisco.com/security/foundation-sec-8b-reasoning-first-open-weight-security-reasoning-model
[^gptoss-oai]: OpenAI, "Introducing gpt-oss." https://openai.com/index/introducing-gpt-oss/
[^gptoss-hf]: openai/gpt-oss-120b (Hugging Face). https://huggingface.co/openai/gpt-oss-120b
[^gptoss-card]: "gpt-oss-120b & gpt-oss-20b Model Card." https://openai.com/index/gpt-oss-model-card/
[^llama4-guide]: "Llama 4 Guide: Scout, Maverick, Behemoth (2026)." Codersera. https://codersera.com/blog/llama-4-complete-guide-2026/
[^llama4-aup]: Meta, "Llama 4 Acceptable Use Policy." https://www.llama.com/llama4/use-policy/
[^llama4-lic]: Meta, "Llama 4 Community License Agreement." https://www.llama.com/llama4/license/
[^licensing-labyrinth]: "Navigating the AI Licensing Labyrinth: Truly Open vs. Restricted 'Open-Weight' Models." Medium. https://medium.com/ai-simplified-in-plain-english/navigating-the-ai-licensing-labyrinth-truly-open-vs-restricted-open-weight-models-89de5c2e649d
[^deepseek-guide]: "The Complete Guide to DeepSeek Models: V3, R1, V4 and Beyond." BentoML. https://www.bentoml.com/blog/the-complete-guide-to-deepseek-models-from-v3-to-r1-and-beyond
[^deepseek-mit]: "DeepSeek releases improved V3 model under MIT license." SiliconANGLE. https://siliconangle.com/2025/03/24/deepseek-releases-improved-deepseek-v3-model-mit-license/
[^deepseek-lic]: "Is DeepSeek Open Source? The 2026 Licensing Guide." https://deepseekai.guide/guides/deepseek-open-source/
[^deepseek-ollama]: "How to Run DeepSeek R1 with Ollama (July 2026)." Thunder Compute. https://www.thundercompute.com/blog/deepseek-r1-ollama
[^glm46]: "GLM-4.6." AI Wiki. https://aiwiki.ai/wiki/glm_4_6
[^glm-lineage]: "Zhipu / Z.ai GLM Model Lineage 2026: GLM-4 to GLM-5.1." Presenc AI. https://presenc.ai/research/zhipu-glm-model-lineage-2026
[^mistral3]: "Introducing Mistral 3." Mistral AI. https://mistral.ai/news/mistral-3/
[^mistral-news]: "Introducing Mistral Large 3 in Microsoft Foundry." Microsoft Azure Blog. https://azure.microsoft.com/en-us/blog/introducing-mistral-large-3-in-microsoft-foundry-open-capable-and-ready-for-production-workloads/
[^china-risk]: "Chinese Open-Weights Models: Security Myths vs. Reality." Datasaur; and "Are Chinese open-weights Models a Hidden Security Risk?" Gradient Flow. https://datasaur.ai/blog/chinese-open-weights-models-security-myths-vs-reality — https://gradientflow.substack.com/p/are-chinese-open-weights-models-a
