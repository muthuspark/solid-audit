# Tasks: Solid Audit Claude Code Plugin

**Input**: Design documents from `/specs/001-solid-audit-plugin/`
**Prerequisites**: plan.md ✓, spec.md ✓, research.md ✓, data-model.md ✓, contracts/ ✓

**Tests**: No automated tests — this is a Markdown-only Claude Code plugin. Verification is manual (run each slash command in a Claude Code session against prepared test files, as defined in plan.md "Test Cases" table).

**Organization**: Tasks are grouped by user story to enable independent delivery of each skill.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no shared state)
- **[Story]**: Which user story this task belongs to (US1, US2, US3)

---

## Phase 1: Setup

**Purpose**: Create the `.claude/skills/` directory structure required by `plugin.json`

- [ ] T001 Create `.claude/skills/` directory tree: `solid-audit/references/`, `srp/`, `ocp/`, `lsp/`, `isp/`, `dip/`, `solid-fix/` — place an empty `SKILL.md` placeholder in each skill folder

---

## Phase 2: Foundational

**Purpose**: Confirm the existing plugin manifests are consistent before writing any skill content

**⚠️ CRITICAL**: Plugin must be structurally valid before skills can be tested

- [ ] T002 Verify `.claude-plugin/plugin.json` `skills` path is `"./.claude/skills"` and `version` matches the `version` field in `.claude-plugin/marketplace.json` — fix any mismatch before proceeding

**Checkpoint**: Plugin manifests are consistent. Skill content can now be written and tested.

---

## Phase 3: User Story 1 — On-Demand SOLID Audit (Priority: P1) 🎯 MVP

**Goal**: Deliver a working `/solid-audit` command that reviews changed files for all five SOLID principles and outputs a structured, per-principle violation report — without modifying any files.

**Independent Test**: Run `/solid-audit` on a Python file containing a class with two unrelated responsibilities → report shows the correct class flagged under "S — Single Responsibility" with reason and suggestion. Run on a clean file → all reviewed files listed explicitly with zero violations.

### Implementation

- [ ] T003 [US1] Write `.claude/skills/solid-audit/SKILL.md` — include: YAML frontmatter (`description: Audit changed files for SOLID design principle violations`), Trigger section, Scope Detection (git-staged → git-unstaged → user-provided fallback, skip lock/generated/migration/fixture files), Audit Logic for each principle (SRP: multiple responsibilities; OCP: switch-on-type / direct subclass instantiation; LSP: `NotImplementedError` in subclasses / narrowed preconditions; ISP: fat ABCs with unused methods; DIP: concrete instantiation in constructors), Output Format (structured report per contracts/skill-commands.md: files reviewed count, total violations, per-principle grouped violations with file+symbol+reason+suggestion, explicit "no violations" list, skipped files list), Rules (never modify files, language-appropriate idioms per file extension, note files >500 lines, skip intentional patterns like thin data containers)
- [ ] T004 [P] [US1] Write `.claude/skills/solid-audit/references/examples.md` — 5 before/after code samples (one per principle) for Python and TypeScript: SRP (class with two responsibilities → two focused classes), OCP (switch-on-type → Protocol+registry), LSP (subclass throws `NotImplementedError` → composition), ISP (fat ABC with 8 methods → two narrow ABCs), DIP (class instantiates DB in `__init__` → constructor injection with Protocol)

**Checkpoint**: `/solid-audit` is fully functional and safe to share with the team. All five principles detected. Stop here for MVP.

---

## Phase 4: User Story 2 — Single-Principle Fix Skills (Priority: P2)

**Goal**: Deliver five fix commands (`/srp`, `/ocp`, `/lsp`, `/isp`, `/dip`) — one per SOLID principle. Each proposes a behavior-preserving fix, asks for confirmation, applies it, and shows a diff summary.

**Independent Test**: Run `/srp` on a Python file with a class holding two unrelated responsibilities → plugin identifies the class, shows before/after diff, asks for confirmation, applies fix, shows diff summary. Decline confirmation → no files modified.

### Implementation

All five skills share the same structure (research.md Section 2) and can be written in parallel.

- [ ] T005 [P] [US2] Write `.claude/skills/srp/SKILL.md` — description: `Fix Single Responsibility Principle violations`; Trigger, Scope Detection (same as solid-audit); SRP Fix Logic (detect classes/functions with multiple responsibilities, extract to separate focused class/function, preserve public interface); Confirmation Flow (show before/after snippet, diff summary, ask "Proceed with fix? (yes/no)", skip if declined); Fix Rules (behavior-preserving only, skip-on-uncertainty with `[SKIP — may affect behavior]` flag, match existing style: type hints/docstrings/imports, show diff summary per modified file); Language patterns: Python (extract class, use dataclass for value objects), TypeScript (extract class/module)
- [ ] T006 [P] [US2] Write `.claude/skills/ocp/SKILL.md` — description: `Fix Open/Closed Principle violations`; same structure as srp/SKILL.md; OCP Fix Logic (detect switch-on-type strings or direct subclass instantiation, propose Protocol+registry pattern for Python / interface+factory for TypeScript/Java / interface registration for Go); include note on when OCP refactor is too invasive to apply safely
- [ ] T007 [P] [US2] Write `.claude/skills/lsp/SKILL.md` — description: `Fix Liskov Substitution Principle violations`; LSP Fix Logic (detect `NotImplementedError` in subclasses / narrowed preconditions / strengthened postconditions, propose flattening hierarchy or replacing inheritance with composition); flag cases where callers outside the reviewed scope must be updated
- [ ] T008 [P] [US2] Write `.claude/skills/isp/SKILL.md` — description: `Fix Interface Segregation Principle violations`; ISP Fix Logic (detect fat ABC/Protocol/interface with 8+ methods where implementors stub half, propose splitting into two or more narrow ABCs/Protocols/interfaces by responsibility); skip if the class is a thin data container
- [ ] T009 [P] [US2] Write `.claude/skills/dip/SKILL.md` — description: `Fix Dependency Inversion Principle violations`; DIP Fix Logic (detect concrete class instantiation in `__init__` / hardcoded `import` inside methods, propose constructor injection with a Protocol/interface type for Python/TypeScript, `@Autowired` or manual injection for Java, interface acceptance for Go); preserve existing default values where possible to minimize call-site changes

