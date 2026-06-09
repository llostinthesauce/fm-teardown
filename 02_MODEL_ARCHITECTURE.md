# Apple Foundation Models — On-Device & PCC Architecture (from binary analysis)

Everything here is **[STRINGS]** (recovered from `FoundationModels.framework` + `ModelCatalog` + Siri orchestration code in the dyld shared cache) unless marked **[OBSERVED]**. Raw evidence: `artifacts/fm_model_ids.txt` (1,037 IDs), `artifacts/pcc_use_cases.txt` (193), `artifacts/cloud_endpoints.txt`.

---

## 1. The layering

```
fm  ──links──▶  FoundationModels.framework (public, v2.0.51)
                    │  public types: SystemLanguageModel, LanguageModelSession
                    │  concrete impl: RealLanguageModelSession (respondText(), transcript mgmt)
                    ▼
                ModelCatalog (private framework)
                    │  ResourceContainer, ResourceBundleContainer,
                    │  GuardrailResultWrapper, SafetyFailureWrapper,
                    │  AvailableUseCasesWrapper, SiriResourceAvailabilityInfo,
                    │  AcquireCoherenceTokenResponse  ← PCC handshake
                    ▼
                MobileAsset payloads on disk (SIP-restricted):
                  /System/Library/AssetsV2/com_apple_MobileAsset_UAF_FM_GenerativeModels
                  …_UAF_FM_CodeLM   …_UAF_FM_Visual   …_UAF_FM_Overrides
                  …_IntelligentRouting   …_ImageCaptionModel
```

`SystemLanguageModel` / `LanguageModelSession` are the public Swift API (the same one third-party apps get). `fm` is just a CLI over them. The catalog resolves a logical model name to a **base weight set + an adapter** and (for PCC) negotiates a trusted session.

`du`/`find` on the asset dirs returns *Operation not permitted* — they carry SIP flag `0x80000` (`restricted`). So weight files exist but are unreadable without an Apple entitlement; sizes below are **catalog labels, not measured**. **[OBSERVED limitation]**

---

## 2. On-device model zoo

### 2.1 Language tiers (`com.apple.fm.language.instruct_*`)

| Tier ID | Label | Catalog IDs | Role (inferred from adapter names) |
|---------|-------|------------:|------------------------------------|
| `instruct_9m`   | ~9M  | 4   | Tiny — predictive/keyboard-class tasks |
| `instruct_85m`  | ~85M | 6   | Small utility model |
| `instruct_300m` | ~300M | 71 | **The workhorse "Core" model** — most lightweight Apple-Intelligence adapters bind here |
| `instruct_3b`   | ~3B  | **411** | **The large on-device model** ("Core Advanced"-class); dense **and** sparse |

The `instruct_3b` tier ships **dense + sparse** variants: every adapter has both `.generic` and `.generic_sparse` forms (**116 of 411** IDs are `generic_sparse`). This is the sparse/conditional-compute on-device model Apple has described. **No on-device 20B asset appears in this build** — the published "20B sparse" figure is not represented in the on-device catalog; if real it would live server-side.

Most adapters also have a `.draft` twin (e.g. `instruct_3b.answer_synthesis.draft`) → **speculative decoding**: a small draft model proposes tokens, the base verifies, for faster latency. **[OBSERVED]** short greedy gen ≈ 0.26 s.

### 2.2 What the 300M "Core" model is used for (selected adapters)

`messages_action`, `messages_reply`, `emoji_keyword_extraction`, `factual_consistency_classifier`,
`adm_prompt_rewriting` / `adm_people_grounding` / `adm_background_prompt` (Image Playground / "ADM" = Apple Diffusion/Image Model prompt prep),
`image_tokenizer`, `mm_guard`, `mm_safety`, `misc_safety*`, `ipi_classifier`, `pqa_verification`, `action_validator`, `gvicc`, `open_ended_extract` (+`.draft`).

### 2.3 What the 3B model adds (selected adapters)

