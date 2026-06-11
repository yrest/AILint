# AI Presence Marker — Invisible Form, Experimental Specification v0

**Status: EXPERIMENTAL — use only for prose and conversational text.**

> **Scope:** This technique applies to **unstructured prose and conversational text only** — chat messages, forum posts, plain-text documents. It is **not suitable for source code, JSON, YAML, XML, or any structured format.** For those, use the visible form (`visible-provenance-headers-v0.md`).

> The marker is a cooperative signal, not proof of AI authorship.

---

## Purpose

The invisible form exists for one specific use case: **content traveling out of context.**

When a single LLM-generated message is copied and pasted blind — to another LLM, into a document, shared as plain text — the surrounding metadata (who said what, which model, which conversation) is stripped. The invisible marker travels with the text itself and allows a receiver to detect LLM origin without any surrounding context.

This is a per-message signal, not a per-conversation or per-session signal. Each LLM message carries exactly one marker. Human messages carry none. An exported chat transcript therefore becomes individually distinguishable message by message.

---

## ⚠️ Caveats and Limitations

**Cooperative only.**
The marker is trivially removable. It is a cooperative AI-presence signal indicating the text was emitted or materially transformed by an LLM — not a cryptographic proof of authorship. Never use it as the sole basis for an origin decision.

**Beginning placement resists, not prevents, truncation.**
The marker is placed after the first character of the message. Someone copying the entire message from character 2 onward can strip it. This is deliberate evasion and an accepted limitation; casual copy-paste preserves the marker.

**Pipeline survivability is low.**
Unicode normalizers, copy-paste in many editors, log aggregators, CMS pipelines, email clients, and search indexes may silently strip the marker. Absence of a marker never implies human origin.

**Invisible Unicode is a source hygiene risk.**
These characters are flagged by SAST tools and security linters in codebases (ref: Trojan Source, CVE-2021-42574). Do not use this form in source files. Use the visible comment form instead.

**Disclosure.**
Embedding invisible characters without disclosure can be construed as covert tracking. Deployments should document what is embedded and why.

---

## The Marker

Four invisible Unicode characters. No provider. No model. No timestamp. No lineage.

| Position | Codepoint | Name | Role |
|----------|-----------|------|------|
| 1 | U+2060 | WORD JOINER | Start boundary |
| 2 | U+200C | ZERO WIDTH NON-JOINER | Marker preamble (paired with U+200D to form the signal) |
| 3 | U+200D | ZERO WIDTH JOINER | AI-present = 1 |
| 4 | U+2060 | WORD JOINER | End boundary |

U+FEFF (BOM / Zero Width No-Break Space) is deliberately avoided: renderers and
text processors may strip or misinterpret it, especially near the start of a
stream. U+200C and U+200D are natural complements with no BOM semantics.

**Pre-computed** — copy the 4 invisible characters between `→` and `←` (exclude
the arrows):

→⁠‌‍⁠←

There is exactly one canonical marker. Unlike earlier drafts of this spec, there
are no per-model or per-provider variants. The marker encodes one bit of information:
an LLM was here.

---

## Placement

Insert immediately **after the first character** of the message.

```
H[MARKER]ere is the answer to your question...
```

For a Markdown heading:
```markdown
# T[MARKER]itle
```

**Do not embed at the end of the message.** End placement is vulnerable to the
most common truncation pattern (copy the beginning, lose the tail). Beginning
placement means a full copy of any contiguous prefix preserves the signal.

**Do not embed at position 0 (before the first character).** Some renderers treat
a leading U+2060 as a formatting artifact and strip it, losing the boundary.

---

## Detection and Decoding

Scan for the two-character preamble **U+2060 followed by U+200C**. On match:

1. Read the next character. U+200D = AI-present. Anything else = malformed.
2. Expect closing U+2060 at position 3.

**Interpreting results:**

| Result | Meaning |
|--------|---------|
| Valid marker found | Content carries a cooperative AI-origin claim |
| Marker absent | No signal. Do not infer human origin — marker may have been stripped |
| Marker malformed | Log and discard; do not act on partial decodes |

---

## Idempotency

If the input already contains one or more valid AI-presence markers, emit exactly
one fresh marker in the output. Do not chain markers. Do not count how many were
in the input.

```
Input:  H[MARKER]uman pastes an LLM message here...
Output: S[MARKER]ure, here is my response...
```

Not:
```
Output: S[MARKER][MARKER]ure, here is my response...  ← wrong
```

---

## Verification

```python
import sys

PREAMBLE = '⁠‌'
AI_PRESENT = '‍'
BOUNDARY = '⁠'

text = sys.stdin.read()
i = 0
found = []
while i < len(text) - 1:
    if text[i:i+2] == PREAMBLE:
        payload = text[i+2] if i+2 < len(text) else ''
        close = text[i+3] if i+3 < len(text) else ''
        if payload == AI_PRESENT and close == BOUNDARY:
            found.append(('valid', i))
        else:
            found.append(('malformed', i))
        i += 4
    else:
        i += 1

if not found:
    print("No AILint marker found.")
for status, pos in found:
    print(f"pos {pos}: {status}")
```

---

## When Not To Use This

Do not use the invisible form in:

- Source code files of any language
- JSON, YAML, XML, TOML, or any structured data format
- Content that will pass through strict Unicode normalization
- Any context where invisible Unicode would trigger security linting

For those cases, use the visible comment or frontmatter form.

---

## Production Alternatives

This scheme is a weak, lossy, easily-stripped cooperative hint. For stronger
provenance:

- **Token-level statistical watermarking** (Kirchenbauer et al. 2023, Google SynthID): embeds signal in the sampling distribution at generation time. Survives copy-paste and light paraphrasing. Requires server-side model access.
- **Git trailers**: `AILint-Provenance:` in commit messages — durable, toolchain-compatible, signable.
- **C2PA sidecar**: cryptographically signed manifest for assets outside git.

---

*AILint AI Presence Marker — Invisible Form, Experimental v0*
*Implements: AILint Marker Specification §3.4*
