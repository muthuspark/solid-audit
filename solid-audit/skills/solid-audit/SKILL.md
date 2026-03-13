---
name: solid-audit
description: Audit changed files for SOLID design principle violations and output a structured per-principle report.
---

## Trigger

This skill activates when the user runs `/solid-audit` (optionally with a file path argument, e.g. `/solid-audit src/services/user.py`).

## Scope Detection

Determine which files to analyze using this priority order:

1. **Argument** — If the user provided a file or directory path after `/solid-audit`, use that directly.
2. **Git staged** — Run `git diff --cached --name-only`. If output is non-empty, use those files.
3. **Git unstaged** — Run `git diff --name-only`. If output is non-empty, use those files.
4. **Ask user** — If no git diff is available (no git repo or empty diff), ask: "No changed files detected. Please provide a file or directory path to audit."

**Always skip these files regardless of scope:**
- `*.lock`, `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`, `Gemfile.lock`, `composer.lock`
- Files matching `**/migrations/**`
- Files matching `**/__generated__/**`
- Files matching `**/fixtures/**`
- Files matching `**/*.min.js`

**Note files >500 lines:** Audit normally but append: `⚠️ Large file (>500 lines) — apply any fixes incrementally.`

## Audit Logic

For each file in scope, read its full content and check for each of the five SOLID principles below. Use the language-appropriate idiom patterns based on the file extension.

### S — Single Responsibility Principle

**Violation pattern:** A class or function that handles two or more unrelated concerns. Look for:
- A class with method groups that belong to distinct responsibilities (e.g., data parsing AND email sending AND logging in one class)
- A function that both validates input AND persists data AND formats output
- A class whose name contains "And", "Manager", or "Handler" and handles multiple unrelated operations
- A class that imports from two or more unrelated domains (e.g., imports both an SMTP library and a database driver)

**Language idioms:**
- Python: each class should have one reason to change; use `dataclass` for value objects
- TypeScript: each class/module should have one reason to change
- Java: same principle; extract to new class
- Go: separate concerns into distinct structs or functions
- C#: extract to new `class`; use `record` for value objects
- Kotlin: extract to new `class`; use `data class` for value objects
- Ruby: extract to new class; use modules for shared behavior
- PHP: extract to new class; use traits for shared behavior

### O — Open/Closed Principle

**Violation pattern:** Code that must be modified to extend behavior. Look for:
- `if type == "x": ... elif type == "y": ...` (switch-on-type strings) or equivalent `switch`/`case` statements in Java, C#, PHP
- `when (type) { "x" -> ... "y" -> ... }` in Kotlin
- `isinstance` chains used for dispatch (e.g., `if isinstance(obj, TypeA): ... elif isinstance(obj, TypeB): ...`)
- Direct subclass instantiation chosen by condition

**Language idioms:**
- Python: `Protocol` + registry dict (`handlers: dict[str, Handler] = {}`)
- TypeScript: `interface` + factory/registry
- Java: `interface` + factory pattern
- Go: `interface` + registration map
- C#: `interface` + `Dictionary<string, IHandler>` registry
- Kotlin: `interface` + `mapOf<String, Handler>()` registry; also consider sealed classes with `when`
- Ruby: hash-based dispatch (`HANDLERS = { "x" => XHandler.new }.freeze`)
- PHP: `interface` + array registry

### L — Liskov Substitution Principle

