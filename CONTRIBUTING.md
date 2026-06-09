# Contributing

These are point-in-time notes from **one** macOS 27 build. The most useful contributions are **findings from other builds, hardware, and account states** — especially a host where **Private Cloud Compute is actually available**, which the original author couldn't exercise.

## Good things to contribute

- **New / changed model identifiers** from a different macOS build (the `com.apple.fm.*` catalog shifts between builds).
- **Live PCC behavior** — `fm available`, `fm quota-usage`, latency, and `pcc` responses on an Apple-Intelligence-enrolled host.
- **Corrections** to anything marked **[INFERRED]** that you can move to **[OBSERVED]** with evidence.
- **`fm serve` client recipes** — wiring it into agent frameworks, the Unix-socket path, edge cases.
- **Context-window / reasoning-ceiling probes** and other reproducible experiments.

## Ground rules

1. **No proprietary content.** Do not submit model weights, extracted Apple source, decompiled logic, or bulk copies of Apple binaries. Curated lists of **identifiers** (names, endpoints) are fine; multi-MB raw dumps are not.
2. **Read-only, no bypassing protections.** Use standard tools (`fm`, `codesign`, `otool`, `strings`, `curl`). Don't defeat SIP or any security control to get at restricted assets — note the limitation instead, like the existing docs do.
3. **Keep the evidence grading.** Tag claims `[OBSERVED]`, `[STRINGS]`, or `[INFERRED]` so readers can tell measured facts from interpretation.
4. **Show your build.** Include `sw_vers`, the `FoundationModels.framework` version (`otool -L /usr/bin/fm`), and architecture so findings are comparable across builds.
5. **Make it reproducible.** Paste the exact command(s) and trimmed output.

## How

Open an issue with your build version and findings, or send a PR that adds to the relevant `.md` / `artifacts/` file. Small, well-scoped, well-cited contributions get merged fastest. By contributing you agree your work is released under the repo's [MIT License](LICENSE), within the bounds of the [DISCLAIMER](DISCLAIMER.md).
