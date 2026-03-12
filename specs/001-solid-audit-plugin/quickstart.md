# Quickstart: Solid Audit Claude Code Plugin

**Branch**: `001-solid-audit-plugin` | **Phase**: 1 | **Date**: 2026-03-13

---

## Install

```bash
/plugin marketplace add muthuspark/solid-audit
```

Run this once in any Claude Code session. The plugin is available immediately.

---

## Audit your current changes

```
/solid-audit
```

Runs against your staged files (`git diff --cached`), falling back to all unstaged changes (`git diff`).

**Audit a specific file:**
```
/solid-audit src/services/user.py
```

---

## Fix violations

After reviewing the audit report, run the corresponding fix command:

| Violation | Command |
|-----------|---------|
| Single Responsibility | `/srp` |
| Open/Closed | `/ocp` |
| Liskov Substitution | `/lsp` |
| Interface Segregation | `/isp` |
| Dependency Inversion | `/dip` |
| All violations | `/solid-fix` |

Each fix command will show you a before/after preview and ask for confirmation before writing any changes.

---

## Typical workflow

1. Write your feature code
2. Stage your changes: `git add -p`
3. Run `/solid-audit` — review the report
4. Run the relevant fix command(s) — confirm each fix
5. Review the diff summaries
6. Open your PR

---

## What gets skipped automatically

- `*.lock` files, `package-lock.json`, `yarn.lock`
- Database migrations (`**/migrations/**`)
- Generated code (`**/__generated__/**`)
- Test fixtures (`**/fixtures/**`)

---

## Supported languages

| Language | Idioms used |
|----------|-------------|
| Python | `Protocol`, `abc.ABC`, constructor injection, dataclass |
| TypeScript | `interface`, `abstract class`, constructor injection |
| Java | `interface`, `abstract class`, `@Autowired` or manual injection |
| Go | interface types, struct composition |

---

## File layout after install

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
