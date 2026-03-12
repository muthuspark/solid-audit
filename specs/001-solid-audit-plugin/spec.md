# Feature Specification: Solid Audit Claude Code Plugin

**Feature Branch**: `001-solid-audit-plugin`
**Created**: 2026-03-13
**Status**: Draft

## User Scenarios & Testing *(mandatory)*

### User Story 1 - On-Demand SOLID Audit (Priority: P1)

An engineer working on a feature wants to check their changed files for SOLID design principle violations before opening a PR. They run `/solid-audit` in their active Claude Code session and receive a structured, per-principle report without any files being modified.

**Why this priority**: This is the core, read-only entry point for the plugin. It delivers immediate value with zero risk, can be shared with a team before fix skills are ready, and is the foundation all other commands depend on.

**Independent Test**: Can be fully tested by running `/solid-audit` against a file with known SOLID violations and verifying the structured report is generated without any file changes.

**Acceptance Scenarios**:

1. **Given** a git repository with staged or unstaged changes, **When** the engineer runs `/solid-audit`, **Then** the plugin reviews all changed files and outputs a structured report listing every violation by principle (S/O/L/I/D), including file name, class/function name, one-sentence reason, and a concrete suggestion
2. **Given** `/solid-audit` is run on clean, well-structured files, **When** the audit completes, **Then** all reviewed files are listed explicitly with zero violations (no silent omissions)
3. **Given** a file path argument (e.g. `/solid-audit src/services/user.py`), **When** the audit runs, **Then** only that specific file is reviewed regardless of git diff state
4. **Given** no git repository is present, **When** `/solid-audit` is invoked, **Then** the plugin asks the user for file paths rather than failing silently

---

### User Story 2 - Single-Principle Fix (Priority: P2)

An engineer receives the audit report and sees SRP violations. They run `/srp` (or `/ocp`, `/lsp`, `/isp`, `/dip`) to fix only the violations for that one principle. The engineer is shown what will change, confirms, and the fix is applied with a diff summary.

**Why this priority**: Targeted, single-principle fixes allow engineers to iteratively address violations in manageable steps, reducing cognitive load and risk versus fixing everything at once.

**Independent Test**: Can be tested by running `/srp` on a file with a known SRP violation, confirming the fix, and verifying the extracted class/function is correct and the diff is accurate.

**Acceptance Scenarios**:

1. **Given** a file with a class that handles multiple unrelated responsibilities, **When** the engineer runs `/srp`, **Then** the plugin identifies the violation, proposes a fix, asks for confirmation, and applies only behavior-preserving changes if confirmed
2. **Given** the engineer declines confirmation, **When** the fix is proposed, **Then** no files are modified
3. **Given** a fix that the plugin cannot safely make behavior-preserving, **When** the analysis runs, **Then** the plugin flags the location and skips the change rather than applying an unsafe transformation
4. **Given** a fix is applied, **When** it completes, **Then** a diff summary is shown for every modified file

---

### User Story 3 - Full Fix Across All Principles (Priority: P3)

An engineer wants to address all SOLID violations in their changed files in one operation. They run `/solid-fix`, see violations grouped by principle, confirm once, and all safe fixes are applied.

**Why this priority**: Power users benefit from a single command that orchestrates all five fix skills, but it depends on all individual skills being stable first.

**Independent Test**: Can be tested by running `/solid-fix` on a file with known violations across multiple principles, confirming the grouped prompt, and verifying all safe fixes are applied.

**Acceptance Scenarios**:

1. **Given** files with violations across multiple SOLID principles, **When** `/solid-fix` is run, **Then** violations are grouped by principle and a single confirmation prompt is presented before any changes are written
2. **Given** the engineer confirms, **When** fixes are applied, **Then** all five principles' fixes are attempted and a per-principle diff summary is shown
3. **Given** some violations cannot be safely fixed, **When** `/solid-fix` runs, **Then** those locations are flagged and skipped without blocking the rest of the fix

---

### Edge Cases

