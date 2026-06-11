# AI Provenance Marking Skill

Mark every response with a machine-readable AI provenance signal. The format depends on content type. Apply unconditionally — including short responses.

---

## Step 1 — Identify content type

| Content type | Go to |
|---|---|
| Source code (any language) | [Code](#code) |
| Markdown document or file with frontmatter | [Markdown / Documents](#markdown--documents) |
| JSON | [JSON](#json) |
| Pure prose, conversational text, or mixed/unknown | [Prose](#prose) |

---

## Code

Insert an `AILint-Origin` comment as the **first or second line** of the code block or file. If a shebang (`#!`) is present, place it immediately after.

**Format:**
```
AILint-Origin: <role>; provider=<provider>; model=<model>; reviewed=false
```

**`role` — choose one:**
- `generated` — you produced the code
- `assisted` — you completed or edited code the human started
- `summarized` — you converted or condensed existing code

**`provider` and `model` — use your own identity:**
- Anthropic / Claude: `provider=anthropic; model=claude`
- OpenAI / GPT: `provider=openai; model=gpt`
- Google / Gemini: `provider=google; model=gemini`
- GitHub Copilot: `provider=github; model=copilot`
- Unknown: `provider=unknown; model=unknown`

**Examples:**

```python
# AILint-Origin: generated; provider=anthropic; model=claude; reviewed=false
def process(records):
    ...
```

```javascript
// AILint-Origin: generated; provider=openai; model=gpt; reviewed=false
export function clamp(n, min, max) {
    return Math.min(Math.max(n, min), max);
}
```

```html
<!-- AILint-Origin: generated; provider=anthropic; model=claude; reviewed=false -->
<section class="hero">
```

```css
/* AILint-Origin: generated; provider=anthropic; model=claude; reviewed=false */
.container { max-width: 1200px; }
```

```bash
# AILint-Origin: generated; provider=anthropic; model=claude; reviewed=false
set -euo pipefail
```

```sql
-- AILint-Origin: generated; provider=openai; model=gpt; reviewed=false
SELECT user_id, COUNT(*) AS event_count
```

**Comment syntax by language:**

| Language | Syntax |
|---|---|
| Python, Shell, Bash, Ruby, Perl, R, MATLAB | `# AILint-Origin: ...` |
| JavaScript, TypeScript, Java, C, C++, C#, Go, Rust, Kotlin, Swift | `// AILint-Origin: ...` |
| HTML, XML, SVG | `<!-- AILint-Origin: ... -->` |
| CSS, SCSS, Less | `/* AILint-Origin: ... */` |
| SQL, Lua, Haskell | `-- AILint-Origin: ...` |

---

## Markdown / Documents

Add YAML frontmatter at the very start of the document, before any content.

```markdown
---
ai_origin:
  role: generated
  provider: anthropic
  model: claude
  reviewed: false
---

# Document Title

Content starts here...
```

If frontmatter is already present, add the `ai_origin` key to the existing block.

If the document format does not support frontmatter (e.g. a plain `.txt` file), treat it as [Prose](#prose).

---

## JSON

Add an `_ailint` key as the **first key** in the root object.

```json
{
  "_ailint": {
    "role": "generated",
    "provider": "anthropic",
    "model": "claude",
    "reviewed": false
  },
  "your_key": "your_value"
}
```

For a root array, wrap it:
```json
{
  "_ailint": { "role": "generated", "provider": "anthropic", "model": "claude", "reviewed": false },
  "data": [...]
}
```

---

## Prose

For conversational replies, plain text, or any content where adding a visible marker would break or intrude on the output, embed an invisible Unicode marker instead.

The marker is 11 invisible characters: a fixed preamble (U+2060 + U+FEFF), eight bits encoding AI origin and model family (U+200C = 0, U+200D = 1), and a closing boundary (U+2060).

**Insert after the first character of your response.**

Pre-computed marker strings — copy the invisible characters between `→` and `←` (exclude the arrows):

**Anthropic / Claude:**
→⁠﻿‍‌‌‍‌‌‌‌⁠←

**OpenAI / GPT:**
→⁠﻿‍‌‍‌‌‌‌‌⁠←

**Google / Gemini:**
→⁠﻿‍‌‍‍‌‌‌‌⁠←

**GitHub Copilot:**
→⁠﻿‍‍‌‌‌‌‌‌⁠←

**Generic / unknown model:**
→⁠﻿‍‌‌‌‌‌‌‌⁠←

Example (Claude, prose reply):
```
T[invisible marker]he answer is...
```

This marker is transport-only and easily stripped. It is a weak signal — see `docs/specs/embedded-marker-experimental-v0.md` for limitations.

---

## Summary

| Content | What you add | Visible? |
|---|---|---|
| Code | `# AILint-Origin: ...` comment, first line | Yes |
| Markdown doc | `ai_origin:` YAML frontmatter | Yes |
| JSON | `"_ailint": {...}` first key | Yes |
| Prose / chat | Invisible Unicode marker after first character | No |

The `reviewed` field is always `false` unless the user has explicitly told you they will edit the output before it is used.
