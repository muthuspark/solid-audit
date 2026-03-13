---
name: isp
description: Fix Interface Segregation Principle violations with confirmation and diff summary.
---

## Trigger

This skill activates when the user runs `/isp` (optionally with a file path argument, e.g. `/isp src/interfaces/worker.py`).

## Scope Detection

Determine which files to analyze using this priority order:

1. **Argument** — If the user provided a file or directory path after `/isp`, use that directly.
2. **Git staged** — Run `git diff --cached --name-only`. If output is non-empty, use those files.
3. **Git unstaged** — Run `git diff --name-only`. If output is non-empty, use those files.
4. **Ask user** — If no git diff is available, ask for a file or directory path.

**Always skip:** `*.lock`, `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`, `Gemfile.lock`, `composer.lock`, `**/migrations/**`, `**/__generated__/**`, `**/fixtures/**`, `**/*.min.js`

## ISP Fix Logic

### Detection

An interface/ABC violates ISP when it forces implementors to provide methods they don't actually use. Look for:

1. **Fat ABC/Protocol/interface:**
   - Python: `abc.ABC` or `typing.Protocol` with 6+ abstract methods where a concrete implementor stubs ≥ half with `pass` or `raise NotImplementedError`
   - TypeScript: `interface` with 6+ methods where an implementing class throws `new Error("not supported")` for multiple methods
   - Java: `interface` with multiple methods where an implementing class throws `UnsupportedOperationException`
   - Go: `interface` type that is larger than what any single caller actually needs
   - C#: `interface` with multiple methods where a class throws `NotImplementedException`
   - Kotlin: `interface` with multiple methods where a class uses `TODO()` or throws `UnsupportedOperationException`
   - Ruby: module with multiple methods raising `NotImplementedError` in classes that only use a subset
   - PHP: `interface` with multiple methods where a class throws `BadMethodCallException`

2. **Mixed responsibilities in one interface:** The interface has methods from two or more logically distinct domains (e.g., both `print()` and `scan()` and `send_email()`)

**Skip if:** The class is clearly a thin data container, value object, or configuration struct — not a fat interface violation.

### Fix Strategy

Split the fat interface into two or more narrow role interfaces, each grouped by a single responsibility. Concrete classes then implement only the interfaces they actually need.

**Language patterns:**

**Python:**
```python
# Before
from abc import ABC, abstractmethod

class WorkerInterface(ABC):
    @abstractmethod
    def work(self) -> None: ...
    @abstractmethod
    def eat(self) -> None: ...
    @abstractmethod
    def sleep(self) -> None: ...
    @abstractmethod
    def report_hours(self) -> int: ...

class Robot(WorkerInterface):
    def work(self) -> None: pass
    def eat(self) -> None: raise NotImplementedError
    def sleep(self) -> None: raise NotImplementedError
    def report_hours(self) -> int: return 0

# After
class Workable(ABC):
    @abstractmethod
    def work(self) -> None: ...

class HumanNeeds(ABC):
    @abstractmethod
    def eat(self) -> None: ...
    @abstractmethod
    def sleep(self) -> None: ...

class Reportable(ABC):
    @abstractmethod
    def report_hours(self) -> int: ...

class Robot(Workable, Reportable):
    def work(self) -> None: pass
    def report_hours(self) -> int: return 0
```

**TypeScript:**
```typescript
// Before
interface Printer {
  print(doc: Document): void;
  scan(doc: Document): Document;
  fax(doc: Document): void;
}

// After
interface Printable { print(doc: Document): void; }
interface Scannable { scan(doc: Document): Document; }
interface Faxable { fax(doc: Document): void; }
```

**Java:**
```java
// After
interface Workable { void work(); }
interface HumanNeeds { void eat(); void sleep(); }
interface Reportable { int reportHours(); }
```

**Go:**
```go
// After — narrow to what each caller actually needs
type Worker interface { Work() }
type Reporter interface { ReportHours() int }
```

