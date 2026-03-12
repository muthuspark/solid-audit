---
name: dip
description: Fix Dependency Inversion Principle violations with confirmation and diff summary.
user-invokable: true
args:
  - name: path
    description: Optional file or directory path to audit/fix instead of git diff
    required: false
---

## Trigger

This skill activates when the user runs `/dip` (optionally with a file path argument, e.g. `/dip src/services/order.py`).

## Scope Detection

Determine which files to analyze using this priority order:

1. **Argument** — If the user provided a file or directory path after `/dip`, use that directly.
2. **Git staged** — Run `git diff --cached --name-only`. If output is non-empty, use those files.
3. **Git unstaged** — Run `git diff --name-only`. If output is non-empty, use those files.
4. **Ask user** — If no git diff is available, ask for a file or directory path.

**Always skip:** `*.lock`, `package-lock.json`, `yarn.lock`, `**/migrations/**`, `**/__generated__/**`, `**/fixtures/**`

## DIP Fix Logic

### Detection

A high-level module violates DIP when it depends directly on a concrete low-level implementation instead of an abstraction. Look for:

1. **Concrete instantiation in constructor:**
   - Python: `self.db = DatabaseClient()` or `self.mailer = SmtpMailer(host="...")` inside `__init__`
   - TypeScript/Java: `this.db = new DatabaseClient()` inside a constructor
   - Go: `db: NewDatabaseClient()` assigned directly in a constructor function

2. **Hardcoded import used directly:**
   - A concrete class is imported and used directly with no interface/Protocol wrapping it
   - The class has no way to swap the implementation without editing the source

3. **Module-level singleton without injection:**
   - `DB = PostgresDatabase()` at module level, referenced directly by methods

**Do not flag:**
- `__init__` parameters that already accept an interface/Protocol type
- Constructors that accept the concrete type as a parameter (even if the type hint is concrete) — this is already injectable
- Standard library classes that are universally stable (e.g., `logging.Logger`, `pathlib.Path`)

### Fix Strategy

Introduce constructor injection with an abstract type:

1. Define a `Protocol` (Python), `interface` (TypeScript/Java), or `interface` type (Go) that declares the behavior needed
2. Add the abstraction as a constructor parameter
3. Remove the internal instantiation
4. Preserve existing default values where possible to minimize call-site changes

**Language patterns:**

**Python:**
```python
# Before
class OrderService:
    def __init__(self) -> None:
        self.db = PostgresDatabase(host="localhost", port=5432)

# After
from typing import Protocol

class Database(Protocol):
    def save(self, order: Order) -> None: ...
    def find(self, order_id: str) -> Order | None: ...

class OrderService:
    def __init__(
        self,
        db: Database = PostgresDatabase(host="localhost", port=5432),
    ) -> None:
        self.db = db
```

**TypeScript:**
```typescript
// Before
class OrderService {
  private db = new PostgresDatabase("localhost", 5432);
}

// After
interface Database {
  save(order: Order): void;
  find(orderId: string): Order | null;
}

class OrderService {
  constructor(private db: Database = new PostgresDatabase("localhost", 5432)) {}
}
```

**Java:**
```java
// Before
class OrderService {
    private final DatabaseClient db = new PostgresDatabase("localhost", 5432);
}

// After
interface Database { void save(Order order); Optional<Order> find(String id); }

class OrderService {
    private final Database db;
    public OrderService(Database db) { this.db = db; }
    // Note: Spring users can use @Autowired instead
}
```

**Go:**
```go
// Before
type OrderService struct {
    db *PostgresDatabase
}
func NewOrderService() *OrderService { return &OrderService{db: NewPostgresDatabase()} }

// After
type Database interface { Save(order Order) error; Find(id string) (Order, error) }
type OrderService struct { db Database }
func NewOrderService(db Database) *OrderService { return &OrderService{db: db} }
```

### Protocol/Interface Definition

When defining the abstraction:
- Include only the methods actually called on the concrete class within the file
- Name it after the role, not the implementation (`Database` not `PostgresDatabase`, `Mailer` not `SmtpMailer`)
- Keep it minimal — don't add methods the current code doesn't use

### Safety Check

Before proposing a fix:
- Confirm the concrete class is only instantiated in this one location in the reviewed files
- If the concrete class is instantiated in many places across multiple files → flag as `[SKIP — instantiated in multiple files outside scope]`
- If the concrete class has complex initialization that would be hard to move to a default parameter → note this in the proposal and suggest the caller provide the instance

## Confirmation Flow

For each violation found, present:

```
## DIP Fix Proposal

Found {N} DIP violation(s):

**File**: `{path}` · **Symbol**: `{ClassName}`
**Issue**: {one sentence — e.g., "`OrderService.__init__` directly instantiates `PostgresDatabase`, preventing dependency substitution"}
**Proposed fix**:
- Define `{ProtocolName}` Protocol/interface with methods: {method list}
- Add `{param_name}: {ProtocolName}` constructor parameter with default value
- Remove internal `{ConcreteClass}()` instantiation
- Before: {relevant __init__ snippet}
- After: {updated __init__ with injected parameter}
**What changes**: {plain-English diff summary}

Proceed with fix? (yes/no)
```

If the user confirms → apply the fix and show the diff summary.
If the user declines → do not modify any files.

## Output After Fix

```
## DIP Fix Applied

**Modified**: `{path}`
**Changes**:
{diff summary — Protocol/interface added, constructor updated, internal instantiation removed}

{If skipped:}
**Skipped**: `{path}` · `{Symbol}` — [SKIP — {reason}]
```

## Rules

1. **Always ask for confirmation** before writing any file.
2. **If user declines**: Do not modify any file.
3. **Behavior-preserving only**: The default value on the constructor parameter preserves existing behavior — callers that don't pass a `db` argument get the same concrete class as before.
4. **Preserve defaults**: When the concrete class has sensible default instantiation, keep it as the default parameter value. This ensures zero call-site changes in most cases.
5. **Minimal Protocol**: Define only the methods actually called on the dependency within the file — don't over-engineer the abstraction.
6. **Caller scope**: If removing the internal instantiation would require changing callers that depend on the side effects of instantiation → flag and discuss.
7. **Minimal footprint**: Add the Protocol definition and update the constructor only.
8. **Style matching**: Match existing code style — type hints, docstrings, import ordering.
9. **Diff summary**: After each modified file, show a plain-English summary of what changed.
