# Apple Foundation Models CLI (`fm`) — Reverse-Engineering Field Report

**Date:** 2026-06-09
**Host:** macOS 27.0 (internal SDK build `macosx27.0.internal`), Apple silicon
**Tool:** `/usr/bin/fm` — Apple Foundation Models CLI, 2.1 MB universal binary (x86_64 + arm64e)
**Method:** live CLI execution + binary/string analysis of `fm` and the `FoundationModels.framework` / `ModelCatalog` code in the dyld shared cache.

> **Evidence legend** used throughout these docs:
> - **[OBSERVED]** = directly produced by running commands on this machine.
> - **[STRINGS]** = recovered from symbol/string analysis of the signed Apple binaries. High confidence the capability exists in the OS; not necessarily reachable from `fm`.
> - **[INFERRED]** = reasoned conclusion from the above. Flagged as such.

---

## 0. TL;DR — what `fm` actually is

`fm` is a **thin Swift CLI** (Apple-signed, `com.apple.fm`, identifier `Platform=26`, runtime 27.0) that links **one public framework**: `FoundationModels.framework` (v2.0.51). It is the command-line front door to the exact same on-device stack that powers Apple Intelligence and Siri — it adds no models of its own.

It exposes **two execution tiers**, not two models:

| `fm` label | What it really is |
|------------|-------------------|
| `system`  | On-device inference via `FoundationModels` → `ModelCatalog`. **Available now.** |
| `pcc`     | Private Cloud Compute inference. **Gated by attestation/trust — unavailable in this context.** |

The user's original hypothesis ("Apple exposes execution environments, not models") is **confirmed correct**. What the original notes could *not* see — and what string analysis now reveals — is the **full model zoo** sitting behind those two tiers: at least **four on-device size tiers (9M / 85M / 300M / 3B)**, a separate **code-model family**, **vision** and **speech** models, and a **193-use-case cloud/PCC adapter catalog**, all selected by a Siri-side **orchestration router**, never by `fm`.

---

## 1. Corrections to the original session notes

The original field report (preserved verbatim in `ORIGINAL_NOTES.md`) was a good behavioral read but contained several guesses that the binary evidence now corrects:

| Original claim | Corrected finding |
|----------------|-------------------|
| "system vs pcc — PCC slightly slower, no quality jump" | **Could not be this machine.** Here, `pcc` is *unavailable* (`pccCallerNotTrusted` / "PCC inference is not available in this context"). Any earlier PCC output came from a differently-provisioned device/session. **[OBSERVED]** |
| "Only two models exist; everything else is internal" (framed as uncertainty) | **Two execution *tiers*, many models.** The `ModelCatalog` enumerates 1,000+ resource IDs: tiers `instruct_9m / 85m / 300m / 3b`, `instruct_server*`, code models, vision, speech. **[STRINGS]** |
| "system ≠ a single small model; likely a router over configs" | **Correct, and now concrete:** a shared base model per size tier + dozens of swappable **adapters** (LoRA-style), plus `.draft` speculative-decoding twins. **[STRINGS]** |
| "Core Advanced = 20B sparse, not observed" | The on-device sparse model is the **3B tier**, which ships as a **dense (`.generic`) + sparse (`.generic_sparse`) pair** (116 of its 411 catalog IDs are sparse). No 20B on-device asset appears. The "20B" figure is not represented in this build's on-device catalog. **[STRINGS]** |
| "fm serve likely OpenAI-compatible, probably /v1/chat/completions" | **Confirmed and fully tested.** OpenAI Chat Completions API with streaming, usage accounting, and OpenAI-style error envelopes. TCP **and** Unix-socket transports. **[OBSERVED]** |
| "no visible scaling with task complexity → hard inference caps" | More likely: `fm` always binds the **same general-purpose adapter on one tier**. The complexity-based escalation lives in the **Siri orchestration router**, which `fm` does not invoke. **[INFERRED from STRINGS]** |

---

## 2. What was confirmed by running it

- **Genuine Apple binary.** `codesign`: `Identifier=com.apple.fm`, Authority chain = Apple Software Signing → Apple Root CA, hardened runtime, signed 2026-05-31. Built against `macosx27.0.internal` SDK. **[OBSERVED]**
- **On-device model is live and fast.** Short greedy completions return in **~0.26 s** wall-clock, cold-ish. **[OBSERVED]**
- **PCC is blocked here.** `fm available` → `PCC inference is not available in this context. / System model available`. `fm quota-usage` → PCC `unavailable`, system `Not applicable (quota only applies to PCC)`. **[OBSERVED]**
- **Guided generation works.** Feeding a JSON schema (`fm schema object …`) constrains output to valid structured JSON — e.g. asking for "a classic sci-fi film" returned `{"title":"Blade Runner","year":1982,"rewatchable":true}`. This is **constrained decoding**, the headline `FoundationModels` feature. **[OBSERVED]**
- **`fm serve` is a real OpenAI-compatible server.** `/health`, `/v1/models`, `/v1/chat/completions` (stream + non-stream), proper `usage` token counts, `finish_reason`, and `503 service_unavailable` for `pcc`. See `01_CLI_REFERENCE.md`. **[OBSERVED]**
- **Vision & multimodal are real on-device.** `--image` is accepted by `respond` and `token-count`; the catalog contains `image_tokenizer`, `UAF_FM_Visual`, `mm_guard`, `mm_safety`. **[OBSERVED + STRINGS]**

