# AI Presence Marker — Visible Form, Specification v0

**Status: RECOMMENDED for code and structured documents.**

This specification defines the canonical visible form of the AILint AI-presence marker for source code and structured text. It implements AILint marker types 3.2 (Inline Comment Marker) and 3.3 (Structured Header / Frontmatter).

> The marker is a cooperative signal, not proof of AI authorship.

The visible form carries exactly one claim: **an LLM produced or materially
transformed this content.** It does not encode provider, model, timestamp,
confidence, or lineage.

---

## Design properties

- **Grep-able:** `grep -r "ailint: ai_present"` audits an entire repo
- **Diff-able:** changes to marker presence are visible in code review
- **Linter-enforceable:** CI checks can require, validate, or block the marker
- **Human-readable:** no tooling required to understand what it says
- **Minimal payload:** one boolean claim, one version field, nothing else

---

## Formats

### Inline comment (Type 3.2)

The same key-value string wrapped in the host language's comment syntax:

```
ailint: ai_present
```

**Placement:** first or second line of the file or code block. After a shebang
(`#!`) if present.

**By language:**

| Language | Full marker line |
|----------|-----------------|
| Python / Shell / Bash / Ruby / Perl / R / MATLAB | `# ailint: ai_present` |
| JS / TS / Java / C / C++ / C# / Go / Rust / Kotlin / Swift | `// ailint: ai_present` |
| HTML / XML / SVG | `<!-- ailint: ai_present -->` |
| CSS / SCSS / Less | `/* ailint: ai_present */` |
| SQL / Lua / Haskell | `-- ailint: ai_present` |

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

```css
/* ailint: ai_present */
.container { max-width: 1200px; }
```

```bash
# ailint: ai_present
set -euo pipefail
```

```sql
-- ailint: ai_present
SELECT user_id, COUNT(*) AS event_count
```

---

### Structured frontmatter (Type 3.3)

For Markdown, configuration files, and any format supporting YAML/TOML frontmatter.

**YAML:**
```yaml
---
ailint:
  ai_present: true
  version: 0
---
```

**TOML:**
```toml
+++
[ailint]
ai_present = true
version = 0
+++
```

**Placement:** absolute start of the file, before any content. If frontmatter
already exists, add the `ailint` key to it.

---

### JSON inline field

For JSON files where comment syntax is unavailable.

```json
{
  "_ailint": {"ai_present": true, "version": 0},
  "your_key": "your_value"
}
```

**Caveat:** parsers with strict schema validation may reject unknown keys.
Verify compatibility before using in API responses or schema-validated configs.

---

## Detection

```bash
# Find all marked files in a repo
grep -rn "ailint: ai_present\|\"_ailint\"\|ai_present: true" .

# Find unmarked Python files (example policy check)
git ls-files "*.py" | while read f; do
    grep -qL "ailint: ai_present" "$f" && echo "no marker: $f"
done
```

---

## Schema (frontmatter / JSON form)

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "AILint AI-Presence Marker",
  "type": "object",
  "required": ["ai_present"],
  "properties": {
    "ai_present": {
      "type": "boolean",
      "const": true
    },
    "version": {
      "type": "integer",
      "const": 0,
      "description": "Spec version. Currently always 0."
    }
  },
  "additionalProperties": false
}
```

---

## What this marker does not claim

- It does not identify the provider or model
- It does not record when the content was generated
- It does not indicate how much of the content is AI-generated vs human-edited
- It does not prove the claim — any file can have the marker added manually

These omissions are intentional. The marker makes only the claim it can honestly
make without cryptographic infrastructure: "an LLM was present."

---

## Relationship to other marker types

| Scenario | Recommended approach |
|----------|---------------------|
| Source code file | Type 3.2 inline comment (this spec) |
| Markdown / config document | Type 3.3 frontmatter (this spec) |
| JSON / structured data | `_ailint` field (this spec) |
| Conversational text / prose | Type 3.4 invisible marker (see `embedded-marker-experimental-v0.md`) |
| Git commit | Type 3.1 git trailer (`AILint-Provenance:`) |
| Verified / signed provenance | C2PA sidecar or signed git commit |

---

*AILint AI Presence Marker — Visible Form, v0*
*Implements: AILint Marker Specification §3.2, §3.3*
