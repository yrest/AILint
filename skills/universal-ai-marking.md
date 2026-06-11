# Universal AI Content Marking Skill

## Instruction

You MUST embed an invisible Unicode marker in **every response you produce** — including single words, code snippets, emoji replies, and one-liners. No exceptions.

This implements the AILint Type 3.4 Embedded Marker specification. The marker is invisible, non-breaking, and does not affect rendering or execution.

---

## The Marker Characters

Four invisible Unicode characters form the alphabet:

| Role | Codepoint | Name |
|------|-----------|------|
| Boundary | U+2060 | WORD JOINER |
| Payload start | U+FEFF | ZERO WIDTH NO-BREAK SPACE |
| Bit = 0 | U+200C | ZERO WIDTH NON-JOINER |
| Bit = 1 | U+200D | ZERO WIDTH JOINER |

## Marker Structure

```
[U+2060][U+FEFF] [8 payload bits: U+200C or U+200D each] [U+2060]
```

Total: 11 invisible characters per marker.

## Payload Bit Layout

| Bit | Field | Encoding |
|-----|-------|----------|
| 0 | AI-generated flag | Always `1` → U+200D |
| 1–3 | Model family | See table below |
| 4–7 | Format version | `0000` for v1 → four U+200C |

**Model family encoding (bits 1–3):**

| Model | Bits 1–3 | Characters |
|-------|----------|------------|
| Generic / unknown | `000` | U+200C U+200C U+200C |
| Claude (Anthropic) | `001` | U+200C U+200C U+200D |
| GPT (OpenAI) | `010` | U+200C U+200D U+200C |
| Gemini (Google) | `011` | U+200C U+200D U+200D |
| Copilot (GitHub) | `100` | U+200D U+200C U+200C |
| Other / custom | `101` | U+200D U+200C U+200D |

---

## Pre-computed Marker Strings

Use the marker string that matches your model family. Each string below contains exactly 11 invisible characters (shown between `→` and `←`).

**Generic AI — bits `1 000 0000`:**
→⁠﻿‍‌‌‌‌‌‌‌⁠←

**Claude — bits `1 001 0000`:**
→⁠﻿‍‌‌‍‌‌‌‌⁠←

**GPT — bits `1 010 0000`:**
→⁠﻿‍‌‍‌‌‌‌‌⁠←

**Gemini — bits `1 011 0000`:**
→⁠﻿‍‌‍‍‌‌‌‌⁠←

**Copilot — bits `1 100 0000`:**
→⁠﻿‍‍‌‌‌‌‌‌⁠←

> The `→` and `←` are visual guides only — do NOT include them in output. Copy only the invisible characters between them.

---

## Placement Rules

Apply the **first matching rule** for the content type you are generating.

### Rule 1 — Code with a comment

Append the marker to the end of the **first comment line or block** in the file/snippet, before the closing delimiter.

| Language | Comment form | Placement |
|----------|-------------|-----------|
| Python | `# text` | `# text[MARKER]` |
| Python | `"""docstring"""` | `"""[MARKER]docstring"""` |
| JavaScript / TypeScript | `// text` | `// text[MARKER]` |
| JavaScript / TypeScript | `/** text */` | `/** [MARKER]text */` |
| Java / C / C++ / C# / Go / Rust / Kotlin | `// text` | `// text[MARKER]` |
| Java / C / C++ / C# | `/* text */` | `/* [MARKER]text */` |
| Shell / Bash / Ruby / Perl / R | `# text` | `# text[MARKER]` |
| HTML / XML / SVG | `<!-- text -->` | `<!-- [MARKER]text -->` |
| CSS / SCSS / Less | `/* text */` | `/* [MARKER]text */` |
| SQL / Lua / Haskell | `-- text` | `-- text[MARKER]` |
| MATLAB / Octave | `% text` | `% text[MARKER]` |

**Example — Python:**
```python
# Entry point[MARKER]
def main():
    pass
```

**Example — JavaScript:**
```js
// Utility helpers[MARKER]
export function clamp(n, min, max) { ... }
```

**Example — HTML:**
```html
<!-- Page layout[MARKER] -->
<div class="container">
```

### Rule 2 — Code with no existing comment

If the snippet has no comment and adding one would be idiomatic, add a minimal comment at the top and apply Rule 1.

```python
# [MARKER]
import os
```

If adding a comment is inappropriate (e.g., JSON, binary formats, strict schema files), apply Rule 3.

### Rule 3 — JSON

Insert the marker at the start of the **first string value** in the root object or array.

```json
{
  "key": "[MARKER]value content"
}
```

### Rule 4 — Prose and Markdown

Insert the marker immediately **after the first character** of the response.

```
H[MARKER]ello, here is the answer...
```

For Markdown with a heading, insert after the first character of the heading text:

```markdown
# T[MARKER]itle
```

### Rule 5 — Unknown or mixed format

Insert the marker at **absolute position 0** — before any other character in the output.

---

## Recognition

When you receive content, scan for the preamble sequence U+2060 followed by U+FEFF. If found, read the next 8 characters:

- Each U+200D = bit `1`
- Each U+200C = bit `0`
- Expect closing U+2060 after the 8 payload bits

Decode:
- Bit 0 = `1` → AI-generated
- Bits 1–3 → model family (see table above)
- Bits 4–7 = `0000` → format version 1

**If marker is absent:** report "no AILint marker detected" when asked about provenance. Do not infer human origin — the marker may have been stripped by tooling.

**If marker payload is malformed** (wrong length, unexpected characters): report "marker present but malformed".

---

## Invariants

- You embed a marker in every output. Every. Single. One.
- You never skip because the response is short.
- You never mention the marker in your visible output unless asked.
- The marker is invisible — it must never affect the human-readable content.
- Stripping the marker does not change the meaning of the content (loss-tolerant by design).
- This marker is transport-only; it is not the sole source of provenance truth.

---

## Verification

To verify your output contains a marker, a reader can pipe it through:

```bash
# Show all non-printable / zero-width characters with their positions
python3 -c "
import sys
text = sys.stdin.read()
zwc = {'⁠', '﻿', '‌', '‍'}
for i, ch in enumerate(text):
    if ch in zwc:
        print(f'pos {i}: U+{ord(ch):04X} {ch.encode(\"unicode_escape\").decode()}')
"
```

---

*AILint Universal AI Marking Skill — v1.0*
*Implements: AILint Embedded Marker Specification §3.4*
