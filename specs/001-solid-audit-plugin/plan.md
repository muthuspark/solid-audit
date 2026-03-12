# Implementation Plan: Solid Audit Claude Code Plugin

**Branch**: `001-solid-audit-plugin` | **Date**: 2026-03-13 | **Spec**: [spec.md](spec.md)

## Summary

Build a Claude Code marketplace plugin that reviews changed source code for SOLID design principle violations and applies targeted, behavior-preserving fixes. The plugin delivers seven slash commands (`/solid-audit`, `/srp`, `/ocp`, `/lsp`, `/isp`, `/dip`, `/solid-fix`) implemented as `SKILL.md` files under `.claude/skills/`. No traditional programming language code is written — the deliverables are structured Markdown prompt-instructions that drive Claude's behavior in a Claude Code session.

## Technical Context

**Language/Version**: Markdown (SKILL.md instruction files) — no compiled or interpreted language
**Primary Dependencies**: Claude Code plugin system (`.claude-plugin/` manifests), Claude Code marketplace
**Storage**: File system — seven `SKILL.md` files under `.claude/skills/`, one `examples.md` reference file
**Testing**: Manual — run each slash command in a Claude Code session against prepared test files
**Target Platform**: Claude Code (cross-platform: Linux, macOS, Windows)
**Project Type**: Claude Code marketplace plugin
**Performance Goals**: Not applicable — skills are static Markdown files loaded on invocation
**Constraints**: Each `SKILL.md` must stay under 500 lines; `version` in `plugin.json` and `marketplace.json` must always match
**Scale/Scope**: 7 skill files, 2 manifest files, 1 reference file, 1 README

## Constitution Check

*No project constitution has been defined (constitution.md contains only the template placeholder). No gates to evaluate.*

## Project Structure

### Documentation (this feature)

```text
specs/001-solid-audit-plugin/
├── plan.md              # This file
├── research.md          # Phase 0 — decisions and rationale
├── data-model.md        # Phase 1 — in-context entities
├── quickstart.md        # Phase 1 — install and usage guide
├── contracts/
│   └── skill-commands.md  # Phase 1 — slash command contracts
└── tasks.md             # Phase 2 output (/speckit.tasks — NOT created here)
```

### Source Code (repository root)

```text
.claude-plugin/
├── plugin.json          # Plugin identity and skills path (already exists)
└── marketplace.json     # Marketplace listing and schema (already exists)

.claude/
└── skills/
    ├── solid-audit/
    │   ├── SKILL.md                  # Audit logic — all 5 principles, output format
    │   └── references/
    │       └── examples.md           # Before/after code samples per principle
    ├── srp/
    │   └── SKILL.md                  # SRP fix skill
    ├── ocp/
    │   └── SKILL.md                  # OCP fix skill
    ├── lsp/
    │   └── SKILL.md                  # LSP fix skill
    ├── isp/
    │   └── SKILL.md                  # ISP fix skill
    ├── dip/
    │   └── SKILL.md                  # DIP fix skill
    └── solid-fix/
        └── SKILL.md                  # Orchestrator — all five fix skills

README.md                             # Install command + command reference
```

**Structure Decision**: Plugin-only structure — no `src/` or `tests/` directories. All deliverables are Markdown files. The `.claude-plugin/` manifests already exist from the initial commit and require no structural changes.

## Phase 0: Research Findings

All research consolidated in [research.md](research.md). No NEEDS CLARIFICATION items remain.

**Key decisions resolved:**
1. SKILL.md format: YAML frontmatter (`description`) + prose sections — matches Claude Code plugin conventions
2. Scope detection: `git diff --cached` → `git diff` → ask user
3. Audit output: per-principle grouped report with explicit "no violations" listing
4. Fix safety: confirmation gate + skip-on-uncertainty policy
5. Fix execution order for `/solid-fix`: S → O → L → I → D (avoids conflicting transformations)
6. Language idioms: documented per-language in research.md Section 6

## Phase 1: Design Artifacts

- [data-model.md](data-model.md) — in-context entities: Violation, AuditReport, Fix, FixSession, Skill, PluginManifest
- [contracts/skill-commands.md](contracts/skill-commands.md) — full input/output/behavior contract for all seven slash commands
- [quickstart.md](quickstart.md) — install instructions and typical engineer workflow

## Implementation Sequence

The skills must be built in this order to enable incremental team adoption:

### Stage 1 — Audit Skill (read-only, zero risk)
1. `solid-audit/SKILL.md` — core audit logic
2. `solid-audit/references/examples.md` — before/after samples for all five principles

**Gate**: Run `/solid-audit` on a Python file with known SRP violation → correct violation flagged. Run on clean file → zero violations reported.

### Stage 2 — Single-Principle Fix Skills
3. `srp/SKILL.md`
4. `ocp/SKILL.md`
5. `lsp/SKILL.md`
6. `isp/SKILL.md`
7. `dip/SKILL.md`

**Gate per skill**: Run fix command on a prepared test file → correct violation identified, fix proposed with diff, confirmation requested, fix applied behavior-preservingly.

### Stage 3 — Orchestrator
8. `solid-fix/SKILL.md`

**Gate**: Run `/solid-fix` on a file with 3+ violations across different principles → grouped confirmation shown, all safe fixes applied, per-principle diff summary output.

### Stage 4 — Polish
9. `README.md` — install command, all slash commands, usage examples
10. Version tag `v1.0.0` — update both manifest files, create git tag

## SKILL.md Content Requirements

Each SKILL.md must include these sections (from research.md Section 2):

```
---
description: [one-line matching the slash command]
---

## Trigger
## Scope Detection
## [Audit / Fix] Logic
## Output Format
## Rules
```

**All fix skills additionally require:**
- Confirmation prompt pattern (from contracts/skill-commands.md)
- Skip-on-uncertainty rule
- Style-matching rule (type hints, imports, docstrings)
- Diff summary after each modified file

## Test Cases

From spec.md testing strategy:

| Test File | Violation to introduce | Command to test | Pass criteria |
|-----------|----------------------|-----------------|---------------|
| Python class with two unrelated responsibilities | SRP | `/solid-audit` + `/srp` | Flags correct class, fix extracts correctly |
| TypeScript with `switch` on type string | OCP | `/solid-audit` + `/ocp` | Flags switch, suggests registry pattern |
| Python subclass throwing `NotImplementedError` | LSP | `/solid-audit` + `/lsp` | Flags subclass, proposes composition |
| Python ABC with 8+ methods, half unused | ISP | `/solid-audit` + `/isp` | Flags ABC, suggests split |
| Python class instantiating DB client in `__init__` | DIP | `/solid-audit` + `/dip` | Flags hardcoded dep, suggests injection |
| Clean, well-structured file | None | `/solid-audit` | Zero violations, file listed explicitly |
| File with 3 violations across different principles | S + O + D | `/solid-fix` | Grouped confirmation, all safe fixes applied |
