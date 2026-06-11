# Embedded Zero-Width AI-Origin Marker — Experimental Specification v0

**Status: EXPERIMENTAL — not normative, not suitable for production use without explicit policy controls.**

This document describes an optional, transport-only technique for encoding an AI-origin signal as invisible Unicode characters inside text or source code output. It corresponds to AILint marker type 3.4 (Embedded / Invisible Marker) in the main marker specification.

---

## ⚠️ Caveats and Limitations

Read this section before using or implementing this scheme.

**Not a proof of origin.**
The marker is trivially spoofable. Any party can copy and paste the character sequence. It signals "this content claims AI origin" — not "this content was verified as AI-generated." Treat it as a weak, non-authoritative hint, consistent with §5 (Marker Precedence) of the AILint marker spec. Never use it as the sole basis for a provenance decision.

**Invisible Unicode is a source hygiene risk.**
Zero-width characters are in the same class as the characters exploited in Trojan Source-style attacks (CVE-2021-42574). Many organizations have pre-commit hooks, linters, and CI checks that reject or flag files containing them — for good reason. GitHub itself surfaces a warning on files with hidden Unicode. Do not add these markers to source code without an explicit team policy and tooling that can audit and strip them.

**Pipeline fragility.**
The following common operations may silently strip or corrupt the marker: copy-paste into many editors, diff tools, log aggregators, search indexes, CMS pipelines, email clients, Unicode normalization (NFC/NFD/NFKC/NFKD), JSON parsers that normalize string values, and any system that filters non-printable characters. Absence of a marker never implies human origin.

**Disclosure obligation.**
Embedding invisible characters in content without disclosure can be construed as covert tracking or steganography, even when the intent is benign. Any deployment should be accompanied by clear documentation of what is embedded and why.

**Detector before embedder.**
AILint tooling should prioritize detecting and reporting these markers before promoting their insertion. Widespread embedding without corresponding detection infrastructure creates noise, not signal.

---

## Relationship to C2PA / Content Credentials

For production provenance use cases, prefer signed sidecar metadata aligned with the [C2PA (Coalition for Content Provenance and Authenticity)](https://c2pa.org/) standard. C2PA uses cryptographically signed manifests attached to assets, providing verifiable, tamper-evident provenance that this scheme cannot.

This embedded marker scheme is complementary, not a substitute:

| Property | This scheme | C2PA |
|----------|------------|------|
| Verification | None (trivially spoofable) | Cryptographic signature |
| Survives copy-paste | Partially | No (manifest is separate) |
| Source code friendly | Conditional | Not designed for code |
| Tooling required | None (human-invisible) | C2PA-aware reader |
| Suitable for compliance | No | Yes |

Use this scheme where C2PA is impractical (inline text, short snippets, chat output) and you only need a weak, lossy signal — not a binding provenance claim.

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

## Placement Guidelines

These are guidelines, not mandates. The correct placement depends on the target environment and whether zero-width characters are acceptable there at all (see caveats above).

The governing principle: place the marker where it is syntactically inert and will survive the most common transformations for that content type.

### Code — inside the first comment

Append to the end of the first comment line or block, before the closing delimiter.

| Language | Placement |
|----------|-----------|
| Python `#` | `# text[MARKER]` |
| Python docstring | `"""[MARKER]text"""` |
| JS / TS / Java / C / C++ / C# / Go / Rust / Kotlin `//` | `// text[MARKER]` |
| JS / TS / Java / C / C++ / C# `/* */` | `/* [MARKER]text */` |
| Shell / Bash / Ruby / Perl / R `#` | `# text[MARKER]` |
| HTML / XML / SVG | `<!-- [MARKER]text -->` |
| CSS / SCSS / Less | `/* [MARKER]text */` |
| SQL / Lua / Haskell `--` | `-- text[MARKER]` |
| MATLAB `%` | `% text[MARKER]` |

If no comment exists, add a minimal one only if doing so is idiomatic for the language. Do not add a comment purely to carry the marker when the file otherwise has none.

### JSON

Insert at the start of the first string value in the root object or array. Be aware that many JSON processors normalize string content; verify that the target parser preserves arbitrary Unicode before relying on this.

```json
{"key": "[MARKER]value"}
```

### Prose and Markdown

Insert after the first character of the first sentence or heading. This placement survives most line-wrapping and copy-paste operations better than position 0.

```
H[MARKER]ere is the content...
```

### Fallback

If none of the above applies, insert at absolute position 0.

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