`answer_synthesis`, `auto_tagger`, `autonaming_messages`, `bullets_transform`, `concise_tone`, `friendly_tone` (Writing Tools tones),
`asr_afmdictation` / `asr_fullpayloadcorrection` / `asr_natural_dictation_speech` (**dictation/ASR correction**),
`adm_prompt_analyzer`, plus the dense/sparse/draft matrix on each.

### 2.4 Separate model families

- **Code model** (`com.apple.fm.code.*`, asset `UAF_FM_CodeLM`): `generate_small_v1..v5`, `generate_large_v1..v5`, `generate_v1..v4`, and a dedicated **`generate_v1_ane_3b`** (3B, ANE-optimized, with `.draft`) + `generate_safety_guardrail`. A distinct code-completion stack (Xcode predictive code).
- **Vision** (`UAF_FM_Visual`, `image_tokenizer`, `ImageCaptionModel`): on-device multimodal image understanding. **[OBSERVED]** `--image` accepted by `respond`/`token-count`.
- **Speech** (`com.apple.fm.speech.synthesis.speech_detokenizer`): TTS/dictation pipeline.
- **Safety/guard models** as first-class catalog entries (`code_safety_guardrail`, `*_safety_guardrail`, `mm_guard`, `model_abuse_guardrail`) — guardrails are *separate models*, not just prompt rules. This is what produces `[Content blocked by safety guardrails.]`. **[OBSERVED string]**

### 2.5 Adapter naming grammar

```
com.apple.fm.language.<tier>.<use_case>[.draft][.generic][_sparse]
                       │       │          │        │       └ sparse (MoE-style) weights
                       │       │          │        └ "generic" (non-personalized) weight set
                       │       │          └ speculative-decoding draft companion
                       │       └ the task adapter (LoRA-style)
                       └ base size tier (9m/85m/300m/3b) or instruct_server*
```
`.base` = the shared base weights for that tier; `.tokenizer` = its tokenizer; `.base_adapter` = the default/general adapter (this is effectively what `fm --use-case general` binds to).

---

## 3. The PCC (cloud) tier

### 3.1 Cloud model + use-cases (`instruct_server*`)

Cloud bases: `instruct_server_small`, `instruct_server_v1`, `instruct_server_v2`. **193 server use-case adapters** (`artifacts/pcc_use_cases.txt`), most with `.draft` twins. Highlights:

- **`llm_siri_pcc`** — the adapter that runs **Siri's LLM on Private Cloud Compute**. This is the direct "Siri → cloud" path.
- **`lw_planner_v1..v5`** — "lightweight planner" (agentic task planning for Siri). Mirrors the on-device `planner_v2..v9` server-instruct configs seen in AJAX routing.
- **`answer_synthesis`**, **`magic_rewrite`**, **`open_ended_compose`** (+ `.workflow`) — Writing Tools / world-knowledge answers.
- **`mail_reply_long_form_basic` / `_rewrite` / `mail_reply_qa`** — Mail Smart Reply.
- **`friendly_tone` / `concise_tone` / `bullets_transform`** — Writing Tools transforms (cloud copies of the on-device ones).
- **`journal_followup_prompts`, `generative_shortcuts`, `fitness_workout_voice`, `accessibility_magnifier`, `autograder`, `describe_your_edit`** — feature-specific cloud adapters.
- Server-side safety: `model_abuse_guardrail`, `safety_nsfw`, `misc_safety`, `mm_guard`, `ipi_classifier`.

### 3.2 PCC trust & privacy gating (why it's "unavailable in this context")

The framework strings expose the PCC handshake:
- `PrivateCloudCompute.TC2XPCTrustedRequestProtocol` — trusted XPC request channel.
- `ModelCatalog.AcquireCoherenceTokenResponse` — a **coherence token** acquired before a PCC session.
- **Attestation** machinery: `AntiTargetability{Expected,Max}…Attestations`, `TrustedProxy{Max,Default}…Attestations`, `attestationExpiry`, `maxInlineAttestations` — PCC verifies the server fleet via attestations and resists *targeting* a specific user's request to a compromised node.
- `fm`'s own gate: code paths `FMCLI.PCC.Public` vs `FMCLI.PCC.AppleInternal`, and error `pccCallerNotTrusted`.

