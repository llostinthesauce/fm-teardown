# Cross-Reference: Our Findings ↔ Apple's Official 3rd-Gen Post

This document reconciles the reverse-engineering findings in `00`–`02` against Apple's own writeup:

**Source:** [*Introducing the third generation of Apple Foundation Models*](https://machinelearning.apple.com/research/introducing-third-generation-of-apple-foundation-models) — Apple Machine Learning Research.

The two views are complementary. Apple's post names the **models** and headline **architecture**; it says little about the **runtime plumbing** (the catalog, adapters, CLI, endpoints, routing). Our binary analysis sees the plumbing but can't read weights. Together they line up remarkably well.

Evidence tags: **[OFFICIAL]** = stated by Apple; **[STRINGS]/[OBSERVED]** = our findings; **[INFERRED]** = our mapping between the two.

---

## 1. Model name reconciliation

Apple's marketed model names map onto the `ModelCatalog` identifiers we recovered:

| Apple name **[OFFICIAL]** | Spec **[OFFICIAL]** | Our catalog identifier **[STRINGS]** | Mapping |
|---|---|---|---|
| **AFM 3 Core** | 3B **dense**, on-device | `com.apple.fm.language.instruct_3b.*.generic` | direct — the `3b` label matches **[INFERRED]** |
| **AFM 3 Core Advanced** | **20B sparse**, activates **1–4B** at a time, NAND-resident, natively multimodal | `com.apple.fm.language.instruct_3b.*.generic_sparse` | the `_sparse` variant; labeled by **active** scale, not 20B total **[INFERRED]** |
| **AFM 3 Cloud** | server, multimodal | `com.apple.fm.language.instruct_server_v1.*` (+ `llm_siri_pcc`) | server-tier base **[INFERRED]** |
| **AFM 3 Cloud Pro** | most capable server model | `com.apple.fm.language.instruct_server_v2.*` | likely the v2 server base **[INFERRED, unconfirmed]** |
| **ADM 3 Cloud (Image)** | image generation/editing, Genmoji | `instruct_*.adm_prompt_rewriting` / `adm_people_grounding` / `adm_background_prompt` / `adm_prompt_analyzer` | "ADM" = Apple's image model; the `adm_*` entries are **language adapters that prep prompts for it** **[INFERRED]** |
| *(not named in post)* | — | `instruct_9m`, `instruct_85m`, `instruct_300m` | smaller utility tiers Apple doesn't mention publicly **[STRINGS]** |
| *(not named in post)* | — | `com.apple.fm.code.*` (small/large v1–v5, `generate_v1_ane_3b`) | dedicated code-model family **[STRINGS]** |

**Headline correction:** an earlier version of `02` claimed "no 20B asset appears." That was wrong — see §5. Core Advanced **is** the `_sparse` variant; the catalog just names it by active (~1–4B) scale.

---

## 2. Where Apple confirms what we inferred

| Our inference | Apple's post says | Verdict |
|---|---|---|
| `system` ≠ one small model; it's a router over multiple on-device configs | Two on-device models (Core 3B dense, Core Advanced 20B sparse) + smaller specialized variants | ✅ confirmed |
| `_sparse` variant = a sparse/conditional-compute model | Core Advanced uses a **sparse architecture**, **shared experts + routed experts**, experts paged **NAND→DRAM** | ✅ confirmed (and named) |
| On-device dictation/ASR adapters (`asr_*` on `instruct_3b`) imply local multimodal speech | Core Advanced is "natively multimodal, enabling **expressive voices and higher-accuracy dictation**" | ✅ confirmed |
| `adm_*` adapters = image-model prompt prep | **ADM 3 Cloud** image model with "specialized adapters to power specific downstream editing experiences" | ✅ confirmed |
| Vision works on-device (`UAF_FM_Visual`, `image_tokenizer`) | On-device **image understanding** benchmarked (>61% preference) | ✅ confirmed |
| Weights are quantized (couldn't read bit-width) | **Quantization-Aware Training** "compressed our models substantially while maintaining high accuracy" | ✅ confirmed (bit-width still undisclosed) |
| Per-prompt expert selection inside the sparse model | "a lightweight, dense block selects a fixed set of experts… periodically reselecting them," decisions made **per prompt** not per token | ✅ confirmed |
| Two execution tiers (on-device vs PCC) | On-device models + **Private Cloud Compute** server models | ✅ confirmed |

---

## 3. What our analysis adds *beyond* the post

Apple's writeup does **not** mention these; they are only visible from the binaries/CLI:

- **The `fm` CLI itself** and **`fm serve`** — an OpenAI Chat-Completions server over the on-device model. **[OBSERVED]**
- **The full adapter catalog**: 1,037 `com.apple.fm.*` identifiers, incl. **193 server/PCC use-case adapters** (`mail_reply_*`, `magic_rewrite`, `friendly_tone`, `journal_followup_prompts`, …). **[STRINGS]**
- **Speculative decoding** — `.draft` companion models throughout (`*.answer_synthesis.draft`, `generate_v1_ane_3b.base.draft`). Apple's post is silent on draft models. **[STRINGS]**
- **The smaller tiers** `instruct_9m / 85m / 300m` and the **separate code-model family**. **[STRINGS]**
- **PCC trust mechanics**: `TC2XPCTrustedRequestProtocol`, `AcquireCoherenceTokenResponse`, attestation / anti-targetability counters, `pccCallerNotTrusted`. **[STRINGS]**
- **Siri tier routing**: `ORCHNLRouterBridge` → `RoutingDecisionSource` → `serverFallback`, `meetsUserSessionThreshold`, and the `llm_siri_pcc` / `lw_planner_v1..v5` PCC adapters. **[STRINGS]**
- **Concrete cloud endpoints**: `gr-g-carry.smoot.apple.com/v1`, `api.afm.aiml.apple.com/sage-answer-synthesis/...`. **[STRINGS]**
- **Guardrails as separate models** (`*_safety_guardrail`, `mm_guard`, `model_abuse_guardrail`). **[OBSERVED string]**

See [`02_MODEL_ARCHITECTURE.md` §4.1](02_MODEL_ARCHITECTURE.md) for the three distinct routing layers these touch.

---

## 4. Architecture specifics from Apple (for the record)

- **AFM 3 Core**: 3-billion-parameter **dense** model, on-device. **[OFFICIAL]**
- **AFM 3 Core Advanced**: **20B sparse**, activating **1–4B** parameters per request; full model in **flash (NAND)**; **shared experts** (always active) + **routed experts** (DRAM-on-demand); created via **Instruction-Following Pruning (IFP)**; per-prompt expert selection by a lightweight dense block. Natively multimodal. **[OFFICIAL]**
- **AFM 3 Cloud**: server model on a **Parallel-Track Mixture-of-Experts (PT-MoE)** foundation with "architectural refinements [that] stabilize training" and improved multimodal reasoning. **[OFFICIAL]**
- **AFM 3 Cloud Pro**: most capable server model; ≈10% better than AFM 3 Cloud on text human-eval. **[OFFICIAL]**
- **ADM 3 Cloud**: image generation/editing + Genmoji, with specialized downstream-edit adapters. **[OFFICIAL]**
- **Quantization**: **Quantization-Aware Training (QAT)**. Bit-widths not disclosed. **[OFFICIAL]**
- **Training**: pretrained on **cloud TPU accelerators**; data mixture = publicly available + licensed/purchased + open-source + dedicated studies + synthetic; **no user private data**. Post-training = **supervised fine-tuning + multi-stage reinforcement learning**. **[OFFICIAL]**

### Human-evaluation preference numbers **[OFFICIAL]**

| Capability | Model | Result |
|---|---|---|
| Text | AFM 3 Core | 45.6% preferred (vs 23.3% baseline) |
| Text | AFM 3 Cloud | 64.7% preferred (vs 8.7% predecessor) |
| Text | AFM 3 Cloud Pro | ≈10% over AFM 3 Cloud |
| Image understanding | AFM 3 Core | >61% preferred |
| Image understanding | AFM 3 Cloud | 37.8% (vs 9.6% baseline) |
| Text-to-speech (MOS) | Core Advanced | 4.15 general / 4.24 conversational (vs 3.87 / 3.82) |
| Dictation | Core Advanced | 44.7% → 17.6% quality-preference shift |

> These are Apple's self-reported human-eval figures, included for completeness; we did not reproduce them.

---

## 5. Corrections this cross-reference forces

1. **"No 20B on-device asset appears."** ❌ → ✅ Core Advanced **is** the `instruct_3b...generic_sparse` variant. The catalog labels it by **active** scale (~1–4B), and the 20B total isn't a single mapped file (sparse experts are flash-resident), so it *looked* absent. Fixed in `00` and `02`.
2. **"PCC adds latency but not capability"** (from the original behavioral notes). Apple's numbers say otherwise: AFM 3 Cloud / Cloud Pro are materially preferred over the on-device models in human eval. The earlier read came from a constrained CLI sample, not a capability ceiling.
3. **"Core Advanced may be heavily constrained / not exposed."** It's present on-device (the `_sparse` tier) and is the multimodal voice/dictation engine — not absent, just not surfaced as a distinct `fm --model` option.

---

## 6. Still undisclosed by *both* Apple and our analysis

- Exact **context-window** length, **tokenizer vocab** size, **layer counts**, **KV-cache** design.
- **Adapter/LoRA rank** and per-adapter sizes.
- **Quantization bit-widths** (QAT confirmed; numbers not given).
- Live **PCC** behavior, quotas, and the definitive `instruct_server_v2` ↔ **AFM 3 Cloud Pro** mapping.
- The numeric **routing thresholds** (`meetsUserSessionThreshold`, confidence gates).