---

## 3. The big picture (how it all fits)

```
        ┌─────────────────────────────────────────────────────────────┐
        │  Your call:  fm respond / fm chat / fm serve                  │
        │  picks an EXECUTION TIER only:  --model system | pcc          │
        └───────────────┬──────────────────────────┬───────────────────┘
                        │                           │
                 ┌──────▼──────┐            ┌────────▼─────────┐
                 │  system     │            │  pcc             │
                 │ (on-device) │            │ (Private Cloud)  │
                 └──────┬──────┘            └────────┬─────────┘
                        │                            │ attestation-gated
        ┌───────────────▼──────────────┐    ┌────────▼──────────────────┐
        │  FoundationModels.framework  │    │ PrivateCloudCompute        │
        │      → ModelCatalog          │    │  TC2XPCTrustedRequest +    │
        │  selects base + adapter      │    │  AcquireCoherenceToken     │
        └───────────────┬──────────────┘    └────────┬──────────────────┘
                        │                            │
   on-device base tiers │                  cloud base: instruct_server_v1/v2
   9m · 85m · 300m · 3b │                  + 193 use-case adapters
   (+ dense/sparse,     │                  (llm_siri_pcc, planner, mail_reply,
    + .draft spec-dec)  │                   answer_synthesis, magic_rewrite …)
                        ▼
        per-use-case ADAPTER swapped onto the base
        (messages_reply, emoji_keyword_extraction, summarizer,
         safety/guard classifiers, ADM image-prompt rewriting, …)

  ── Siri path (NOT via fm) ────────────────────────────────────────────
  SiriKit request → ORCHNLRouterBridge (RoutingDecisionSource)
        → meetsUserSessionThreshold? complexity? privacy?
        → on-device adapter  OR  serverFallback → PCC (llm_siri_pcc / lw_planner_vN)
        → cloud endpoints: gr-g-carry.smoot.apple.com/v1, api.afm.aiml.apple.com
```

Key insight for the routing question you asked: **`fm` does not route.** It pins one tier and uses the general adapter. The "local-array → cloud" routing you were looking for is a **Siri/Apple-Intelligence orchestration layer** (`ORCHNLRouterBridge`) that lives *above* `FoundationModels` and makes per-request escalation decisions. Details in `02_MODEL_ARCHITECTURE.md`.

---

## 4. What remains unknowable from here (honest limits)

- **Raw weights are unreadable.** The model payloads under `/System/Library/AssetsV2/com_apple_MobileAsset_UAF_FM_*` carry the SIP `restricted` flag (`0x80000`) and return *Operation not permitted*; `sudo` cannot prompt for a password in this non-interactive session. So exact parameter counts, quantization, and tensor layouts are **not** directly verified — the size labels (`9m/85m/300m/3b`) are Apple's own catalog identifiers, not measured. **[OBSERVED limitation]**
- **PCC behavior** cannot be exercised on this box (attestation gate). The cloud model list, endpoints, and adapter names are from **[STRINGS]**, not live calls.
- **Exact routing heuristics** (the thresholds inside `meetsUserSessionThreshold`, complexity scoring) are compiled logic, not readable constants.

---

## 5. File map of this research folder

```
fm-teardown/
├── README.md                 ← start here
├── 00_FIELD_REPORT.md        ← this file (consolidated, corrected conclusions)
├── 01_CLI_REFERENCE.md       ← every command, every flag, the serve API, copy-paste recipes
├── 02_MODEL_ARCHITECTURE.md  ← model catalog, tiers, adapters, Siri routing, PCC trust
├── ORIGINAL_NOTES.md         ← earlier behavioral session notes (host specs redacted)
├── DISCLAIMER.md             ← scope, copyright, trademark, no-affiliation
├── CONTRIBUTING.md           ← how to submit findings from other builds
├── LICENSE                   ← MIT
├── logs/
│   └── serve.log             ← captured fm serve output (loopback only)
└── artifacts/
    ├── fm_model_ids.txt       ← 1,037 com.apple.fm.* model-catalog identifiers (raw evidence)
    ├── pcc_use_cases.txt      ← 193 cloud/PCC instruct_server use-cases
    ├── cloud_endpoints.txt    ← afm.aiml.apple.com / smoot.apple.com endpoints
    ├── siri_pcc_models.txt    ← llm_siri_pcc + lw_planner_v1..v5
    └── movie_schema.json      ← sample guided-generation schema (round-tripped live)
```

> A 1.5 MB raw string dump (`artifacts/fw_strings_model.txt`) is kept locally but
> excluded from the repo (see `.gitignore`) to avoid redistributing bulk
> binary-derived content; the curated lists above carry the findings.
