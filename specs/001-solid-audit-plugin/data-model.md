# Data Model: Solid Audit Claude Code Plugin

**Branch**: `001-solid-audit-plugin` | **Phase**: 1 | **Date**: 2026-03-13

---

> This plugin has no persistent data storage. All entities below are **in-context structures** — they exist within a single Claude Code session and are expressed as structured Markdown in Claude's responses.

---

## Entity: Violation

The atomic unit of audit output. One Violation is produced per detected SOLID issue.

| Field | Type | Description | Constraints |
|-------|------|-------------|-------------|
| `file` | string | Relative path to the source file | Required |
| `symbol` | string | Class or function name where violation occurs | Required |
| `principle` | enum | One of: S, O, L, I, D | Required |
| `reason` | string | One sentence explaining why this is a violation | Required, max 1 sentence |
| `suggestion` | string | Concrete, actionable fix recommendation | Required |
| `safe_to_fix` | boolean | Whether a behavior-preserving fix can be applied automatically | Required |

**Validation rules**:
- `reason` must reference the specific violation pattern, not just name the principle
- `suggestion` must be language-appropriate for the file's extension
- `safe_to_fix = false` is set when the fix requires understanding callers outside the reviewed scope

---

## Entity: AuditReport

The structured output of a `/solid-audit` run. Contains zero or more Violations.

| Field | Type | Description | Constraints |
|-------|------|-------------|-------------|
| `files_reviewed` | list[string] | All files that were analyzed | Must be non-empty |
| `files_clean` | list[string] | Files with zero violations | Subset of `files_reviewed` |
| `violations` | list[Violation] | All detected violations | May be empty |
| `total_count` | integer | Total number of violations | Derived from `violations` length |
| `by_principle` | map[S/O/L/I/D → list[Violation]] | Violations grouped by principle | Derived |
| `scope_method` | enum | How scope was determined: `git-staged`, `git-unstaged`, `user-provided`, `argument` | Required |
| `skipped_files` | list[string] | Files excluded (lock files, generated, etc.) | May be empty |

**State transitions**:
- `scope_method = git-staged` → when `git diff --cached` has output
- `scope_method = git-unstaged` → when staged is empty but `git diff` has output
- `scope_method = user-provided` → when no git diff available
- `scope_method = argument` → when a file path was passed directly to the command

---

## Entity: Fix

A proposed code transformation for a single Violation.

| Field | Type | Description | Constraints |
|-------|------|-------------|-------------|
| `violation` | Violation | The violation this fix addresses | Required |
| `before` | string | Code snippet showing current state | Required |
| `after` | string | Code snippet showing proposed change | Required |
| `diff_summary` | string | Human-readable description of what changed | Required |
| `confirmed` | boolean | Whether the user approved the fix | Defaults to false |
| `applied` | boolean | Whether the fix was written to disk | Defaults to false |
| `skip_reason` | string | Reason fix was skipped (if not applied) | Required if `applied = false` and `confirmed = true` |

**State transitions**:
- `confirmed = false` → Fix proposed, awaiting user response
- `confirmed = true, applied = false` → User confirmed, fix in progress or skipped due to safety
- `confirmed = true, applied = true` → Fix successfully written to disk
- `confirmed = false, applied = false` → User declined or timed out

---

## Entity: FixSession

The output of a single fix command run (`/srp`, `/ocp`, etc., or `/solid-fix`).

| Field | Type | Description | Constraints |
|-------|------|-------------|-------------|
| `command` | string | The slash command that triggered this session | Required |
| `principle` | enum or "ALL" | Which principle(s) this session targets | Required |
| `fixes` | list[Fix] | All proposed fixes in this session | May be empty |
| `applied_count` | integer | Number of fixes actually written | Derived |
| `skipped_count` | integer | Number of fixes skipped (safety or declined) | Derived |
| `files_modified` | list[string] | Files that were changed | Derived |

---

## Entity: Skill

Represents a single `SKILL.md` file. Not a runtime entity — this describes the file structure.

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Directory name under `.claude/skills/` (e.g., `solid-audit`, `srp`) |
| `command` | string | The slash command that invokes this skill (e.g., `/solid-audit`, `/srp`) |
| `modifies_files` | boolean | Whether this skill writes changes to disk |
| `requires_confirmation` | boolean | Whether this skill requires user confirmation before acting |
| `path` | string | Path to the SKILL.md file |

| name | command | modifies_files | requires_confirmation |
|------|---------|---------------|----------------------|
| `solid-audit` | `/solid-audit` | false | false |
| `srp` | `/srp` | true | true |
| `ocp` | `/ocp` | true | true |
| `lsp` | `/lsp` | true | true |
| `isp` | `/isp` | true | true |
| `dip` | `/dip` | true | true |
| `solid-fix` | `/solid-fix` | true | true |

---

## Entity: PluginManifest

The two files in `.claude-plugin/` that register the plugin with the Claude Code marketplace.

| Field | Source File | Description |
|-------|-------------|-------------|
| `name` | plugin.json | Plugin identifier: `"solid-audit"` |
| `version` | plugin.json + marketplace.json | Must be identical in both files |
| `skills` | plugin.json | Relative path to skills directory: `"./.claude/skills"` |
| `$schema` | marketplace.json | Validates against Anthropic's marketplace schema |
| `category` | marketplace.json | `"code-quality"` |
| `tags` | marketplace.json | `["solid", "design-patterns", "refactoring", "code-review"]` |

**Validation rule**: `version` in `plugin.json` and `version` in `marketplace.json` (inside the `plugins` array) MUST always match. Both must be updated atomically when releasing.
