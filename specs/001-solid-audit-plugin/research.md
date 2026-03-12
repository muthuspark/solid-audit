# Research: Solid Audit Claude Code Plugin

**Branch**: `001-solid-audit-plugin` | **Phase**: 0 | **Date**: 2026-03-13

## 1. SKILL.md Format for Claude Code Marketplace Plugins

**Decision**: Each skill is a Markdown file (`SKILL.md`) placed in its own named directory under `.claude/skills/`. The file begins with a YAML frontmatter block containing a `description` field, followed by prose sections that instruct Claude how to behave when the skill is invoked.

**Rationale**: This matches the pattern established by Claude Code marketplace plugins and the path declared in `plugin.json` (`"./.claude/skills"`). The `description` field is used by Claude Code to match the slash command to the correct skill. Prose instructions (not code) drive Claude's behavior — the skill is a prompt template, not executable code.

**Alternatives considered**:
- Single large `SKILL.md` covering all principles — rejected because it would exceed 500 lines, making iteration harder and loading unnecessary content for single-principle commands.
- Executable scripts — rejected because the plugin target is Claude Code's skill system, not a standalone CLI.

---

## 2. SKILL.md Internal Structure

**Decision**: Each SKILL.md follows this section structure:

```markdown
---
description: [One-line description matching the slash command]
---

## Trigger
[When this skill activates and what command invokes it]

## Scope Detection
[How to determine which files to analyze]

## [Audit / Fix] Logic
[Step-by-step instructions for Claude to follow]

## Output Format
[Exact structure of the response the user sees]

## Rules
[Constraints: safety, style matching, confirmation flow]
```

**Rationale**: Consistent structure across all seven skills reduces cognitive load, makes testing predictable, and allows the `solid-fix` orchestrator to compose the other five skills reliably.

---

## 3. Scope Detection Strategy

**Decision**: Use a three-stage fallback:
1. `git diff --cached` (staged files)
2. `git diff` (unstaged changes against HEAD)
3. Ask the user for explicit file paths (no-git fallback)

Files matching these patterns are always excluded: `*.lock`, `package-lock.json`, `yarn.lock`, `**/migrations/**`, `**/__generated__/**`, `**/fixtures/**`.

**Rationale**: Staged files are the most intent-aligned — the engineer is about to commit them. Falling back to all unstaged changes gives useful results even when nothing is staged. Asking the user avoids silent failure in non-git environments.

---

## 4. Audit Output Format

**Decision**: Structured Markdown report with these sections:

```
## SOLID Audit Report

**Files reviewed**: N
**Total violations**: N

### S — Single Responsibility
- `path/to/file.py` · `ClassName` — [one-sentence reason] → [concrete suggestion]

### O — Open/Closed
(none)

### L — Liskov Substitution
...

### I — Interface Segregation
...

### D — Dependency Inversion
...

---
Files with no violations: file1.py, file2.ts
```

**Rationale**: Per-principle grouping matches the slash command structure (each command targets one letter). Explicit "no violations" section eliminates ambiguity. File + class/function + reason + suggestion gives actionable, scannable output.

---

## 5. Fix Safety Protocol

**Decision**: Before any file modification:
1. Claude proposes the fix with a before/after diff preview
2. User must explicitly confirm ("yes", "y", or "proceed")
3. If a fix cannot be confirmed as behavior-preserving, Claude flags it with `[SKIP — may affect behavior]` and skips it
4. After each file is modified, Claude shows a diff summary

**Rationale**: Fix skills modify source code. A single unsafe transformation could break production code. The confirmation gate and skip-on-uncertainty policy prioritize safety over completeness.

---

## 6. Language-Specific SOLID Idioms

**Decision**:

| Language | SRP | OCP | LSP | ISP | DIP |
|----------|-----|-----|-----|-----|-----|
| Python | Extract class/function | `Protocol` + registry | Flatten hierarchy or use composition | Split `Protocol`/`abc.ABC` | Constructor injection, `Protocol` for deps |
| TypeScript | Extract class/module | `interface` + registry/factory | Flatten or use composition | Split `interface` | Constructor injection, `interface` for deps |
| Java | Extract class | `interface` + factory | Flatten or composition | Split `interface` | Constructor + `@Autowired` or manual injection |
| Go | Extract struct/func | `interface` + registration | Flatten struct embedding | Narrow `interface` | Accept `interface` in constructor |

**Rationale**: Language-appropriate idioms ensure fixes are idiomatic and accepted by the team. Using `Protocol` for Python DIP (rather than forcing `abc.ABC`) reflects modern Python 3.8+ practice. TypeScript `interface` is preferred over `abstract class` for flexibility.

**Alternatives considered**: A single generic fix pattern for all languages — rejected because it produces non-idiomatic code (e.g., Java-style abstract classes in Python).

---

## 7. solid-fix Orchestration Strategy

**Decision**: `solid-fix` runs the five principle detections sequentially (S→O→L→I→D), collects all violations, groups them by principle in a single confirmation prompt, then applies all confirmed fixes in a second pass.

**Rationale**: A single confirmation prompt reduces friction. Sequential detection (not parallel) avoids conflicting fixes — e.g., an SRP extraction might change the class structure that an OCP fix would also modify.

---

## 8. examples.md Content Strategy

**Decision**: The `solid-audit/references/examples.md` file contains before/after code samples for Python and TypeScript, one pair per principle (5 pairs total). Each example is minimal (10–30 lines) and illustrates only the targeted violation.

**Rationale**: Concrete examples ground Claude's violation detection in real patterns, reducing false positives and missed violations. Keeping examples minimal ensures the reference file loads quickly and stays under 200 lines.