**C#:**
```csharp
// After
interface IWorkable { void Work(); }
interface IHumanNeeds { void Eat(); void Sleep(); }
interface IReportable { int ReportHours(); }

class Robot : IWorkable, IReportable {
    public void Work() { /* impl */ }
    public int ReportHours() => 0;
}
```

**Kotlin:**
```kotlin
// After
interface Workable { fun work() }
interface HumanNeeds { fun eat(); fun sleep() }
interface Reportable { fun reportHours(): Int }

class Robot : Workable, Reportable {
    override fun work() { /* impl */ }
    override fun reportHours() = 0
}
```

**Ruby:**
```ruby
# After — split into focused modules with real implementations
# No NotImplementedError stubs needed — Robot only includes what it actually implements
module Workable; end
module HumanNeeds; end
module Reportable; end

class Robot
  include Workable
  include Reportable
  def work; end          # real implementation
  def report_hours = 0   # real implementation
end
```

**PHP:**
```php
// After
interface Workable { public function work(): void; }
interface HumanNeeds { public function eat(): void; public function sleep(): void; }
interface Reportable { public function reportHours(): int; }

class Robot implements Workable, Reportable {
    public function work(): void { /* impl */ }
    public function reportHours(): int { return 0; }
}
```

### How to Split

1. Group abstract methods by logical responsibility
2. Create one narrow interface per responsibility group
3. Update concrete classes to inherit/implement only the interfaces they need
4. Remove `raise NotImplementedError` / `throw new Error` stubs — they're no longer needed
5. Update any type hints or parameter types that used the fat interface

### Safety Check

Before proposing a fix:
- Confirm all callers that use the fat interface type only call methods within one of the split interfaces, or update them if needed
- If callers outside the reviewed scope use the fat interface type → flag as `[SKIP — callers outside scope use the full interface]`

## Confirmation Flow

For each violation found, present:

```
## ISP Fix Proposal

Found {N} ISP violation(s):

**File**: `{path}` · **Symbol**: `{InterfaceName}`
**Issue**: {one sentence — e.g., "WorkerInterface has 7 methods; Robot stubs 4 as NotImplementedError"}
**Proposed fix**: Split into {N} narrow interfaces: {Interface1}, {Interface2}, ...
- Before: {fat interface definition}
- After: {split interface definitions}
**What changes**: {plain-English diff summary}

Proceed with fix? (yes/no)
```

If the user types "yes", "y", or "proceed" → apply the fix and show the diff summary.
If the user types "no", "n", "skip", or "cancel" → do not modify any files.

## Output After Fix

```
## ISP Fix Applied

**Modified**: `{path}`
**Changes**:
{diff summary — what interfaces were created, what implementors were updated}

{If skipped:}
**Skipped**: `{path}` · `{Symbol}` — [SKIP — {reason}]
```

## Rules

1. **Always ask for confirmation** before writing any file.
2. **If user declines**: Do not modify any file.
3. **Behavior-preserving only**: Splitting an interface must not change any runtime behavior. The concrete implementations keep the same logic — only the interface hierarchy changes.
4. **Skip thin containers**: If the "fat" interface is actually a data container or value object, skip and note it.
5. **Caller scope**: If callers outside the reviewed file use the fat interface type and need updating, flag with `[SKIP — callers outside scope use the full interface]`.
6. **Out-of-scope implementors**: If the fat interface has implementors in other files not in the reviewed scope, flag as `[SKIP — implementors outside scope may be affected]` rather than splitting blindly.
7. **Minimal footprint**: Only change the interface definition and the `implements`/`extends` declarations on concrete classes. Remove stubs (`raise NotImplementedError`, `throw new Error`, `TODO()`) that are no longer needed after the split.
8. **Style matching**: Match existing code style — type hints, docstrings, import ordering.
9. **Diff summary**: After each modified file, show a plain-English summary of what changed.
10. **Test files**: Flag violations only if they cause real maintainability issues.
11. **Large files (>500 lines)**: Note that the fix may need to be applied incrementally.
