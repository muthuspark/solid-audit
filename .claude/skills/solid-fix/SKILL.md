---
name: solid-fix
description: Fix all SOLID principle violations across all five principles with a single grouped confirmation.
user-invokable: true
args:
  - name: path
    description: Optional file or directory path to audit/fix instead of git diff
    required: false
---

## Trigger

This skill activates when the user runs `/solid-fix` (optionally with a file path argument, e.g. `/solid-fix src/services/order.py`).

## Scope Detection

Determine which files to analyze using this priority order:

1. **Argument** — If the user provided a file or directory path after `/solid-fix`, use that directly.
2. **Git staged** — Run `git diff --cached --name-only`. If output is non-empty, use those files.
3. **Git unstaged** — Run `git diff --name-only`. If output is non-empty, use those files.
4. **Ask user** — If no git diff is available, ask for a file or directory path.

**Always skip:** `*.lock`, `package-lock.json`, `yarn.lock`, `**/migrations/**`, `**/__generated__/**`, `**/fixtures/**`

## Orchestration Logic

Run all five SOLID principle analyses **sequentially** in this order: **S → O → L → I → D**

This order is deliberate:
- SRP extraction changes the class structure that OCP/DIP fixes might also target
- Running S first ensures the class boundaries are stable before open/closed or injection fixes are proposed

### Step 1 — Collect All Violations

For each principle in order, analyze the scoped files:

**S — Single Responsibility**: Classes/functions with multiple unrelated concerns
**O — Open/Closed**: Switch-on-type chains, `isinstance` dispatch, direct subclass instantiation by condition
**L — Liskov Substitution**: `raise NotImplementedError` in overrides, narrowed preconditions, strengthened postconditions
**I — Interface Segregation**: Fat ABCs/interfaces where implementors stub ≥ half the methods
**D — Dependency Inversion**: Concrete class instantiation in constructors, no abstraction layer

Group all violations by principle. A violation flagged as unsafe (`safe_to_fix = false`) during analysis is kept in the report but marked as skipped.

### Step 2 — Present Grouped Confirmation Prompt

Present **one single confirmation prompt** listing all violations grouped by principle:

```
## SOLID Fix Proposal

Found {N} violation(s) across {M} principle(s):

**S — Single Responsibility** ({n} violation(s))
  - `{file}` · `{Symbol}`: {one-sentence reason} → {proposed fix summary}
  [additional violations...]

**O — Open/Closed** ({n} violation(s))
  - `{file}` · `{Symbol}`: {one-sentence reason} → {proposed fix summary}

**L — Liskov Substitution** ({n} violation(s))
  - `{file}` · `{Symbol}`: {one-sentence reason} → {proposed fix summary}

**I — Interface Segregation** ({n} violation(s))
  - `{file}` · `{Symbol}`: {one-sentence reason} → {proposed fix summary}

**D — Dependency Inversion** ({n} violation(s))
  - `{file}` · `{Symbol}`: {one-sentence reason} → {proposed fix summary}

Already marked as unsafe (will be skipped):
  - `{file}` · `{Symbol}` ({Principle}) — [SKIP — {reason}]

Proceed with all fixes? (yes/no)
Note: Any additional fixes that cannot be confirmed as behavior-preserving during application will also be skipped and flagged.
```

If the user types "yes", "y", or "proceed" → proceed to Step 3.
If the user types "no", "n", "skip", or "cancel" → do not modify any files.

### Step 3 — Apply Fixes in Order

Apply confirmed fixes in order: **S → O → L → I → D**

For each fix:
1. Attempt to apply the behavior-preserving transformation
2. If during application the fix cannot be confirmed safe → flag with `[SKIP — may affect behavior]: {reason}` and continue to the next fix
3. After each **principle group** (not each individual fix), show a brief progress note

## Output After Fix

```
## SOLID Fix Applied

**S — Single Responsibility**
  ✓ Modified: `{file}` — {diff summary}
  ✗ Skipped: `{file}` · `{Symbol}` — [SKIP — {reason}]

**O — Open/Closed**
  ✓ Modified: `{file}` — {diff summary}

**L — Liskov Substitution**
  (no safe fixes)

**I — Interface Segregation**
  ✓ Modified: `{file}` — {diff summary}

**D — Dependency Inversion**
  ✓ Modified: `{file}` — {diff summary}

---
**Total**: {N} file(s) modified, {M} violation(s) skipped
```

## Rules

1. **One confirmation prompt**: Always collect all violations first, then ask once — never prompt per-principle.
2. **If user declines**: Do not modify any file.
3. **Application order is fixed**: Always apply in S → O → L → I → D order, regardless of what the user requested.
4. **Skip-on-uncertainty**: If a fix cannot be confirmed as behavior-preserving during application, flag `[SKIP — may affect behavior]` and continue — do not block other fixes.
5. **Unsafe violations pre-flagged in the proposal**: Violations identified as unsafe during analysis are shown in the proposal under "Already marked as unsafe" — the user can see them but they will be skipped regardless of confirmation.
6. **Per-principle diff summaries**: After applying all fixes, show a grouped diff summary per principle.
7. **Style matching**: All fixes must match existing code style — type hints, docstrings, import ordering.
8. **Large files (>500 lines)**: Note that fixes may need to be applied incrementally for each affected file.
9. **No git repo / empty diff**: Ask the user for file paths before proceeding — never fail silently.
