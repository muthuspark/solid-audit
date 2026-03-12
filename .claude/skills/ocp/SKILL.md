---
name: ocp
description: Fix Open/Closed Principle violations with confirmation and diff summary.
user-invokable: true
args:
  - name: path
    description: Optional file or directory path to audit/fix instead of git diff
    required: false
---

## Trigger

This skill activates when the user runs `/ocp` (optionally with a file path argument, e.g. `/ocp src/services/export.py`).

## Scope Detection

Determine which files to analyze using this priority order:

1. **Argument** — If the user provided a file or directory path after `/ocp`, use that directly.
2. **Git staged** — Run `git diff --cached --name-only`. If output is non-empty, use those files.
3. **Git unstaged** — Run `git diff --name-only`. If output is non-empty, use those files.
4. **Ask user** — If no git diff is available, ask for a file or directory path.

**Always skip:** `*.lock`, `package-lock.json`, `yarn.lock`, `**/migrations/**`, `**/__generated__/**`, `**/fixtures/**`

## OCP Fix Logic

### Detection

Code violates OCP when adding new behavior requires modifying existing code. Look for:

- `if type == "x": ... elif type == "y": ...` chains dispatching on a string or enum type
- `isinstance` chains used for dispatch: `if isinstance(obj, TypeA): ... elif isinstance(obj, TypeB): ...`
- Direct subclass instantiation chosen by condition: `if kind == "pdf": return PdfExporter()`
- Factory functions with growing if/elif blocks that must be edited to add new variants

### Fix Strategy

Replace the conditional dispatch with a registry or strategy pattern:

1. Define a `Protocol` (Python), `interface` (TypeScript/Java), or `interface` type (Go) for the behavior
2. Create a registry mapping type keys to implementations
3. Replace the if/elif chain with a registry lookup
4. New variants are added by registering — not by editing the dispatch function

**Language patterns:**

**Python:**
```python
from typing import Protocol

class Renderer(Protocol):
    def render(self, data: list) -> str: ...

_renderers: dict[str, Renderer] = {
    "pdf": PdfRenderer(),
    "csv": CsvRenderer(),
}

def export(data: list, format: str) -> str:
    if format not in _renderers:
        raise ValueError(f"Unknown format: {format}")
    return _renderers[format].render(data)
```

**TypeScript:**
```typescript
interface Renderer {
  render(data: unknown[]): string;
}
const renderers: Record<string, Renderer> = {
  pdf: new PdfRenderer(),
  csv: new CsvRenderer(),
};
function export_(data: unknown[], format: string): string {
  const r = renderers[format];
  if (!r) throw new Error(`Unknown format: ${format}`);
  return r.render(data);
}
```

**Java:**
```java
interface Renderer { String render(List<?> data); }
Map<String, Renderer> renderers = Map.of("pdf", new PdfRenderer(), "csv", new CsvRenderer());
```

**Go:**
```go
type Renderer interface { Render(data []interface{}) string }
var renderers = map[string]Renderer{"pdf": &PdfRenderer{}, "csv": &CsvRenderer{}}
```

### Safety Check

Before proposing a fix:
- Confirm all existing type keys are preserved in the registry
- Confirm the function signature is unchanged
- If the dispatch involves side effects that are hard to isolate, or if implementations are defined in multiple files outside the reviewed scope → mark as `[SKIP — may affect behavior]`

## Confirmation Flow

For each violation found, present:

```
## OCP Fix Proposal

Found {N} OCP violation(s):

**File**: `{path}` · **Symbol**: `{FunctionOrClassName}`
**Issue**: {one sentence explaining the violation}
**Proposed fix**:
- Before: {the if/elif chain or isinstance dispatch}
- After: {Protocol/interface + registry pattern}
**What changes**: {plain-English diff summary}

Proceed with fix? (yes/no)
```

If the user confirms → apply the fix and show the diff summary.
If the user declines → do not modify any files.

## Output After Fix

```
## OCP Fix Applied

**Modified**: `{path}`
**Changes**:
{diff summary}

{If skipped:}
**Skipped**: `{path}` · `{Symbol}` — [SKIP — may affect behavior]: {reason}
```

## Rules

1. **Always ask for confirmation** before writing any file.
2. **If user declines**: Do not modify any file.
3. **Behavior-preserving only**: All existing type keys must continue to work identically. If the refactor requires updating callers in other files, flag and skip.
4. **Invasive refactors**: If the OCP violation is deeply embedded (e.g., the conditions span multiple functions, or implementations live in other files), flag with `[SKIP — too invasive for safe automated fix]` and explain what would be needed.
5. **Minimal footprint**: Touch only the dispatch logic and the Protocol/interface definition.
6. **Style matching**: Match existing code style — type hints, docstrings, import ordering.
7. **Diff summary**: After each modified file, show a plain-English summary of what changed.
