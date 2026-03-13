---
name: srp
description: Fix Single Responsibility Principle violations with confirmation and diff summary.
---

## Trigger

This skill activates when the user runs `/srp` (optionally with a file path argument, e.g. `/srp src/services/user.py`).

## Scope Detection

Determine which files to analyze using this priority order:

1. **Argument** — If the user provided a file or directory path after `/srp`, use that directly.
2. **Git staged** — Run `git diff --cached --name-only`. If output is non-empty, use those files.
3. **Git unstaged** — Run `git diff --name-only`. If output is non-empty, use those files.
4. **Ask user** — If no git diff is available, ask for a file or directory path.

**Always skip:** `*.lock`, `package-lock.json`, `yarn.lock`, `**/migrations/**`, `**/__generated__/**`, `**/fixtures/**`

## SRP Fix Logic

### Detection

A class or function violates SRP when it handles two or more **unrelated** concerns. Look for:

- A class with method groups belonging to distinct responsibilities (e.g., data parsing AND persistence AND notification in one class)
- A function that validates, persists, and formats output all in one body
- A class whose name contains "And", "Manager", or "Handler" and does multiple unrelated things
- A class that imports from two or more unrelated domains (e.g., imports both `smtplib` and database drivers)

### Fix Strategy

Extract each distinct responsibility into its own focused class or function:

1. Identify the distinct responsibility groups within the violating class/function
2. For each group, create a new class/function with a single, clear name
3. Move the relevant methods/logic to the new class
4. Update the original class to accept the new classes via constructor injection (if needed) or remove it entirely
5. Preserve the original public interface — callers of the original class must not need to change

**Language patterns:**
- **Python**: Extract to new `class`, use `@dataclass` for value objects, inject dependencies via `__init__`
- **TypeScript**: Extract to new `class` or module, inject via constructor
- **Java**: Extract to new `class`, inject via constructor
- **Go**: Extract to new `struct` or standalone functions
- **C#**: Extract to new `class`; use `record` for value objects; inject via constructor
- **Kotlin**: Extract to new `class`; use `data class` for value objects; inject via constructor
- **Ruby**: Extract to new `class`; move shared behavior to a `module` if needed
- **PHP**: Extract to new `class`; use `trait` for shared behavior

**Style matching:**
- Match existing type annotation style (Python type hints, TypeScript types)
- Match existing docstring format
- Preserve import ordering conventions
- Do not add dependencies not already in the file

### Safety Check

Before proposing a fix, verify:
- The extracted class has a clear, single responsibility name
- The public interface of the original class is preserved (no renamed public methods)
- All internal references are updated (e.g., `self.old_method()` → `self.new_service.method()`)
- If callers outside the reviewed file must change, mark as `[SKIP — may affect behavior]`

## Confirmation Flow

For each violation found, present:

```
## SRP Fix Proposal

Found {N} SRP violation(s):

**File**: `{path}` · **Symbol**: `{ClassName}`
**Issue**: {one sentence explaining the violation}
**Proposed fix**:
- Before: {relevant code snippet showing the mixed responsibilities}
- After: {proposed extracted classes/functions}
**What changes**: {plain-English diff summary — what moves where}

Proceed with fix? (yes/no)
```

If the user types "yes", "y", or "proceed" → apply the fix and show the diff summary.
If the user types "no", "n", "skip", or "cancel" → do not modify any files.

## Output After Fix

```
## SRP Fix Applied

**Modified**: `{path}`
**Changes**:
{diff summary — what was extracted, what was added, what was updated}

{If any violations were skipped:}
**Skipped**: `{path}` · `{Symbol}` — [SKIP — may affect behavior]: {reason}
```

## Rules

1. **Always ask for confirmation** before writing any file. Never apply changes silently.
2. **If user declines**: Do not modify any file, not even partially.
3. **Behavior-preserving only**: If a fix cannot be confirmed as behavior-preserving (e.g., requires updating callers outside the reviewed scope), flag with `[SKIP — may affect behavior]: {reason}` and skip it.
4. **Minimal footprint**: Touch only the lines needed for the fix. Do not refactor unrelated code.
5. **Style matching**: Match existing code style — type hints, docstrings, import ordering.
6. **Diff summary**: After each modified file, always show a plain-English summary of what changed.
7. **Test files**: Flag violations only if they cause real maintainability issues.
8. **Large files (>500 lines)**: Note that the fix may need to be applied incrementally.