- What happens when a file is over 500 lines? The audit runs but a note is added that fixes may need to be applied incrementally.
- How does the plugin handle mixed-language monorepos? Each file is audited using the idioms appropriate for its file extension (Python, TypeScript, Java, Go).
- What happens with intentional "violations" such as a thin data container God object? The plugin skips them and adds a note explaining why.
- How does the plugin handle test files? Violations are only flagged if they cause real maintainability issues.
- What if `git diff --cached` returns nothing? The plugin falls back to `git diff`, then asks the user for file paths if both are empty.
- What happens when lock files, generated code, or migration files appear in the diff? They are skipped by default.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The plugin MUST provide a `/solid-audit` command that reviews files for violations of all five SOLID principles and outputs a structured report without modifying any files
- **FR-002**: The audit report MUST include: list of files reviewed, total violation count, per-principle breakdown (S/O/L/I/D), and for every violation: file name, class/function name, one-sentence reason, and a concrete suggestion
- **FR-003**: Files with zero violations MUST be listed explicitly in the report — they must not be silently omitted
- **FR-004**: The plugin MUST provide individual fix commands `/srp`, `/ocp`, `/lsp`, `/isp`, and `/dip` that fix violations for the named principle only
- **FR-005**: The plugin MUST provide a `/solid-fix` command that orchestrates all five fix skills with a single grouped confirmation prompt
- **FR-006**: Every fix command MUST ask for confirmation before writing any changes to disk
- **FR-007**: All fixes MUST be behavior-preserving; if a fix cannot be confirmed as behavior-preserving, the plugin MUST flag and skip it
- **FR-008**: After applying any fix, the plugin MUST display a diff summary for each modified file
- **FR-009**: Fix commands MUST match the existing code style (type hints, docstrings, import ordering) of the file being modified
- **FR-010**: The plugin MUST auto-detect scope using `git diff --cached` first, then `git diff`, then prompt the user for file paths
- **FR-011**: The plugin MUST accept an optional file path argument to scope the audit or fix to a specific file
- **FR-012**: The plugin MUST skip lock files, generated code, migrations, and test fixtures by default
- **FR-013**: For files over 500 lines, the plugin MUST audit normally but note that fixes may need to be applied incrementally
- **FR-014**: The plugin MUST apply language-appropriate SOLID idioms per file extension (Python, TypeScript, Java, Go)
- **FR-015**: The plugin MUST be installable via the Claude Code marketplace using `plugin.json` and `marketplace.json` manifests

### Key Entities

- **Skill**: A `SKILL.md` file that defines one command's behavior — scope detection, audit logic, fix logic, confirmation flow, and output format. One skill per SOLID principle plus one for the full audit and one for the orchestrator.
- **Violation**: A detected SOLID design issue, identified by file, class/function, principle, reason, and suggestion.
- **Fix**: A behavior-preserving code transformation that resolves a violation, applied only after user confirmation.
- **Audit Report**: The structured output of `/solid-audit` — a per-file, per-principle breakdown of all violations found.
- **Plugin Manifest**: The `.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json` files that register the plugin with the Claude Code marketplace.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Engineers can run `/solid-audit` and receive a complete, actionable violation report in a single command without switching tools or context
- **SC-002**: Zero false positives on clean, well-structured files — the audit reports no violations on intentionally clean code samples
- **SC-003**: Every fix skill (`/srp` through `/dip`) preserves the public interface and runtime behavior of the modified code (verified by manual regression check)
- **SC-004**: The plugin installs successfully via the Claude Code marketplace install command on a fresh Claude Code session
- **SC-005**: Each individual fix skill can be used independently before all five skills are complete, enabling incremental team rollout
- **SC-006**: Engineers on the team can adopt the plugin without any training beyond reading the README install instructions and command reference
- **SC-007**: The audit correctly identifies violations across Python and TypeScript files using language-appropriate idiom suggestions

## Assumptions

- The plugin targets Claude Code as the host environment; no standalone CLI or CI integration is in scope for v1.0.
- Python and TypeScript are the primary languages for v1.0; Java and Go support is planned for v1.1.
- "Behavior-preserving" is verified by the plugin author through manual testing on real code samples — automated regression testing is not in scope for v1.0.
- The team uses git for version control; the git-based scope detection is the primary workflow.
- All seven skills (solid-audit, srp, ocp, lsp, isp, dip, solid-fix) are delivered as `SKILL.md` files under `.claude/skills/`.