**Violation pattern:** A subclass that cannot be used in place of its base class. Look for:
- `raise NotImplementedError` (Python/Ruby) in an overriding method
- `throw new Error("not implemented")` (TypeScript) in an overriding method
- `throw new UnsupportedOperationException()` (Java) in an overriding method
- `throw UnsupportedOperationException()` or `TODO()` (Kotlin) used as a stub in an overriding method
- `throw new NotImplementedException()` (C#) in an overriding method
- `throw new \BadMethodCallException("not implemented")` (PHP) in an overriding method
- A subclass method with an empty body (`pass` in Python, `{}` in TS/Java/C#/Go) that overrides meaningful base-class behavior (silent no-op)
- A subclass method that rejects inputs the base class accepts (narrowed preconditions — e.g., base accepts any `int`, subclass only accepts `int > 0`)
- A subclass method that returns a more restricted type or raises exceptions not declared in the base contract

**Language idioms:**
- Python: flatten hierarchy or replace inheritance with composition
- TypeScript/Java: same — prefer composition over inheritance when LSP is violated
- Go: struct embedding replaced with interface acceptance
- C#: prefer composition; `NotImplementedException` in override signals hierarchy mismatch
- Kotlin: `UnsupportedOperationException()` or `TODO()` in override signals hierarchy mismatch
- Ruby: use duck typing with focused modules instead of deep inheritance
- PHP: prefer composition; `BadMethodCallException` in override signals hierarchy mismatch

### I — Interface Segregation Principle

**Violation pattern:** A fat interface that forces implementors to provide methods they don't use. Look for:
- `abc.ABC` or `Protocol` (Python), `interface` (TS/Java/C#/Kotlin/PHP), or a named `interface` type (Go) with 6+ abstract methods where a concrete implementor stubs ≥ half with `pass`, `raise NotImplementedError`, `throw new Error`, `throw new UnsupportedOperationException()`, `throw new NotImplementedException()`, or `TODO()`
- A single interface mixing logically distinct responsibilities (e.g., `print()`, `scan()`, and `send_email()` in one interface)

**Identifying a "thin data container":** Skip if the type has no behavior — only fields, getters/setters, or a constructor. Do not skip if it has 2+ distinct behavioral method groups.

**Skip** if the class is clearly a thin data container or value object (not a fat interface violation).

**Language idioms:**
- Python: split `abc.ABC` or `Protocol` into narrow role interfaces
- TypeScript/Java: split `interface`
- Go: narrow the `interface` type to what callers actually need
- C#: split `interface`; C# supports multiple interface implementation
- Kotlin: split `interface`; Kotlin interfaces can have default method implementations
- Ruby: split into smaller focused modules; use `include` selectively
- PHP: split `interface`; PHP supports multiple interface implementation

### D — Dependency Inversion Principle

**Violation pattern:** A high-level module depending directly on a concrete low-level implementation. Look for:
- Concrete class instantiation inside `__init__` (Python), constructor (TS/Java/C#/Kotlin), or `initialize` (Ruby) — e.g., `self.db = DatabaseClient()`, `this.db = new DatabaseClient()`
- Hardcoded `import` of a concrete class used directly with no abstraction layer
- Module-level singleton dependency fetched without injection — e.g., `DB = PostgresDatabase()` at module level

**Do not flag:**
- Parameters that already accept an interface/Protocol type
- Standard library classes that are universally stable (e.g., `logging.Logger`, `pathlib.Path`, `fmt.Println`, `Console`)
- Constructors that already accept the concrete type as an injected parameter (even if untyped)

**Language idioms:**
- Python: accept a `Protocol` type in `__init__`, remove internal instantiation. Preserve defaults where possible: `def __init__(self, db: DbProtocol = DatabaseClient())`
- TypeScript: accept interface type in constructor
- Java: accept interface in constructor; note `@Autowired` for Spring users
- Go: accept interface in constructor/function
- C#: accept interface in constructor; ASP.NET Core users register in `IServiceCollection`
- Kotlin: accept interface in constructor; Android users use Hilt/Koin for injection
- Ruby: inject via constructor keyword argument with a default: `def initialize(db: PostgresDatabase.new)`
- PHP: accept interface in constructor; Symfony/Laravel DI containers resolve automatically

## Output Format

After analyzing all files, output the following structured report:

```
## SOLID Audit Report

**Files reviewed**: {N}
**Total violations**: {N}

### S — Single Responsibility
{list of violations, or "(none)"}

### O — Open/Closed
{list of violations, or "(none)"}

### L — Liskov Substitution
{list of violations, or "(none)"}

### I — Interface Segregation
{list of violations, or "(none)"}

### D — Dependency Inversion
{list of violations, or "(none)"}

---
**Files with no violations**: {comma-separated list, or "none"}
**Files skipped**: {comma-separated list, or "none"}
```

**Violation format** (one line per violation):
```
- `path/to/file.ext` · `SymbolName` — {one sentence: why this violates the principle} → {concrete, language-appropriate suggestion}
```

Example:
```
- `src/services/user.py` · `UserService` — handles both user authentication and email notification, violating single responsibility → extract `EmailNotificationService` with a single `notify(user)` method
```

## Rules

1. **Never modify any file** — this is a read-only audit command.
2. **Always list all reviewed files explicitly**, including those with zero violations. Silence is ambiguous.
3. **Apply language-appropriate idiom suggestions** based on file extension:
   - `.py` → Python idioms (Protocol, abc.ABC, dataclass)
   - `.ts`, `.tsx` → TypeScript idioms (interface, abstract class)
   - `.java` → Java idioms (interface, abstract class, @Autowired)
   - `.go` → Go idioms (interface types, struct composition)
   - `.cs` → C# idioms (interface, abstract class, IServiceCollection)
   - `.kt` → Kotlin idioms (interface, data class, sealed class)
   - `.rb` → Ruby idioms (modules, duck typing, keyword arguments)
   - `.php` → PHP idioms (interface, abstract class, DI container)
4. **Skip intentional patterns**: If a class is clearly a thin data container, configuration holder, or value object, skip it and note: `{SymbolName} — skipped (appears to be a data container)`
5. **Test files**: Flag violations only if they cause real maintainability issues, not just structural purity.
6. **Large files (>500 lines)**: Audit normally, but add the large-file warning at the end of that file's section.
7. **Mixed-language monorepos**: Apply the correct idiom per file — never apply Python idioms to TypeScript files.
8. **No git repo / empty diff**: Ask the user for file paths before proceeding — never fail silently.
