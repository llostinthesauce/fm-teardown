# `fm` — Apple Foundation Models CLI: Reverse-Engineering Notes (macOS 27)

Field notes on **`/usr/bin/fm`**, the Apple Foundation Models command-line tool that shipped with macOS 27, and the on-device + Private Cloud Compute model stack behind it.

Everything here was derived from **publicly-shipped macOS binaries** using standard tools (`strings`, `otool`, `codesign`, `curl`) plus live runs of the `fm` CLI itself. No proprietary weights, source, or assets are redistributed. See [DISCLAIMER](DISCLAIMER.md).

> Snapshot from an early macOS 27 build (internal SDK `macosx27.0.internal`, `FoundationModels.framework` v2.0.51). Identifiers and behavior **will change** in later builds.

---

## TL;DR

- **`fm` is a thin Apple-signed CLI** over the public `FoundationModels.framework`. It adds no models — it's a developer front door to the same stack Siri and Apple Intelligence use.
- It exposes **two execution *tiers*, not two models**: `system` (on-device) and `pcc` (Private Cloud Compute).
- Behind those tiers, the `ModelCatalog` enumerates a real **model zoo**: on-device language tiers **`instruct_9m / 85m / 300m / 3b`**, a separate **code-model** family, **vision** and **speech** models, and a **193-use-case cloud/PCC adapter catalog**.
- The 3B on-device model ships as a **dense + sparse pair** (`.generic` / `.generic_sparse`), with **`.draft`** speculative-decoding twins for low latency (~0.26 s short completions measured).
- **`fm` does not route** between models — it pins one tier. The real local↔cloud routing is a **Siri-side orchestration layer** (`ORCHNLRouterBridge` → `serverFallback` → PCC), gated by **attestation** (`TC2XPCTrustedRequestProtocol`, coherence tokens).
- **`fm serve`** is a working **OpenAI Chat-Completions API** server (streaming + non-streaming, real token accounting, TCP or Unix socket) — drop-in for OpenAI-compatible agent frameworks.

---

## Documents

| Doc | What's in it |
|-----|--------------|
| **[00 — Field Report](00_FIELD_REPORT.md)** | Consolidated, evidence-graded conclusions; corrections to common assumptions; the big-picture diagram; honest limits. |
| **[01 — CLI Reference](01_CLI_REFERENCE.md)** | Every command and flag, the in-REPL slash commands, and the full `fm serve` API with verified request/response examples. |
| **[02 — Model Architecture](02_MODEL_ARCHITECTURE.md)** | The `ModelCatalog` layering, on-device size tiers + adapters, the code/vision/speech families, the PCC tier, trust/attestation, and how Siri routes local↔cloud. |
| **[Original Notes](ORIGINAL_NOTES.md)** | The earlier behavioral session notes that seeded this (host specs redacted), kept for provenance. |

### Evidence grading

Findings are tagged so you can tell measured facts from inference:

- **[OBSERVED]** — produced by running commands on a real macOS 27 host.
- **[STRINGS]** — recovered from symbol/string analysis of the signed Apple binaries (high confidence the capability exists in the OS; not necessarily reachable from `fm`).
- **[INFERRED]** — reasoned conclusion, flagged as such.

---

## Raw evidence (`artifacts/`)

| File | Contents |
|------|----------|
| [`fm_model_ids.txt`](artifacts/fm_model_ids.txt) | 1,037 `com.apple.fm.*` model-catalog identifiers (the core evidence) |
| [`pcc_use_cases.txt`](artifacts/pcc_use_cases.txt) | 193 cloud/PCC `instruct_server` use-case adapters |
| [`cloud_endpoints.txt`](artifacts/cloud_endpoints.txt) | `afm.aiml.apple.com` / `smoot.apple.com` inference endpoints |
| [`siri_pcc_models.txt`](artifacts/siri_pcc_models.txt) | `llm_siri_pcc` + `lw_planner_v1..v5` |
| [`movie_schema.json`](artifacts/movie_schema.json) | Sample guided-generation schema (round-tripped live) |
| `logs/serve.log` | Captured `fm serve` output (loopback only) |

---

## Reproduce it yourself

On a macOS 27 machine:

```bash
# CLI surface
fm --help
for c in respond chat token-count schema serve available quota-usage; do fm $c --help; done

# Live on-device generation
fm respond --greedy --no-stream 'Say hello in three words.'
fm available
fm quota-usage

# Guided / constrained generation
fm schema object --name Movie --string title --integer year --boolean rewatchable > movie.json
fm respond --no-stream --schema movie.json 'Give me a classic sci-fi film.'

# OpenAI-compatible server
fm serve --port 1976 &
curl -s localhost:1976/v1/models
curl -s localhost:1976/v1/chat/completions -H 'Content-Type: application/json' \
  -d '{"model":"system","messages":[{"role":"user","content":"PONG?"}],"stream":false}'
```

Binary / catalog inspection (the model-id list came from this):

```bash
codesign -dvvv /usr/bin/fm
otool -L /usr/bin/fm
CACHE=/System/Volumes/Preboot/Cryptexes/OS/System/Library/dyld
for f in "$CACHE"/dyld_shared_cache_arm64e*; do strings -a "$f"; done \
  | grep -aoE 'com\.apple\.fm\.[a-z0-9._]+' | sort -u
```

> The actual model weight files under `/System/Library/AssetsV2/com_apple_MobileAsset_UAF_FM_*` are SIP-protected (`restricted` flag) and not readable, so size labels are Apple's own catalog identifiers, not measured parameter counts.

---

## Contributing

These are notes from one macOS 27 build — findings from **other builds, hardware, or a PCC-enabled host** are very welcome. See **[CONTRIBUTING](CONTRIBUTING.md)** for what's useful and the ground rules (no proprietary content, keep the evidence grading, show your build).

## What's *not* here

- No model weights, no extracted Apple source, no proprietary assets.
- No live PCC traces — PCC was attestation-gated on the test host, so the cloud-tier details are from string analysis, not cloud calls.

See **[DISCLAIMER](DISCLAIMER.md)** for scope, copyright, and trademark notes.