**[INFERRED]** On this host PCC fails the caller-trust/attestation precondition (consumer Apple-Intelligence enrolment / entitlement not satisfied for the `fm` caller), so the tier reports unavailable rather than degrading silently — consistent with Apple's privacy model.

### 3.3 Cloud endpoints (`artifacts/cloud_endpoints.txt`)

- `https://gr-g-carry.smoot.apple.com/v1/` — Siri/Apple-Intelligence "carry" inference gateway (used by `planner*`, `response_generation`, `text_summarizer`).
- `https://api.afm.aiml.apple.com/sage-answer-synthesis/api/v1/` — answer synthesis / open-ended interaction; sibling paths `…/photos-intelligence-storytelling/…`, `…/llmqu-memory-creation/…`.
- `api.smoot.apple.com` — broader Siri backend (search/bag/user).

These are the AJAX-config "static URLs" for `com.apple.fm.language.instruct_server_v1.*` use-cases.

---

## 4. How routing actually works (Siri ↔ local array ↔ cloud)

**Critical correction:** `fm` does **not** route between models. It pins one tier and uses the general adapter. Real routing is a **Siri-side orchestration layer** above FoundationModels:

- **`ORCHNLRouterBridge`** with `…RoutingDecision`, **`…RoutingDecisionSource`**, `…Started/Ended/Failed/Context/SubComponent` — Siri's NL **router bridge** that classifies each request and emits a routing decision.
- **`serverFallback*`**: `ServerFallbackMessage`, `ServerFallbackPreferences`, `serverFallbackReason`, `serverFallbackContextId` — the explicit **on-device → server escalation** mechanism, with a recorded *reason*.
- **`meetsUserSessionThreshold`**, `lowConfidenceThreshold`, `voiceConfidenceScore`, `PersonalRequest`, `siriXRedirectContext` — inputs to the decision: confidence, whether the request needs personal context, session thresholds, redirect context.
- `FallbackToIFRequestedMessage`, `FallbackToPommesMessage`, `fallbackToLegacyAllowed` — additional fallback ladders (incl. to legacy Siri / "Pommes").

**[INFERRED] routing pipeline for a Siri request:**
```
SiriKit intent / voice request
   → ORCHNLRouterBridge classifies (intent type, confidence, personal-context need, privacy)
   → tries on-device adapter on instruct_300m/3b (fast path; speculative-decoding draft)
   → if meetsUserSessionThreshold fails / low confidence / needs world knowledge:
        serverFallback → PCC (attestation + coherence token)
        → llm_siri_pcc  or  lw_planner_vN  on instruct_server_v1/v2
        → endpoint gr-g-carry.smoot.apple.com / api.afm.aiml.apple.com
   → guardrail models screen input & output at both tiers
   → response synthesized (answer_synthesis), returned to Siri UI
```
The `IntelligentRouting` MobileAsset (`com_apple_MobileAsset_IntelligentRouting`) is the downloadable policy/model backing this classifier. `ModelCatalog.SiriResourceAvailabilityInfo` is how Siri asks the catalog which adapters/tiers are currently available before routing.

So the "local array of models → cloud models" routing you asked about is **real**, but it is an **Apple-Intelligence/Siri concern, not an `fm` concern**. `fm` is a developer/debug front-end to the *primitives*, deliberately exposing only the tier choice and hiding the per-use-case adapter routing that Siri performs.

---

## 5. Open questions (not resolvable from this host)

1. Exact parameter counts / quantization — weights are SIP-locked.
2. Live PCC behavior, quotas, and whether `instruct_server_v2` is the default cloud base — PCC is attestation-blocked here.
3. The numeric thresholds inside `meetsUserSessionThreshold` / confidence gating — compiled, not string-readable.
4. Whether the `permissive` (3rd) guardrail level is reachable from `fm` or internal-only.
5. The on-device context-window ceiling — not probed in this session (next experiment: ramp `--load-transcript` / large `--text` until truncation).
