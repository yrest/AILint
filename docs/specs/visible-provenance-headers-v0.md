# Visible AI Provenance Headers — Specification v0

**Status: RECOMMENDED for code and structured documents.**

This specification defines the canonical format for visible, human-readable AI provenance markers in source code and structured text. It implements AILint marker types 3.2 (Inline Comment Marker) and 3.3 (Structured Header / Frontmatter) from the AILint marker specification.

These markers are designed to be:
- **Grep-able**: `grep -r "AILint-Origin"` finds all AI-touched files in a repo
- **Diff-able**: changes to provenance metadata are visible in code review
- **Linter-enforceable**: policy tools can require, validate, or block them
- **Human-readable**: no tooling required to understand what the marker says

---

## Marker Format

### Inline Comment Marker (Type 3.2)

Used in any file where a comment can be placed. Syntax is language-agnostic — the same key-value string is wrapped in whatever comment delimiter the language uses.

**Canonical string:**
```
AILint-Origin: <role>; provider=<provider>; model=<model>; reviewed=<bool>
```

**Fields:**

| Field | Required | Values |
|-------|----------|--------|
| `role` | Yes | `generated` \| `assisted` \| `summarized` \| `reviewed` \| `translated` |
| `provider` | Yes | `anthropic` \| `openai` \| `google` \| `github` \| `meta` \| `unknown` |
| `model` | Recommended | Model identifier string, e.g. `claude-sonnet-4-6`, `gpt-4o`, `gemini-2.0` |
| `reviewed` | Recommended | `true` if a human reviewed and edited the output; `false` otherwise |

**Role definitions:**

| Role | Meaning |
|------|---------|
| `generated` | Content was primarily produced by the AI with minimal human editing |
| `assisted` | Human wrote the majority; AI contributed completions, suggestions, or edits |
| `summarized` | AI summarized or condensed existing human-authored content |
| `reviewed` | Human wrote it; AI reviewed, checked, or critiqued it |
| `translated` | AI translated content from another language or format |

**Examples by language:**

```python
# AILint-Origin: generated; provider=anthropic; model=claude-sonnet-4-6; reviewed=false
def process_records(records):
    ...
```

```javascript
// AILint-Origin: assisted; provider=openai; model=gpt-4o; reviewed=true
export function clamp(n, min, max) {
    return Math.min(Math.max(n, min), max);
}
```

```java
// AILint-Origin: generated; provider=anthropic; model=claude; reviewed=false
public class RateLimiter {
```

```html
<!-- AILint-Origin: generated; provider=google; model=gemini-2.0; reviewed=false -->
<section class="hero">
```

```css
/* AILint-Origin: generated; provider=anthropic; model=claude; reviewed=false */
.container {
```

```bash
# AILint-Origin: generated; provider=anthropic; model=claude; reviewed=false
set -euo pipefail
```

```sql
-- AILint-Origin: generated; provider=openai; model=gpt-4o; reviewed=false
SELECT user_id, COUNT(*) AS event_count
```

**Placement rule:** First or second line of the file or code block. If a shebang line (`#!`) is present, place immediately after it.

---

### Structured Frontmatter Marker (Type 3.3)

Used in Markdown documents, configuration files, and any format that supports YAML/TOML/JSON frontmatter.

**YAML frontmatter:**
```yaml
---
ai_origin:
  role: generated
  provider: anthropic
  model: claude-sonnet-4-6
  reviewed: false
---
```

**TOML frontmatter (Hugo, etc.):**
```toml
+++
[ai_origin]
role = "generated"
provider = "anthropic"
model = "claude-sonnet-4-6"
reviewed = false
+++
```

**Placement rule:** At the absolute start of the file, before any content.

---

### JSON Inline Marker

For JSON files and API responses that cannot use comment syntax.

```json
{
  "_ailint": {
    "role": "generated",
    "provider": "anthropic",
    "model": "claude",
    "reviewed": false
  }
}
```

**Placement rule:** As the first key in the root object. For JSON arrays, wrap the array: `{"_ailint": {...}, "data": [...]}`.

**Note:** The `_ailint` key is a convention, not a standard. Parsers that strictly reject unknown keys will break. Verify schema compatibility before using in API responses.

---

## Detection

Any of the following patterns indicates a visible provenance marker is present:

```
# Inline comment form (any language)
AILint-Origin:

# Frontmatter form
ai_origin:

# JSON form
"_ailint"
```

**Grep one-liner to audit a repository:**
```bash
grep -rn "AILint-Origin\|ai_origin\|\"_ailint\"" --include="*.py" --include="*.js" --include="*.ts" --include="*.md" .
```

**Detect files missing a marker** (example for Python):
```bash
git ls-files "*.py" | while read f; do
    grep -qL "AILint-Origin" "$f" && echo "no marker: $f"
done
```

---

## Linter Integration

A pre-commit hook or CI check can enforce presence of markers on new files:

```bash
# .git/hooks/pre-commit (example, not production-ready)
git diff --cached --name-only --diff-filter=A | grep "\.py$" | while read f; do
    if ! grep -q "AILint-Origin" "$f"; then
        echo "Missing AILint-Origin marker in new file: $f"
        exit 1
    fi
done
```

---

## Relationship to Other Marker Types

| Scenario | Recommended marker |
|----------|--------------------|
| Source code file | Type 3.2 inline comment (this spec) |
| Markdown / config document | Type 3.3 frontmatter (this spec) |
| Git commit | Type 3.1 git trailer (`AILint-Provenance:`) |
| Ephemeral chat / prose | Type 3.4 invisible Unicode (experimental) |
| Signed/verified provenance | C2PA sidecar or signed git commit |

---

## Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "AILint Provenance Header",
  "type": "object",
  "required": ["role", "provider"],
  "properties": {
    "role": {
      "type": "string",
      "enum": ["generated", "assisted", "summarized", "reviewed", "translated"]
    },
    "provider": {
      "type": "string",
      "enum": ["anthropic", "openai", "google", "github", "meta", "unknown"]
    },
    "model": {
      "type": "string",
      "description": "Model identifier as reported by the provider, e.g. claude-sonnet-4-6"
    },
    "reviewed": {
      "type": "boolean",
      "description": "True if a human reviewed and edited the AI output before commit"
    }
  },
  "additionalProperties": false
}
```

---

*AILint Visible Provenance Headers — v0*
*Implements: AILint Marker Specification §3.2, §3.3*
