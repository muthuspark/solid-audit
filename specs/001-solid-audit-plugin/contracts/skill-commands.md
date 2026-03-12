# Contract: Skill Command Interface

**Branch**: `001-solid-audit-plugin` | **Phase**: 1 | **Date**: 2026-03-13

> This contract defines the public interface each slash command exposes to the user. It is technology-agnostic — it describes inputs, outputs, and behaviors, not implementation.

---

## /solid-audit

**Type**: Read-only audit command
**Modifies files**: No

### Input

```
/solid-audit [optional: file-or-directory-path]
```

| Argument | Required | Description |
|----------|----------|-------------|
| `path` | No | A specific file or directory to audit. If omitted, scope is auto-detected via git diff. |

### Output

A structured Markdown report containing:

```
## SOLID Audit Report

**Files reviewed**: N
**Total violations**: N

### S — Single Responsibility
- `<file>` · `<symbol>` — <reason> → <suggestion>

### O — Open/Closed
(none)

### L — Liskov Substitution
...

### I — Interface Segregation
...

### D — Dependency Inversion
...

---
Files with no violations: <file1>, <file2>
Files skipped: <lock-file>, <generated-file>
```

### Behavior Contract

- MUST list every reviewed file, including clean ones
- MUST NOT modify any file
- MUST apply language-appropriate idiom suggestions per file extension
- MUST skip: lock files, generated code, migrations, test fixtures
- MUST note when a file exceeds 500 lines
- On no git repo: MUST ask user for file paths before proceeding

---

## /srp — Single Responsibility Fix

**Type**: Fix command — Single Responsibility Principle
**Modifies files**: Yes, with confirmation

### Input

```
/srp [optional: file-or-directory-path]
```

### Output — Phase 1 (Proposal)

```
## SRP Fix Proposal

Found N SRP violation(s):

**File**: <file> · **Symbol**: <class/function>
**Issue**: <one-sentence reason>
**Proposed fix**:
- Before: <code snippet>
- After: <code snippet>
**What changes**: <diff summary>

Proceed with fix? (yes/no)
```

### Output — Phase 2 (After confirmation)

```
## SRP Fix Applied

**Modified**: <file>
**Changes**:
<diff summary>

Skipped (unsafe): <file> · <symbol> — <reason>
```

### Behavior Contract

- MUST ask for confirmation before writing any file
- MUST skip violations it cannot fix safely (flag with reason)
- MUST match existing code style (type hints, imports, docstrings)
- MUST show diff summary for each modified file
- If user declines: MUST NOT modify any file

---

## /ocp — Open/Closed Fix

**Type**: Fix command — Open/Closed Principle
**Modifies files**: Yes, with confirmation

Same input/output structure as `/srp`, targeting OCP violations (switch-on-type, direct subclass instantiation).

**Behavior Contract**: Identical to `/srp` behavior contract.

---

## /lsp — Liskov Substitution Fix

**Type**: Fix command — Liskov Substitution Principle
**Modifies files**: Yes, with confirmation

Same input/output structure as `/srp`, targeting LSP violations (subclasses throwing `NotImplementedError`, narrowing preconditions, strengthening postconditions).

**Behavior Contract**: Identical to `/srp` behavior contract.

---

## /isp — Interface Segregation Fix

**Type**: Fix command — Interface Segregation Principle
**Modifies files**: Yes, with confirmation

Same input/output structure as `/srp`, targeting ISP violations (fat interfaces/ABCs with unused methods in implementors).

**Behavior Contract**: Identical to `/srp` behavior contract.

---

## /dip — Dependency Inversion Fix

**Type**: Fix command — Dependency Inversion Principle
**Modifies files**: Yes, with confirmation

Same input/output structure as `/srp`, targeting DIP violations (concrete class instantiation in constructors, hardcoded dependencies).

**Behavior Contract**: Identical to `/srp` behavior contract.

---

## /solid-fix — Full Fix (All Principles)

**Type**: Fix command — All five SOLID principles
**Modifies files**: Yes, with single grouped confirmation

### Input

```
/solid-fix [optional: file-or-directory-path]
```

### Output — Phase 1 (Grouped Proposal)

```
## SOLID Fix Proposal

Found N violation(s) across M principle(s):

**S — Single Responsibility** (N violations)
  - <file> · <symbol>: <reason> → <proposed fix summary>

**O — Open/Closed** (N violations)
  - <file> · <symbol>: <reason> → <proposed fix summary>

[... per principle ...]

Proceed with all fixes? (yes/no)
Note: Unsafe fixes will be skipped and flagged individually.
```

### Output — Phase 2 (After confirmation)

```
## SOLID Fix Applied

**S — Single Responsibility**
  Modified: <file> — <diff summary>

**D — Dependency Inversion**
  Modified: <file> — <diff summary>
  Skipped: <file> · <symbol> — [SKIP — may affect behavior]: <reason>

Total: N files modified, M violations skipped
```

### Behavior Contract

- MUST run all five principle analyses before presenting the confirmation prompt
- MUST group the confirmation by principle (one prompt, not five)
- MUST apply fixes in order: S → O → L → I → D (to avoid conflicts between fixes)
- MUST flag and skip any violation it cannot fix safely
- MUST show per-principle diff summary after applying fixes
- If user declines: MUST NOT modify any file

---

## Cross-Command Rules

These rules apply to all seven commands:

1. **Language detection**: Determine idiom patterns from file extension (`.py` → Python, `.ts`/`.tsx` → TypeScript, `.java` → Java, `.go` → Go)
2. **Test files**: Flag violations only if they cause real maintainability issues
3. **Large files** (>500 lines): Audit normally, note that fixes may need incremental application
4. **Intentional patterns**: If a "violation" matches a known intentional pattern (thin data container, value object), skip and note it
5. **Mixed monorepo**: Apply the correct idiom per file — do not use Python idioms in TypeScript files
