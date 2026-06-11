# Embedded Zero-Width AI-Origin Marker — Experimental Specification v0

**Status: EXPERIMENTAL — not normative, not suitable for production use without explicit policy controls.**

> **Scope:** This technique applies to **unstructured text and prose only** — chat output, forum posts, documents, email. It is **not suitable for source code** in any context where standard SDLC tooling runs. See [When Not To Use This](#when-not-to-use-this) and [Production Alternatives](#production-alternatives) below.

This document describes an optional, transport-only technique for encoding an AI-origin signal as invisible Unicode characters inside prose output. It corresponds to AILint marker type 3.4 (Embedded / Invisible Marker) in the main marker specification.

---

## ⚠️ Caveats and Limitations

Read this section before using or implementing this scheme.

**Not a proof of origin.**
The marker is trivially spoofable. Any party can copy and paste the character sequence. It signals "this content claims AI origin" — not "this content was verified as AI-generated." Treat it as a weak, non-authoritative hint, consistent with §5 (Marker Precedence) of the AILint marker spec. Never use it as the sole basis for a provenance decision.

**Model-instruction-level control is unreliable.**
If this marker is embedded by instructing a language model, any user or downstream prompt can override it with "do not include invisible Unicode characters." The marker is not enforced at the generation layer; it is a best-effort convention.

**Invisible Unicode is a source hygiene risk.**
Zero-width characters belong to the same class exploited in Trojan Source attacks (CVE-2021-42574). Since that disclosure, VS Code, GitHub, and most enterprise SAST tools actively flag bidirectional and invisible Unicode characters as severe security risks. Embedding these markers in source code means AI-generated code will immediately fail standard enterprise security linting. This is not just a fragility concern — it is a direct conflict with secure SDLC hygiene.

**Pipeline survivability for prose is also low.**
Prettier, Black, ESLint, and standard editor save-hooks strip non-printing characters outside comments and strings. Unicode normalizers (NFC/NFD/NFKC/NFKD), log aggregators, search indexes, CMS pipelines, email clients, and JSON parsers that normalize string values will silently strip or corrupt the marker. Absence of a marker never implies human origin.

**Disclosure obligation.**
Embedding invisible characters without disclosure can be construed as covert tracking or steganography even when the intent is benign. Any deployment must be accompanied by clear documentation of what is embedded and why.

**Detector before embedder.**
AILint tooling should prioritize detecting and reporting these markers before promoting their insertion. Widespread embedding without detection infrastructure creates noise, not signal.

---

## When Not To Use This

**Do not use this technique for source code.** There is no safe placement in a source file. Comments get stripped by minifiers. String literals containing invisible Unicode trigger SAST warnings. Pre-commit hooks and CI pipelines will reject or silently mutate the markers. The Trojan Source class of attacks means the security industry is actively working to eliminate invisible Unicode from codebases — embedding it intentionally puts AI-generated code on the wrong side of that process.

For source code provenance, use out-of-band approaches. See [Production Alternatives](#production-alternatives).

---

## Production Alternatives

This scheme is a weak, lossy, easily-stripped hint. For real provenance needs, use the right tool for the layer.

### For prose and text content

**Token-level statistical watermarking** embeds a signal into the actual token sampling distribution at generation time, making it statistically detectable without polluting the string data. It survives copy-paste and is robust against light paraphrasing. Key implementations and research:

- Kirchenbauer et al. (2023), "A Watermark for Large Language Models" — partitions the token vocabulary into green/red lists per secret key; the model preferentially samples green tokens; detection is statistical.
- Google SynthID (text) — similar approach; claimed robust to paraphrasing and translation.

**Constraint:** both require server-side access to the model's sampling layer. They cannot be implemented as a prompt instruction or client-side post-processing.

### For source code

**Git trailers** are the correct mechanism. They are durable, human-readable, toolchain-compatible, and can be signed.

```
commit abc123
Author: Dev <dev@example.com>

    Add rate limiting middleware

    AILint-Provenance: prov-2025-001
    Generated-By: claude-sonnet-4-6
    Generation-Role: assisted
```

**Signed commits via AI service account** — if an AI agent makes commits directly, use a dedicated service account with a verified GPG or SSH signing key. The signature proves the commit came from that account; the account's identity encodes the AI origin.

**C2PA sidecar** for assets that travel outside git (exported documentation, generated reports, media): attach a signed C2PA manifest. This is verifiable and tamper-evident in a way invisible Unicode markers are not.

---

## Relationship to C2PA / Content Credentials

For production provenance use cases, prefer signed sidecar metadata aligned with the [C2PA (Coalition for Content Provenance and Authenticity)](https://c2pa.org/) standard. C2PA uses cryptographically signed manifests attached to assets, providing verifiable, tamper-evident provenance that this scheme cannot.

This embedded marker scheme is complementary, not a substitute:

| Property | This scheme | C2PA | Git trailers |
|----------|------------|------|-------------|
| Verification | None (trivially spoofable) | Cryptographic signature | Signing key |
| Survives copy-paste | Partially | No (manifest is separate) | No (git-only) |
| Source code | Not suitable | Not designed for code | Yes |
| Prose / chat | Yes (weakly) | Impractical | Not applicable |
| Tooling required | None | C2PA-aware reader | Git |
| Suitable for compliance | No | Yes | Partial |

Use this scheme only where C2PA and Git trailers are both impractical (inline chat, ephemeral prose) and a weak, lossy, non-binding hint is acceptable.

---

## Encoding Scheme

### Character Alphabet

| Role | Codepoint | Name |
|------|-----------|------|
| Boundary | U+2060 | WORD JOINER |
| Payload start | U+FEFF | ZERO WIDTH NO-BREAK SPACE |
| Bit = 0 | U+200C | ZERO WIDTH NON-JOINER |
| Bit = 1 | U+200D | ZERO WIDTH JOINER |

### Marker Structure

```
[U+2060][U+FEFF] [8 payload bits] [U+2060]
```

11 invisible characters total. The U+2060 + U+FEFF preamble is the detection anchor.

### Payload Bit Layout

| Bit | Field | Notes |
|-----|-------|-------|
| 0 | AI-origin flag | Always `1` (U+200D) in a valid AILint marker |
| 1–3 | Model family | 3-bit code; see table below |
| 4–7 | Format version | `0000` = v0 of this spec |

**Model family codes (bits 1–3):**

| Family | Bits | Characters |
|--------|------|------------|
| Generic / unknown | `000` | U+200C U+200C U+200C |
| Claude (Anthropic) | `001` | U+200C U+200C U+200D |
| GPT (OpenAI) | `010` | U+200C U+200D U+200C |
| Gemini (Google) | `011` | U+200C U+200D U+200D |
| Copilot (GitHub) | `100` | U+200D U+200C U+200C |
| Other / custom | `101` | U+200D U+200C U+200D |

### Pre-computed Marker Strings

Each string below contains exactly 11 invisible characters, shown between `→` and `←` as visual guides (exclude the arrows when embedding).

**Generic AI — `1 000 0000`:**
→⁠﻿‍‌‌‌‌‌‌‌⁠←

**Claude — `1 001 0000`:**
→⁠﻿‍‌‌‍‌‌‌‌⁠←

**GPT — `1 010 0000`:**
→⁠﻿‍‌‍‌‌‌‌‌⁠←

**Gemini — `1 011 0000`:**
→⁠﻿‍‌‍‍‌‌‌‌⁠←

**Copilot — `1 100 0000`:**
→⁠﻿‍‍‌‌‌‌‌‌⁠←

---

## Placement Guidelines (Prose Only)

These guidelines apply only to unstructured prose and text content. Do not use this technique in source code files — see [When Not To Use This](#when-not-to-use-this).

The governing principle: place the marker where it is syntactically inert and will survive the most common transformations for the content type.

### Prose and Markdown

Insert after the first character of the first sentence or heading. This placement survives most line-wrapping and copy-paste operations better than absolute position 0.

```
H[MARKER]ere is the content...
```

For a Markdown heading:

```markdown
# T[MARKER]itle
```

### Fallback

If the content type is unclassifiable, do not embed a marker. Embedding invisible characters in an unknown format risks corrupting structured data or triggering security linting on content that never needed a marker.

---

## Code Placement — For Detectors Only

The following table documents where an earlier version of this spec suggested placing markers in source code. **This is retained solely as a reference for building detectors** that need to know where to scan for pre-existing markers. Do not use it as a guide for embedding.

| Language | Where to scan |
|----------|---------------|
| Python `#` | End of first `#` comment line |
| Python docstring | Start of module-level `"""..."""` |
| JS / TS / Java / C / C++ / C# / Go / Rust / Kotlin `//` | End of first `//` line |
| JS / TS / Java / C / C++ / C# `/* */` | Start of first `/* */` block |
| Shell / Bash / Ruby / Perl / R `#` | End of first `#` line |
| HTML / XML / SVG | Inside first `<!-- -->` comment |
| CSS / SCSS / Less | Inside first `/* */` block |
| SQL / Lua / Haskell `--` | End of first `--` line |
| MATLAB `%` | End of first `%` line |
| JSON | Start of first string value in root object/array |

---

## Detection and Decoding

Scan for the two-character preamble U+2060 U+FEFF. On match, read the next 8 characters:

- U+200D = bit `1`
- U+200C = bit `0`
- Any other character = malformed; abort

Expect a closing U+2060 at position 10. Decode the payload using the bit layout above.

**Interpreting results:**

- Marker present and valid → content carries an AI-origin claim for the indicated model family. This is not verified.
- Marker absent → no signal. Do not infer human origin; the marker may have been stripped.
- Marker malformed → log and discard; do not act on partial decodes.

---

## Verification Utility

```python
import sys

MARKER_CHARS = {'⁠', '﻿', '‌', '‍'}

text = sys.stdin.read()
hits = [(i, ch) for i, ch in enumerate(text) if ch in MARKER_CHARS]

if not hits:
    print("No AILint marker characters found.")
else:
    for pos, ch in hits:
        print(f"pos {pos}: U+{ord(ch):04X}  {ch.encode('unicode_escape').decode()}")
```

---

## Implementation Notes

- This marker is transport-only. It does not constitute full provenance on its own (AILint spec §3.4, §5).
- Stripping the marker must not invalidate the content (loss-tolerant).
- Presence must be auditable by standard Unicode inspection — this is not a covert channel.
- Tools consuming this marker should always apply the lower-confidence weight appropriate to an unsigned, spoofable signal.

---

*AILint Embedded Marker Experimental Specification — v0*
*Normative reference: AILint Marker Specification §3.4*
*See also: [C2PA Specification](https://c2pa.org/specifications/specifications/2.1/specs/C2PA_Specification.html)*