**Checkpoint**: All five single-principle fix commands work independently. Team can use `/srp` through `/dip` before `/solid-fix` is ready.

---

## Phase 5: User Story 3 — Full Fix Orchestrator (Priority: P3)

**Goal**: Deliver `/solid-fix` — a single command that runs all five principle analyses, presents a grouped confirmation prompt, and applies all safe fixes in order (S → O → L → I → D).

**Independent Test**: Run `/solid-fix` on a file with violations across three different principles → see all violations grouped by principle in one confirmation prompt → confirm → all safe fixes applied → per-principle diff summary shown → unsafe violations flagged with skip reason.

### Implementation

- [ ] T010 [US3] Write `.claude/skills/solid-fix/SKILL.md` — description: `Fix all SOLID principle violations across all five principles`; Trigger, Scope Detection (same as solid-audit); Orchestration Logic (run S/O/L/I/D analyses sequentially to collect all violations, group by principle for the confirmation prompt, present single grouped confirmation per contracts/skill-commands.md Phase 1 Proposal format); Fix Application (apply confirmed fixes in order S → O → L → I → D to avoid conflicting transformations, skip any violation flagged as unsafe); Output (per-principle diff summary after all fixes applied, total count: N files modified, M violations skipped)

**Checkpoint**: All three user stories complete. Full plugin is functional.

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: Documentation and release tagging

- [ ] T011 [P] Write `README.md` at repository root — include: one-line description, install command (`/plugin marketplace add muthuspark/solid-audit`), slash command reference table (all 7 commands, whether they modify files, confirmation required), typical workflow (write code → stage → `/solid-audit` → fix command → review diff → PR), supported languages table, what gets skipped automatically
- [ ] T012 Confirm `.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json` both show `"version": "1.0.0"` (must match), then create git tag `v1.0.0`

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — start immediately
- **Foundational (Phase 2)**: Depends on T001 — MUST complete before testing any skill
- **US1 (Phase 3)**: Depends on Phase 2 — T003 and T004 can run in parallel with each other
- **US2 (Phase 4)**: Depends on Phase 2 — T005 through T009 can all run in parallel
- **US3 (Phase 5)**: Depends on Phase 4 being complete (solid-fix references all five fix skills)
- **Polish (Phase 6)**: Depends on all user stories complete

### User Story Dependencies

- **US1 (P1)**: No dependency on US2 or US3 — deliverable independently
- **US2 (P2)**: No dependency on US1 (fix skills are independent) — but team gets more value after seeing US1 audit output
- **US3 (P3)**: Depends on US2 — `/solid-fix` orchestrates the five fix skills

### Parallel Opportunities

- T003 and T004 can run in parallel (different files within US1)
- T005, T006, T007, T008, T009 can all run in parallel (five different SKILL.md files)
- T011 (README) can run in parallel with T010 (solid-fix SKILL.md)

---

## Parallel Example: User Story 2

```
# Launch all five fix skills in parallel (different files, no shared state):
Task A: Write .claude/skills/srp/SKILL.md      (T005)
Task B: Write .claude/skills/ocp/SKILL.md      (T006)
Task C: Write .claude/skills/lsp/SKILL.md      (T007)
Task D: Write .claude/skills/isp/SKILL.md      (T008)
Task E: Write .claude/skills/dip/SKILL.md      (T009)
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Create directory structure (T001)
2. Complete Phase 2: Verify manifests (T002)
3. Complete Phase 3: Write solid-audit SKILL.md + examples.md (T003, T004)
4. **STOP and VALIDATE**: Run `/solid-audit` on prepared test files (see plan.md Test Cases table)
5. Share `/solid-audit` with team — safe to use for code review before fix skills are ready

### Incremental Delivery

1. T001 + T002 → Structure and manifests ready
2. T003 + T004 → `/solid-audit` ready → share with team (MVP)
3. T005–T009 → `/srp` through `/dip` ready → team can fix violations per-principle
4. T010 → `/solid-fix` ready → full workflow available
5. T011 + T012 → README published, v1.0.0 tagged

---

## Notes

- [P] tasks = different files, no dependencies on incomplete sibling tasks
- All seven skills share the same SKILL.md section structure (research.md Section 2) — write solid-audit first to establish the pattern
- Verify each skill manually before marking complete (see plan.md Test Cases table for inputs and pass criteria)
- Both `.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json` `version` fields must always match — update atomically
- Commit after each phase checkpoint
