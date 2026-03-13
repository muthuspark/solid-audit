---
name: dip
description: Fix Dependency Inversion Principle violations with confirmation and diff summary.
---

## Trigger

This skill activates when the user runs `/dip` (optionally with a file path argument, e.g. `/dip src/services/order.py`).

## Scope Detection

Determine which files to analyze using this priority order:

1. **Argument** — If the user provided a file or directory path after `/dip`, use that directly.
2. **Git staged** — Run `git diff --cached --name-only`. If output is non-empty, use those files.
3. **Git unstaged** — Run `git diff --name-only`. If output is non-empty, use those files.
4. **Ask user** — If no git diff is available, ask for a file or directory path.

**Always skip:** `*.lock`, `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`, `Gemfile.lock`, `composer.lock`, `**/migrations/**`, `**/__generated__/**`, `**/fixtures/**`, `**/*.min.js`

## DIP Fix Logic

### Detection

A high-level module violates DIP when it depends directly on a concrete low-level implementation instead of an abstraction. Look for:

1. **Concrete instantiation in constructor:**
   - Python: `self.db = DatabaseClient()` or `self.mailer = SmtpMailer(host="...")` inside `__init__`
   - TypeScript/Java: `this.db = new DatabaseClient()` inside a constructor
   - Go: `db: NewDatabaseClient()` assigned directly in a constructor function
   - C#: `_db = new PostgresDatabase(...)` inside a constructor body
   - Kotlin: `val db = PostgresDatabase(...)` hardcoded as a constructor body assignment
   - Ruby: `@db = PostgresDatabase.new(...)` inside `initialize`
   - PHP: `$this->db = new PostgresDatabase(...)` inside `__construct`

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

**For constructor/`__init__` instantiation:**

1. Define a `Protocol` (Python), `interface` (TypeScript/Java/C#/PHP), or `interface` type (Go/Kotlin) that declares the behavior needed
2. Add the abstraction as a constructor parameter
3. Remove the internal instantiation
4. Preserve existing default values where possible to minimize call-site changes

**For module-level singletons (`DB = PostgresDatabase()` at module level):**

1. Define the same interface/Protocol as above
2. Move the instantiation to the call site or to an application entry-point (e.g., `main.py`, `Program.cs`, `main.go`)
3. Pass the instance to the classes that need it via constructor injection
4. Flag as `[SKIP — module-level singleton; refactor requires coordinating multiple call sites]` if the singleton is used in many files outside scope

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

**C#:**
```csharp
// Before
class OrderService {
    private readonly PostgresDatabase _db = new("localhost", 5432);
}

// After
interface IDatabase { void Save(Order order); Order? Find(string id); }

class OrderService {
    private readonly IDatabase _db;
    public OrderService(IDatabase db) { _db = db; }
    // Note: ASP.NET Core users register IDatabase in IServiceCollection
}
```

**Kotlin:**
```kotlin
// Before
class OrderService {
    private val db = PostgresDatabase("localhost", 5432)
}

// After
interface Database { fun save(order: Order); fun find(id: String): Order? }

class OrderService(private val db: Database) {
    // Android users: inject Database via Hilt or Koin
}
```

**Ruby:**
```ruby
# Before
class OrderService
  def initialize
    @db = PostgresDatabase.new(host: "localhost", port: 5432)
  end
end

# After — keyword argument with default preserves existing call sites
class OrderService
  def initialize(db: PostgresDatabase.new(host: "localhost", port: 5432))
    @db = db
  end
end
```

**PHP:**
```php
// Before
class OrderService {
    private PostgresDatabase $db;
    public function __construct() { $this->db = new PostgresDatabase("localhost", 5432); }
}

// After
interface DatabaseInterface { public function save(Order $order): void; public function find(string $id): ?Order; }

class OrderService {
    public function __construct(private readonly DatabaseInterface $db) {}
    // Symfony/Laravel DI containers resolve DatabaseInterface automatically
}
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

If the user types "yes", "y", or "proceed" → apply the fix and show the diff summary.
If the user types "no", "n", "skip", or "cancel" → do not modify any files.

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
3. **Behavior-preserving only**: The default value on the constructor parameter preserves existing behavior — callers that don't pass a `db` argument get the same concrete class as before. Always add the default when the concrete class has a clear default instantiation. If no sensible default is possible, call sites will need to be updated — warn the user.
4. **Preserve defaults**: When the concrete class has sensible default instantiation, keep it as the default parameter value. This ensures zero call-site changes in most cases.
5. **Do not flag**: Standard library classes (`logging.Logger`, `pathlib.Path`, `fmt`, `Console`), parameters that already accept an interface type, or constructors where the dependency is already injected.
6. **Minimal Protocol**: Define only the methods actually called on the dependency within the file — don't over-engineer the abstraction.
7. **Caller scope**: If removing the internal instantiation would require changing callers that depend on the side effects of instantiation → flag and discuss.
8. **Minimal footprint**: Add the Protocol/interface definition and update the constructor only.
9. **Style matching**: Match existing code style — type hints, docstrings, import ordering.
10. **Diff summary**: After each modified file, show a plain-English summary of what changed.
11. **Test files**: Flag violations only if they cause real maintainability issues. Do not flag test classes that directly instantiate the system under test.
12. **Large files (>500 lines)**: Note that the fix may need to be applied incrementally.
