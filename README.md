# solid-audit

SOLID design auditor and fixer for Claude Code.

Review your changed files for SOLID design principle violations and apply targeted, behavior-preserving fixes — without leaving your Claude Code session.

## Install

```bash
/plugin marketplace add muthuspark/solid-audit
```

Run this once in any Claude Code session. The plugin is available immediately.

---

## Commands

| Command | Action | Modifies Files? | Confirmation? |
|---------|--------|-----------------|---------------|
| `/solid-audit` | Audit all 5 SOLID principles — structured report | No | No |
| `/srp` | Fix Single Responsibility violations | Yes | Yes |
| `/ocp` | Fix Open/Closed violations | Yes | Yes |
| `/lsp` | Fix Liskov Substitution violations | Yes | Yes |
| `/isp` | Fix Interface Segregation violations | Yes | Yes |
| `/dip` | Fix Dependency Inversion violations | Yes | Yes |
| `/solid-fix` | Fix all violations across all 5 principles | Yes | Yes |

---

## Typical Workflow

1. Write your feature code
2. Stage your changes: `git add -p`
3. Run `/solid-audit` — review the violation report
4. Run the relevant fix command(s) — review the before/after diff and confirm
5. Review the diff summaries
6. Open your PR

---

## Audit a Specific File

```
/solid-audit src/services/user.py
```

All fix commands also accept an optional file path:

```
/srp src/services/user.py
/solid-fix src/
```

---

## What the Audit Report Looks Like

```
## SOLID Audit Report

**Files reviewed**: 3
**Total violations**: 2

### S — Single Responsibility
- `src/services/user.py` · `UserService` — handles both user persistence and email notification → extract `EmailNotificationService` with a single `notify(user)` method

### O — Open/Closed
(none)

### L — Liskov Substitution
(none)

### I — Interface Segregation
(none)

### D — Dependency Inversion
- `src/services/order.py` · `OrderService` — directly instantiates `PostgresDatabase` in `__init__`, preventing dependency substitution → accept a `Database` Protocol parameter with a default value

---
**Files with no violations**: src/models/user.py
**Files skipped**: package-lock.json
```

---

## Fix Safety

Every fix command:

- Shows a **before/after diff** before writing anything
- Asks for **confirmation** — type `yes` to apply, `no` to skip
- **Skips** any fix it cannot apply as behavior-preserving (flags with reason)
- Shows a **diff summary** after each modified file
- Matches your existing code style (type hints, docstrings, import ordering)

---

## Supported Languages

| Language | Idioms Used |
|----------|-------------|
| Python | `Protocol`, `abc.ABC`, constructor injection, `@dataclass` |
| TypeScript | `interface`, `abstract class`, constructor injection |
| Java | `interface`, `abstract class`, `@Autowired` or constructor injection |
| Go | interface types, struct composition |

---

## What Gets Skipped Automatically

- `*.lock` files (`package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`)
- Database migrations (`**/migrations/**`)
- Generated code (`**/__generated__/**`)
- Test fixtures (`**/fixtures/**`)
- Minified files (`**/*.min.js`)

---

## File Layout

```
.claude/
└── skills/
    ├── solid-audit/
    │   ├── SKILL.md
    │   └── references/
    │       └── examples.md
    ├── srp/SKILL.md
    ├── ocp/SKILL.md
    ├── lsp/SKILL.md
    ├── isp/SKILL.md
    ├── dip/SKILL.md
    └── solid-fix/SKILL.md
```

---

## Author

Muthukrishnan · [github.com/muthuspark/solid-audit](https://github.com/muthuspark/solid-audit)
