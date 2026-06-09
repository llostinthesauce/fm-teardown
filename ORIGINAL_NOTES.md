# Original Session Notes (pasted, verbatim)

> These are the prior-session field notes that seeded this investigation.
> Preserved as-is except for host hardware specs, redacted for privacy.
> Where these conclusions were corrected by binary evidence,
> see `00_FIELD_REPORT.md` §1 ("Corrections") and `02_MODEL_ARCHITECTURE.md`.
> Most important correction: PCC was **unavailable** on the host used for *this*
> investigation, so the system-vs-PCC comparisons below must have come from a
> differently provisioned device/session.

---

Below is a consolidated, structured "field report" of what your experiments imply. This is written to be directly consumable by a local agent (OpenCode / OpenDevin-style / custom REPL agent).

## Apple Foundation Models CLI — Reverse Engineering Notes (Session Summary)

### 0. Environment Observed
- Device: Apple silicon Mac (macOS 27 beta)
- Tool: `fm` Apple Foundation Models CLI
- Models exposed: `system` (on-device), `pcc` (Private Cloud Compute)

### 1. CLI Surface Area
1.1 Commands: respond, chat, token-count, schema, serve, available, quota-usage
1.2 Model abstraction layer — only two models exposed: system = on-device runtime, pcc = private cloud runtime. No explicit exposure of AFM 3 Core / Core Advanced / Cloud / Cloud Pro. ⇒ Strong evidence Apple hides model routing behind 2-tier abstraction.

### 2. Observed Behavior Across Tests
2.1 System vs PCC outputs — Simple "hello": system "Hello!"; PCC "Hi! How can I help you today?" → cosmetic only.
Medium reasoning (essay ~1000 words): both coherent, similar; latency system ~0.49s, PCC ~0.75s → PCC slower, not better.
Logic (3-switch puzzle): both correct.
Scientific (QFT): correct high-level, no derivation → "textbook summary ceiling".
Vision: system correctly identifies screenshot → local multimodal confirmed.

### 3. Key Behavioral Inferences
3.1 system is NOT a single small model (essays, logic, image input, long coherence) ⇒ routing layer over multiple on-device AFM configs.
3.2 No observable difference between system and PCC quality ⇒ PCC not exposing a dramatically larger frontier model in this CLI.
3.3 Shared characteristics indicate a capped inference tier.

### 4. Likely Internal Architecture (Inferred)
4.1 SYSTEM router → Core (fast ~3B dense) / Core Advanced (sparse MoE 1–4B active) / multimodal adapter.
4.2 PCC router → Cloud (general) / Cloud Pro (agentic) / ADM Cloud (image gen).

### 5. Discrepancies vs Apple Paper
Apple claims Core Advanced = 20B sparse, 1–4B active, multimodal, adaptive expert routing, NAND-backed expert storage. Not observed in CLI: adaptive depth scaling, stronger reasoning, tier differentiation. ⇒ Core Advanced not fully exposed or heavily constrained.

### 6. Key Architectural Hypothesis
6.1 Apple exposes execution environments, not models. CLI labels system/pcc = inference routers. 6.2 Hidden internal routing likely exists.

### 7. Most Important Findings
7.1 Vision works locally. 7.2 PCC adds latency but not capability jump. 7.3 System more capable than expected. 7.4 No visible scaling with complexity.

### 8. fm serve implication (critical)
Likely exposes OpenAI-compatible API (/v1/chat/completions), routes through SYSTEM or PCC, hides AFM model selection. Usable by agent frameworks.

### 9. What is NOT confirmed
Whether Core Advanced fully active; whether PCC exposes Cloud Pro; actual parameter counts; NAND→DRAM expert loading; real routing heuristics.

### 10. High-value next experiments
10.1 Inspect serve. 10.2 Verbose inference. 10.3 Reasoning ceiling load test. 10.4 Vision stress test. 10.5 Context window test (10k–50k). 10.6 System vs PCC parity sweep.

### 11. Core conclusion
Apple's CLI does NOT expose models — it exposes a two-tier execution abstraction (local vs private cloud). All model routing (Core, Core Advanced, Cloud, Cloud Pro, vision) is hidden behind these two endpoints. system = constrained on-device AFM router; pcc = constrained cloud AFM router; no visible frontier reasoning tier exposed in CLI.
