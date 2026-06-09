# Disclaimer & Scope

## What this is

Independent research notes documenting the observable behavior and structure of Apple's `fm` command-line tool and the Foundation Models stack as shipped in macOS 27.

## How the findings were obtained

- **Live execution** of the publicly-installed `/usr/bin/fm` CLI.
- **Static inspection** of publicly-shipped, Apple-signed system binaries using standard, read-only developer tools: `codesign`, `otool`, `strings`, `file`, `curl`.

No security controls were bypassed. SIP-protected model weights were **not** accessed or extracted — where they were encountered, the access was denied by the OS and that limitation is documented.

## What is *not* included or redistributed

- No model weights, embeddings, or other proprietary assets.
- No Apple source code or decompiled logic.
- No bulk copies of Apple binaries. The published `artifacts/` are short, factual lists of **identifiers** (model/use-case names, endpoint hostnames) recovered from string output — the findings, not the binaries.

## Accuracy & versioning

Findings come from an early/internal macOS 27 build (`FoundationModels.framework` v2.0.51, SDK `macosx27.0.internal`). Identifiers, model tiers, endpoints, and behavior are expected to change in later builds. Conclusions marked **[INFERRED]** are reasoned interpretation, not confirmed fact. Treat everything here as a point-in-time snapshot, not Apple documentation.

## No affiliation

This project is **not** affiliated with, authorized by, or endorsed by Apple Inc. "Apple", "Siri", "Private Cloud Compute", "Apple Intelligence", "HomeKit", and related names are trademarks of Apple Inc., used here only for identification and commentary.

## Purpose

Educational and research commentary. Provided as-is, without warranty. Verify independently before relying on anything here.
