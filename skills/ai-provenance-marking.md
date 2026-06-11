# AI Presence Marking Skill

## What this marks

A cooperative, binary, per-message signal: **an LLM produced this message.**

Not: which model, when, confidence, lineage, or chain of custody.

> The marker is a cooperative signal, not proof of AI authorship.

---

## The rule

Mark every LLM-generated message with exactly one canonical AI-presence marker.
Human messages carry no marker.

**Idempotency:** If your input contains one or more valid AI-presence markers, your
output still carries exactly one fresh marker. Never chain markers. Never copy
input markers into output. The state is simply:

```
human message    → no marker
LLM message      → one marker
LLM reply to LLM → one marker  (not two)
```

---

## Per-message scope

The marker annotates the individual message, not the conversation. When a single
message is copied out of context and shared blind — pasted to another LLM, dropped
into a document, forwarded as plain text — the marker travels with it. The receiver
can detect LLM origin without any surrounding context.

**Placement:** Put the marker near the **beginning** of the message, immediately
after the first character. Beginning placement resists the most common truncation
(someone copying everything except the tail). Deliberate evasion — copying from
the second character onward — can still strip it; that is an accepted limitation
of a cooperative, non-cryptographic signal.

---

## Formats

### Prose and conversational text

Insert the canonical invisible 4-character marker after the first character of
your response.

**The marker** (4 invisible Unicode characters):

| Position | Codepoint | Name |
|----------|-----------|------|
| 1 | U+2060 | WORD JOINER (start boundary) |
| 2 | U+200C | ZERO WIDTH NON-JOINER (marker preamble) |
| 3 | U+200D | ZERO WIDTH JOINER (AI-present = 1) |
| 4 | U+2060 | WORD JOINER (end boundary) |

U+FEFF is avoided — it carries BOM semantics that some renderers act on.
U+200C and U+200D are natural complements with no BOM baggage.

**Pre-computed** — copy the invisible characters between `→` and `←` (exclude
the arrows):

→⁠‌‍⁠←

**Example:**
```
H[MARKER]ere is my answer...
```

This marker does not affect rendering, copy-paste, or word count. It will be
silently stripped by many pipelines — that is an accepted property. Its value
is for systems that preserve it, not for guaranteed delivery.

---

### Code

Insert a visible comment on the **first or second line** of the file or snippet.
If a shebang (`#!`) is present, place immediately after it.

| Language | Marker line |
|----------|-------------|
| Python / Shell / Bash / Ruby / Perl / R | `# ailint: ai_present` |
| JS / TS / Java / C / C++ / C# / Go / Rust / Kotlin / Swift | `// ailint: ai_present` |
| HTML / XML / SVG | `<!-- ailint: ai_present -->` |
| CSS / SCSS / Less | `/* ailint: ai_present */` |
| SQL / Lua / Haskell | `-- ailint: ai_present` |
| MATLAB / Octave | `% ailint: ai_present` |

**Examples:**

```python
# ailint: ai_present
def process(records):
    ...
```

```javascript
// ailint: ai_present
export function clamp(n, min, max) {
    return Math.min(Math.max(n, min), max);
}
```

```html
<!-- ailint: ai_present -->
<section class="hero">
```

---

### Markdown and documents with frontmatter

Add `ailint` to the frontmatter at the absolute start of the file. If frontmatter
already exists, add the key to it.

```markdown
---
ailint:
  ai_present: true
  version: 0
---

# Document title
```

---

### JSON and strict machine-readable formats

Add `_ailint` as the **first key** in the root object. Never use invisible Unicode
in strict-format output — extra characters corrupt parsers and trigger schema
validation errors.

```json
{
  "_ailint": {"ai_present": true, "version": 0},
  "your_key": "your_value"
}
```

For a root array, wrap it:
```json
{
  "_ailint": {"ai_present": true, "version": 0},
  "data": [...]
}
```

If the schema is strict and adding a key would cause a validation failure, omit
the marker entirely rather than corrupting the format.

---

## Never

- Claim the marker proves AI authorship — it is cooperative, not cryptographic
- Include provider name, model identifier, timestamp, or confidence in the payload
- Chain markers (one in → one out, always)
- Use invisible Unicode in source code, JSON, YAML, XML, or any structured format
- Add a visible marker to pure conversational text (use the invisible form instead)
- Omit the marker from a short response because it seems unnecessary
