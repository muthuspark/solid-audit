---
name: lsp
description: Fix Liskov Substitution Principle violations with confirmation and diff summary.
---

## Trigger

This skill activates when the user runs `/lsp` (optionally with a file path argument, e.g. `/lsp src/models/shapes.py`).

## Scope Detection

Determine which files to analyze using this priority order:

1. **Argument** — If the user provided a file or directory path after `/lsp`, use that directly.
2. **Git staged** — Run `git diff --cached --name-only`. If output is non-empty, use those files.
3. **Git unstaged** — Run `git diff --name-only`. If output is non-empty, use those files.
4. **Ask user** — If no git diff is available, ask for a file or directory path.

**Always skip:** `*.lock`, `package-lock.json`, `yarn.lock`, `**/migrations/**`, `**/__generated__/**`, `**/fixtures/**`

## LSP Fix Logic

### Detection

A subclass violates LSP when it cannot be used in place of its base class. Look for:

1. **NotImplementedError / not implemented throws:**
   - Python: `raise NotImplementedError` in a method inherited from the base class
   - TypeScript/Java: `throw new Error("not implemented")` or `throw new UnsupportedOperationException()` in an overriding method

2. **Narrowed preconditions:** A subclass method rejects inputs the base class accepts (e.g., base accepts `int`, subclass only accepts `int > 0`)

3. **Strengthened postconditions:** A subclass method returns a more restricted type or raises exceptions not declared in the base class contract

4. **Silent no-ops:** A subclass overrides a method with an empty body (`pass` or `{}`) that the base class defines as meaningful behavior

### Fix Strategy

**Option 1 — Flatten the hierarchy:**
If the subclass doesn't truly extend the base (e.g., `Penguin` doesn't fly like `Bird`), remove the inheritance and make each class independent.

**Option 2 — Refactor with composition:**
Replace inheritance with composition. The class holds an instance of the concrete implementation rather than inheriting from it.

**Option 3 — Narrow the base class:**
Split the base class into a more abstract base that only contains what all subclasses can truly implement. Move specialised methods to a sub-interface.

**Language patterns:**

**Python (flatten hierarchy):**
```python
# Before
class Bird:
    def fly(self) -> str: return "flying"

class Penguin(Bird):
    def fly(self) -> str: raise NotImplementedError("Penguins cannot fly")

# After
class FlyingBird:
    def fly(self) -> str: return "flying"

class Penguin:
    def swim(self) -> str: return "swimming"
```

**Python (composition):**
```python
# After (composition)
class Penguin:
    def __init__(self, swimmer: SwimBehavior) -> None:
        self._swimmer = swimmer
    def swim(self) -> str:
        return self._swimmer.swim()
```

**TypeScript (flatten):**
```typescript
interface Shape { area(): number; }
class Rectangle implements Shape { area(): number { return this.w * this.h; } }
class Square implements Shape { area(): number { return this.side ** 2; } }
```

### Safety Check

Before proposing a fix:
- Confirm the fix doesn't break callers that currently use the subclass as the base type
- If callers outside the reviewed scope pass the subclass where the base is expected → flag as `[SKIP — callers outside scope may be affected]`
- If the hierarchy is deeply nested or the base is used polymorphically in many places → flag as `[SKIP — too invasive for safe automated fix]`

## Confirmation Flow

For each violation found, present:

```
## LSP Fix Proposal

Found {N} LSP violation(s):

**File**: `{path}` · **Symbol**: `{SubclassName}`
**Issue**: {one sentence — e.g., "Penguin.fly() raises NotImplementedError, breaking substitutability for Bird"}
**Proposed fix**: {flatten hierarchy / use composition — brief description}
- Before: {relevant code snippet}
- After: {proposed fixed code}
**What changes**: {plain-English diff summary}

Proceed with fix? (yes/no)
```

If the user confirms → apply the fix and show the diff summary.
If the user declines → do not modify any files.

## Output After Fix

```
## LSP Fix Applied

**Modified**: `{path}`
**Changes**:
{diff summary}

{If skipped:}
**Skipped**: `{path}` · `{Symbol}` — [SKIP — {reason}]
```

## Rules

1. **Always ask for confirmation** before writing any file.
2. **If user declines**: Do not modify any file.
3. **Behavior-preserving only**: The fix must not change observable behavior for existing callers. If it cannot be guaranteed, flag and skip.
4. **Caller scope**: If the fix requires updating call sites outside the reviewed file(s), flag with `[SKIP — callers outside scope may be affected]`.
5. **Minimal footprint**: Touch only the class hierarchy. Do not refactor unrelated code.
6. **Style matching**: Match existing code style — type hints, docstrings, import ordering.
7. **Diff summary**: After each modified file, show a plain-English summary of what changed.
8. **Test files**: Flag only if the violation causes real maintainability issues.
