---
id: doc-isaac-prd
type: doc
status: draft
updated: 2026-09-22
---

# Isaac — Product Requirements Document

Isaac is a re-platform of `build-agents-sdd` v3.1.0 into a separate git repository. This PRD is
derived entirely by reading the `build-agents-sdd` source tree. Every feature entry cites the
source path it came from. Where a requirement depends on a property of the JEV model that cannot
be verified from this repository, the text carries a `[JEV: ...]` marker and the marker is
repeated in §10 Open Questions. Where the source repository is ambiguous or self-contradictory,
the text carries `[uncertain: <path>]` and both sides are cited in §10.

---

## 1. Overview

### What Isaac is

Isaac is an agent-harness plugin that gives any software project an autonomous multi-phase SDLC
pipeline, persistent project memory, a self-learning loop, a codebase-index MCP server, branch
sync, and cross-harness adapters. It is a functional re-platform of `build-agents-sdd`
(`README.md:3-5`, `.claude-plugin/plugin.json:5`, `CLAUDE.md` "What this repo is").

Isaac ships the same pipeline, the same phase files, the same specialist skills, the same
background agents, the same memory layout, the same JSON schemas, and the same slash-command
names. It is not a redesign.

### Delta 1 — the JEV model

Isaac targets the AI model **JEV** in order to speed up tool calls. Nothing in this repository
documents JEV's capabilities, context window, pricing, tool-call latency, parallelism limits,
or API shape. This PRD therefore never asserts a JEV property. Every place `build-agents-sdd`
names a model or a tool-call strategy is restated in §5 as a functional requirement for Isaac,
with a `[JEV: ...]` marker on the unverified part.

Source for what has to be restated: `commands/build-feature.md:160-177` (per-phase model table),
`commands/setup/agent-spawning-model.md:30-43` (model selection and parallelism),
`skills/*/SKILL.md:4` and `agents/*.md:5` (`model:` frontmatter on 20 files),
`commands/current-context.md:13,92` (1M-token budget), `skills/context-reporter/SKILL.md:35-37,78-80`
(pricing table), `skills/contract-loop/SKILL.md:36` (`IMPLEMENTER_MODEL`).

### Delta 2 — MCP as the only file-write path

In Isaac, **every** file create, modify, delete, move, directory creation, and append goes through
an MCP server tool. Native `Write`, `Edit`, `MultiEdit`, `NotebookEdit`, and shell-based writes
(`>`, `>>`, `tee`, `sed -i`, `cat > file`, `cp`, `mv`, `mkdir`, `touch`, `rm`) are forbidden for
agent-authored artifacts and are denied by a `PreToolUse` hook.

`build-agents-sdd` already contains the seed of this: `commands/setup/preflight-questions.md:70-96`
asks the operator to choose an artifact output backend from `filesystem | mcp | database | custom`,
and `commands/setup/output-write-adapter.md:24-30` defines the `mcp` branch. Isaac removes the
choice: `outputBackend` is always `mcp`, the question is dropped, and the adapter block is
unconditional. Full specification in §6.

### What is explicitly unchanged

| Unchanged | Source of truth in build-agents-sdd |
|---|---|
| All 17 slash-command names | `commands/*.md` |
| All 11 phase files and their phase ids (1, 2+3, 03, 4, 4.5, 5, 6, 6.5, 7, 8a/8b/8c, 9) | `commands/phases/*.md`, `schemas/story-snapshot.schema.json` phase id enum |
| All 6 protocol modules | `commands/protocols/*.md` |
| Both rule modules and all 16 rules | `commands/rules/core-rules.md`, `commands/rules/loop-mode-rules.md` |
| All 8 setup modules | `commands/setup/*.md` |
| All 21 skill names | `skills/*/SKILL.md` |
| All 6 agent names | `agents/*.md` |
| Flags `--council`, `--step`, `--loop`, `--force`, ` resume` | `commands/setup/flags-and-modes.md:10-19` |
| Memory layout `.claude/memory/`, `.claude/learnings/` | `docs/memory-system.md:37-49` |
| All 6 JSON schemas and their field contracts | `schemas/*.schema.json` |
| All 5 templates | `templates/*.md` |
| `sdd-index` MCP server and its 4 tools | `scripts/mcp-index-server.py`, `.mcp.json` |
| Story-id format, track-letter rule, artifact naming | `commands/setup/flags-and-modes.md:44`, `schemas/task-graph.schema.json` |
| Hook events and guard semantics | `hooks/hooks.json` |
| Adapter targets OpenCode / Codex / Devin | `dist/`, `scripts/build-adapters.py` |

---

## 2. Goals and Non-Goals

### Goals

- **G1** — Reproduce every feature in §4 with identical user-facing names.
- **G2** — Run the pipeline on JEV with fewer wall-clock seconds spent in tool calls than the
  equivalent `build-agents-sdd` run. `[JEV: the mechanism by which JEV reduces tool-call cost —
  batched calls, lower per-call latency, higher parallel-call ceiling, or something else — is
  unverified.]`
- **G3** — Make MCP the sole file-write surface, enforced by a hook rather than by convention.
- **G4** — Keep the model name out of every command, skill, agent, phase, protocol, rule, and
  setup file. One config value, read at runtime.
- **G5** — Preserve the memory and learnings file contract byte-for-byte so a project can move
  between `build-agents-sdd` and Isaac without splitting its memory
  (`docs/cross-platform.md:61-64` establishes this property across harnesses; Isaac extends it
  across products).
- **G6** — Preserve the schema contracts so existing `docs/<story-id>/` trees resume under Isaac.

### Non-Goals

- **NG1** — **Renaming anything.** No command, phase, skill, agent, protocol, rule, setup module,
  flag, schema field, artifact filename, or hook may be renamed. A name that exists in
  `build-agents-sdd` exists in Isaac spelled identically.
- **NG2** — **Adding features not present in `build-agents-sdd`.** Isaac ships the inventory in
  §4 and nothing else. New capability is a post-1.0 decision.
- **NG3** — Fixing `build-agents-sdd`'s documented defects as part of the port. The known defects
  are recorded in §10 so the Isaac team decides explicitly; silently "improving" them would break
  parity.
- **NG4** — Changing the pipeline's human-gate structure. Every gate in §4 stays a gate.
- **NG5** — Supporting a non-MCP write backend. `database` and `custom` from
  `commands/setup/preflight-questions.md:88-96` are dropped, not preserved as options.
- **NG6** — Replacing the filesystem as the storage medium. MCP is the write *path*; the files
  still land on disk at the same paths.

---

## 3. Personas

Derived only from what the source docs state. No persona is invented.

| Persona | Description | Source |
|---|---|---|
| **ServiceNow developer new to agent harnesses** | Knows JavaScript, has never used Claude Code. The explicit target audience of the onboarding course material. | `docs/incident-priority-advisor-course.md:126`, `docs/remotion-lecture-prompts.md:28` |
| **Engineering Lead (the orchestrator role)** | The single agent role that owns a `/build-feature` run: spawns specialists, gate-checks outputs, is the user's single point of contact, never implements. | `commands/build-feature.md:14-17`, `commands/rules/core-rules.md:19` |
| **Team rolling out the pipeline** | Wants centralized observability over concurrent runs, cross-story knowledge, and reuse metrics. | `docs/plugin-overview.md:221-225` |
| **Internal ServiceNow staff** | Onboarding videos are SSO-gated; repo is bound to `code.devsnc.com`; announcements go to an internal Teams channel. | `docs/required-viewing.md`, `CLAUDE.md` GitHub Account Binding, `CLAUDE.md` rule 7 |
| **Workflow-depth personas** | Five stated project depths that select spec-light vs full-spec: prototype, spike, feature, platform, regulated. | `docs/development-workflow.md:29-43` |
| **New contributor** | Onboarded by being handed a spec-authoring entry point and a problem. | `docs/development-workflow.md:894` |
| **Council personas (in-pipeline, not human)** | Product Mind, Devil's Advocate, Security Advocate — three parallel perspectives under `--council`. | `agents/product-council.md:40,75,111` |

---

## 4. Feature Inventory

### 4.1 Commands

All 17 files under `commands/*.md`. Names are identical in Isaac.

| Name | Purpose | Trigger | Inputs | Outputs / files written | Human gates | Depends on | Source |
|---|---|---|---|---|---|---|---|
| `/build-feature` | Full autonomous SDLC orchestrator, idea to merged PR. Never implements. | `/build-feature <request>` or `/build-feature <story-id> resume` | `$ARGUMENTS`, `pwd`, `knowledge/`, all setup/protocol/rule/phase files | `docs/<story-id>/`, `docs/<story-id>/loop-contracts/`, `story-snapshot.json`, `session.json`, `.claude/.context-extract-pending`, codebase index | Scaffolding self-check (hard stop), command-fit, staging guard, pre-flight questions, context guard, step gate, evidence gate, blocker escalation, Phase 03 epic gate | All 8 setup modules, all 6 protocols, both rule files, all 11 phases, `story-state.py`, `scan-codebase.py`, 12 skills, `verificator` agent | `commands/build-feature.md` |
| `/quick-feature` | Lightweight 4-phase SDLC: scope, implement, test, commit. No discovery, UX, architecture, or review loop. | `/quick-feature <description>` or `<story-id> resume` | `$ARGUMENTS`, `pwd`, `knowledge/tech-stack.md`, `knowledge/repositories.md`, target `CLAUDE.md` | `docs/<story-id>/scope.md`, `session.json`, `story-snapshot.json`, `implementation-notes.md`, `test-results.md`; worktree; branch `feat/qf-<story-id>`; story id `<APP>-QF-YYYYMMDD-001-<SLUG>` | Command-fit scoring, app question, staging guard, output-backend question, scope approval, test-failure choice, context guard | `developer`, `test-engineer`, `devops-engineer` skills; `story-state.py`, `scan-codebase.py`, `story-graph.py`; `session-config.schema.json` | `commands/quick-feature.md` |
| `/archive` | Move a completed story's artifacts to the archive, clean merged branches, update memory. | `/archive <story-id>` | `story-snapshot.json`, `knowledge/repositories.md`, `.claude/memory/project-memory.md` | creates `docs/_archived/`, moves `docs/<story-id>` into it, `story-state.py status --set archived`, appends a `## State` line to `project-memory.md` | confirm when Phase 9 incomplete; confirm before force-deleting an unmerged branch | `story-state.py`, git | `commands/archive.md` |
| `/context-report` | HTML report of files read, prompts, token breakdown, USD cost for a phase or session. | `/context-report [story-id] [phase-number]` | session JSONL, `skills/context-reporter/SKILL.md` | `/tmp/context_report_gen.py`, `reports/<story-id>/phase-<N>-context-<ts>.html`, git commit of `reports/` | asks the human for story id / phase if absent | `context-reporter` skill | `commands/context-report.md` |
| `/current-context` | Read-only audit of everything in the context window with token estimates and % of budget. | `/current-context` | global and repo `CLAUDE.md` plus its `@`-includes, `distilled.md`, 5 memory files, spec dir, skill/agent/command listings | **none — never writes, never spawns** | none | none | `commands/current-context.md` |
| `/debug-browser` | Reproduce a UI bug with browser automation, capture evidence, root-cause, fix, verify. | `/debug-browser <bug report>` | `$ARGUMENTS`, `pwd` | `docs/debug-<ts>/bug-report.md`, `evidence-raw.md`, `root-cause.md`, `fix-notes.md`, `verification.md`, `screenshots/01-bug-state.png`, `screenshots/02-fix-verified.png`; worktree; branch `fix/<id>-<slug>` | missing-field question, staging guard, "proceed to apply fix?", verification-failure options, context guard | Playwright MCP, Chrome DevTools MCP, git worktree | `commands/debug-browser.md` |
| `/gate-verify` | Independently double-verify one quality gate against memory and governance. Read-only, never fixes. | `/gate-verify <gate>`; empty argument defaults to "current branch is ready to merge" | `CLAUDE.md`, `distilled.md`, 5 memory files, story ADRs, `story-snapshot.json`, `commands/rules/*`, `commands/protocols/*`, `setup/flags-and-modes.md`, relevant SKILL.md files; probes `.mcp.json` for the tracker | **report only — forbidden to write anything** | stop and ask on governance conflict, stale memory, destructive command, or ≥2 missing Step-0 files | SDD Brain MCP read tools, `story-state.py show` | `commands/gate-verify.md` |
| `/learn-from` | Explicit teaching. Writes a high-confidence learning straight to `distilled.md`, skipping pending. | `/learn-from <rule>` | `$ARGUMENTS`, `distilled.md` | `.claude/learnings/distilled.md` entry `## L-YYYY-MM-DD-NNN [active, confidence: high]`; appends `.claude/learnings/corrections.jsonl` | max one clarifying question; conflict resolution REPLACE/NARROW/KEEP BOTH | none | `commands/learn-from.md` |
| `/match-figma-design` | Translate a Figma frame into production code: tokens, assets, responsive and accessible components. | `/match-figma-design figma_url= target_stack= logo_dir= output_dir=` | the four required args, repo framework detection, logo catalog, Figma MCP or REST | components/pages under `output_dir`, design tokens into theme config, exported assets, assets manifest | hard stop on any missing arg; stop if neither Figma MCP nor REST available; flag a missing logo variant rather than substituting | Figma MCP or REST | `commands/match-figma-design.md` |
| `/memory-prune` | Actively trim stale memory. Monthly at most. | `/memory-prune` | `project-memory.md`, `decisions.md`, `patterns.md`, `distilled.md`, `git branch -a` | rewrites those four with `## Archived` sections; `.claude/learnings/retired.md` | batched proposal then per-category yes/no; diff shown before each write | none | `commands/memory-prune.md` |
| `/memory-recall` | Search accumulated memory for context relevant to a query. | `/memory-recall <query>` | SDD Brain MCP first for cross-story questions; then local memory files and 5 recent session logs | **none — read-only** | none | SDD Brain MCP | `commands/memory-recall.md` |
| `/memory-review` | Audit curated memory and active learnings; surface stale entries, pending candidates, conflicts. | `/memory-review` | `project-memory.md`, `decisions.md`, `patterns.md`, `distilled.md`, `pending.md` | updates `distilled.md` and `pending.md` on confirmation; `retired.md` on retirement | PROMOTE/DISCARD/SKIP per candidate; UPDATE/RETIRE/KEEP per staleness flag; diff before every write | none | `commands/memory-review.md` |
| `/memory-save` | Save one explicit fact to long-term memory. | `/memory-save <fact>` | `$ARGUMENTS`, existing memory for duplicate and contradiction check | `project-memory.md` under the matching `##` section, or a new `## D-YYYY-MM-DD-NN` block in `decisions.md` | confirm classification; surface contradictions; show exact diff | none | `commands/memory-save.md` |
| `/pause` | Checkpoint, not end. Snapshot state, ask what to remember, run memory and learning agents synchronously, write resume context. | `/pause` | `git status --short`, task list, session log, `.claude/.learning-pending` | `.claude/.resume-context.md` (overwritten), `<session-log>.curated` marker, whatever curator/observer write; removes `.learning-pending` | waits for the user's reply; reply vocabulary `save:`, `decide:`, `teach:`, `skip <id>`, `proceed`, `more` | `memory-curator`, `learning-observer` agents | `commands/pause.md` |
| `/review-plan` | Read a PR's review comments and produce a prioritised implementation plan. | `/review-plan <PR number \| URL \| owner/repo#number>` | git remote, code-platform MCP tools discovered by name pattern, CLI fallbacks | conversation by default; `docs/pr-<number>-plan.md` on `"save plan"` | prompt for a missing token; diff before writing on `"implement B<N>"` | code-platform MCP or CLI | `commands/review-plan.md` |
| `/review-pr` | Full code review of a PR diff across six lenses with a verdict. | `/review-pr <PR number \| URL \| owner/repo#number>` | same platform detection, PR diff, `distilled.md` for project rules | conversation by default; `docs/pr-<number>-review.md` on `"save review"`; posts to the PR on `"post review"` | never post without explicit confirmation; ask which files to prioritise on a large diff; diff before applying a fix | code-platform MCP or CLI | `commands/review-pr.md` |
| `/stop` | End a unit of work: capture disposition, consolidate memory, run context extraction, archive resume context. | `/stop` | disposition reply, session log, `.context-extract-pending`, `.resume-context.md` | memory files via curator; `.curated` marker; `.claude/.resume-archive/<ts>_<disposition>.md`; deletes `.context-extract-pending` | disposition question (Complete/Shipped/Parked/Abandoned); `save:`/`decide:`/`teach:`/`skip`/`proceed`; extractor A/B/C/D approval; never run destructive git without confirmation | `memory-curator`, `learning-observer`, `context-extractor` agents | `commands/stop.md` |

**Flags on `/build-feature`** (verbatim, `commands/setup/flags-and-modes.md:10-19`):

| Flag | Variable set | Behaviour |
|---|---|---|
| ` resume` (suffix) | — | Skip to the resuming protocol; skip fit check and app question |
| `--force` | — | Skip the command-fit check |
| `--council` | `COUNCIL_MODE=true` | Three council agents debate before Phase 1 synthesis |
| `--step` | `STEP_MODE=true` | Pause after every phase gate for `proceed` / `stop` / `skip` |
| `--loop` | `LOOP_MODE=true` | Contract-driven evidence gating at every phase |
| story id contains `/epics/` | `EPIC_MODE=true` | Skip 0a, story-id generation, Phases 1, 2+3, 03; start at Phase 4 |

Combinations are explicit: `--loop --step`, `--loop --council`, `--loop --step --council`
(`commands/setup/flags-and-modes.md:98-103`). Ordering rule: LOOP_MODE runs first, then STEP_MODE
asks (`:89-90`).

### 4.2 Pipeline Phases

All 11 files under `commands/phases/`. Phase ids come from the snapshot schema enum
(`schemas/story-snapshot.schema.json`): `0, 1, 2, 3, 03, 4, 4.5, 5, 6, 6.5, 7, 8a, 8b, 8c, 9`.

| Phase | File | Purpose | Agent type / model | Required outputs | Validation gate | Human gates |
|---|---|---|---|---|---|---|
| 1 — Discovery | `commands/phases/01-discovery.md` | Validate the idea, produce acceptance criteria | `claude`, inherit; council mode spawns 3 + a synthesis agent | `docs/<story-id>/validated-idea.md`; council adds `council/product-mind.md`, `council/devils-advocate.md`, `council/security-advocate.md` | file exists; `AC_TOTAL ≥ 3`; zero ambiguity markers; context guard | clarifying questions if ACs cannot be made explicit; the built-in-first three-option choice |
| 2+3 — Spec + UX | `commands/phases/02-spec-ux.md` | Spec writing and UX design in parallel | 2× `claude`, inherit | `requirements.md`, `user-stories.md`, `adrs/`, `ux-wireframes.md` | all three exist; ≥1 FR; ≥1 US; context guard | **hard gate** — no output file written until the user approves the draft |
| 03 — Epic Decomposition | `commands/phases/03-epic-decomposition.md` | Propose (never execute) an epic split sharing one spec and ADR set | `claude`, inherit | `epics/_epics-proposal.md`; on accept `epics/_epics.md` and `epics/epic-NN-<slug>/` | proposal exists | **always asks**, regardless of the agent's recommendation: 1 accept / 2 decline / 3 request changes. Accept **stops the session** — never auto-runs an epic's pipeline |
| 4 — Architecture | `commands/phases/04-architecture.md` | Data model, API, integrations, parallel track split | `claude`, inherit | `architecture.md`, `implementation-plan.md`; appends `.claude/memory/decisions.md` | both exist; duplicate-path detector prints `CONFLICTS: NONE`; context guard | the built-in-first three-option choice |
| 4.5 — Task Decomposition | `commands/phases/04b-task-decomposer.md` | Turn the plan into per-task files plus a schema-valid task graph | `claude`, inherit | `codebase-context.md`, `tasks/task-graph.json`, `tasks/task_NN.md` | schema validation; `COUNTS_MATCH: True`; ≥1 task file; context guard | none |
| 5 — Branch Setup | `commands/phases/05-branch-setup.md` | Feature branches, worktrees, project scaffold | `claude`, **`haiku`** in source | branches `feat/<story-id>-<track-short-name>`; worktrees recorded in the snapshot | branches exist; `.gitignore` covers `.env`, `*.key`, `*.pem`; reviewers set; context guard | none |
| 6 — Implementation | `commands/phases/06-implementation.md` | Parallel per-track implementation in isolated worktrees | N× `claude` background calls in one message, inherit | source files; `tracks/06-track-<T>/implementation-notes.md` with `## Build Output`, `## Verification`, `## Unit Tests Written`; appends `project-memory.md` | unticked checkboxes = 0; notes sections present; `TRACKS_FAILED: none`; `MISSING_TESTS: NONE`; context guard | on track failure: 1 retry / 2 fix manually / 3 abort |
| 6.5 — Spec Verification | `commands/phases/06b-verificator.md` | Isolated spec-vs-artifact verification | `build-agents-sdd:verificator` registered agent; model from agent frontmatter | `verification-result.json`, `verification-report.md` | report exists; JSON validates; verdict read from the `verdict` field; context guard | `PARTIAL` asks proceed/fix; `FAIL` stops: 1 back to Phase 6 / 2 continue with gaps / 3 abort |
| 7 — Testing | `commands/phases/07-testing.md` | Test plan, execution, regression, unit-test audit | `claude`, inherit | `test-plan.md`, `test-results.md` | both exist; `TC_TOTAL ≥ AC_TOTAL`; pass/fail rows counted; context guard | if any E2E failure: fix-first or proceed |
| 8 — Review + Audit + Fix | `commands/phases/08-review-loop.md` | 8a code review and 8b security audit in parallel, then 8c fix rounds, max 3 | 2× `claude` parallel then 1× `claude`, inherit | `reviews-001/code_NN.md` (BLOCKER+MAJOR), `reviews-001/security_NN.md` (CRITICAL+HIGH), `review-report.md`, `security-report.md`, `reviews-<NNN>/round-summary.md` | round status clean; the snapshot `review` block is owned by the code-reviewer | after 3 rounds with issues open: 1 continue / 2 proceed with known issues / 3 abort |
| 9 — PR | `commands/phases/09-pr.md` | Assemble and open the pull request | `claude`, **`haiku`** in source | PR titled `[<story-id>] <title>`, labels `feature, ready-for-review`; PR URL into the snapshot; writes `.claude/.context-extract-pending` | PR created | PR approval gate — the user reviews and merges |

Phase-transition protocol between every gate: `commands/protocols/phase-transition.md` (4 steps —
context guard, graph rebuild + MCP write-through, evidence gate if LOOP_MODE, step gate if
STEP_MODE, then load the next phase file).

### 4.3 Protocols

| Name | Purpose | Trigger | Writes | Source |
|---|---|---|---|---|
| `blocker-escalation` | Handle an agent reporting `BLOCKED: <reason>` — stop, report, wait, resume only the blocked phase | any agent emits `BLOCKED:` | none | `commands/protocols/blocker-escalation.md` |
| `context-guard` | Story-directory size proxy for context pressure; threshold `CONTEXT_GUARD_THRESHOLD_KB` default 1600 | one line inside every phase's validation bash block | `story-snapshot.json` via `status --set paused` and `phase --status blocked` | `commands/protocols/context-guard.md` |
| `epic-scoping` | Detect `/epics/` in the story id, split `PARENT_ID` / `EPIC_ID`, resolve the six shared inputs against the parent | story id contains `/epics/` | epic's own `story-snapshot.json` | `commands/protocols/epic-scoping.md` |
| `human-in-the-loop` | The `❓ QUESTION:` block format and the ask-vs-proceed rule | any genuine two-way choice or go/no-go | none | `commands/protocols/human-in-the-loop.md` |
| `phase-transition` | The mandatory 4-step post-gate protocol including the unconditional graph rebuild and MCP write-through | after every phase validation gate | `story-graph.json`; loop-contract, evidence, failure files; snapshot | `commands/protocols/phase-transition.md` |
| `resuming` | Resume flow: ask the state file for the next phase, regenerate the index, resume | `$ARGUMENTS` ends with ` resume` | codebase index only | `commands/protocols/resuming.md` |

### 4.4 Rules

`commands/rules/core-rules.md` — 11 non-negotiables, carried verbatim into Isaac:

1. Never write or modify any file before the staging area is clean AND the worktree is created.
2. Never implement code yourself — always delegate.
3. Never skip phases.
4. Never skip Phase 4.5.
5. Never create a PR with unresolved BLOCKER/CRITICAL findings that were not user-accepted.
6. Never assume a test passed.
7. Never close a review-fix round with unverified fixes.
8. Always run the context guard after each phase validation gate.
9. Always keep `story-snapshot.json` current via the state helper.
10. You are the user's single point of contact.
11. Never proceed past a validation gate with failing checks.

`commands/rules/loop-mode-rules.md` — 5 additional rules when `LOOP_MODE=true`: **No evidence, not
done**; **Prove before move**; **Evidence is repeatable**; **One phase per dispatch**; **Descope
decisions are user decisions** (`MAX_ATTEMPTS` default 3).

### 4.5 Setup modules

| Name | Purpose | Writes | Human gate | Source |
|---|---|---|---|---|
| `agent-spawning-model` | Spawn discipline: skills are files an agent reads, never agent types; only the 6 registered agents are valid types; model selection and parallel-vs-sequential rules | none | none | `commands/setup/agent-spawning-model.md` |
| `command-fit-check` | 12-signal scoring table; ≥4 proceed, 1–3 borderline, ≤0 mismatch | none | waits for explicit confirmation | `commands/setup/command-fit-check.md` |
| `final-report-template` | Standard and loop-mode final report shapes | `loop-report.md` when LOOP_MODE | none | `commands/setup/final-report-template.md` |
| `flags-and-modes` | Flag parsing, app-name and slug sanitisation, story-id generation, per-flag behaviour, combination matrix | none directly | the app/package question | `commands/setup/flags-and-modes.md` |
| `output-write-adapter` | The block pasted into agent prompts when the backend is not `filesystem`; defines the `mcp`, `database`, `custom` branches | `.ref` stubs | none | `commands/setup/output-write-adapter.md` |
| `preflight-questions` | Deploy environment, reviewers, artifact output backend; writes and validates the session config | `docs/<story-id>/session.json` | waits for the reply to each question | `commands/setup/preflight-questions.md` |
| `session-initialization` | Create the story directory, seed the snapshot, build the codebase index, seed the story graph, announce active modes | story dir, `story-snapshot.json`, codebase index, `story-graph.json` | none | `commands/setup/session-initialization.md` |
| `staging-guard-and-worktree` | Staging-clean guard, worktree creation, branch creation, `WORKING_DIR` reassignment | git state | hard stop on a dirty index | `commands/setup/staging-guard-and-worktree.md` |

### 4.6 Skills

All 21 directories under `skills/`. Each has exactly one `SKILL.md`; only `spec-writer` has a
`references/` subdirectory.

| Skill | Purpose | Trigger | Key outputs | Human gates | `model:` in source | Source |
|---|---|---|---|---|---|---|
| `code-reviewer` | Pre-PR review for spec compliance, quality, security, performance, maintainability, unit-test coverage | Phase 8a | `reviews-001/code_NN.md`, `review-report.md`; snapshot `review` block | none; BLOCKER/MAJOR halts the PR | `sonnet` | `skills/code-reviewer/SKILL.md` |
| `context-reporter` | Self-contained HTML report of session context, tokens, cost | `/context-report` | `/tmp/context_report_gen.py`, `reports/<story-id>/*.html`, git commit | asks for the session JSONL path if not found | absent | `skills/context-reporter/SKILL.md` |
| `contract-loop` | Contract → Plan → Act → Prove → Move → Observe sweep with evidence bundles and an orchestrator/implementer split | user asks for a codebase sweep | `.claude/agents/loop-implementer.md`, `<out>/loop/{contract,plan,state,flagged,baseline}.md`, evidence dirs, reports, branch, commits, PR | dirty-tree stop, plan checkpoint, stop-and-ask on deletes/deps/schema/public-API | absent; `IMPLEMENTER_MODEL` defaults to `haiku` | `skills/contract-loop/SKILL.md` |
| `developer` | Track-scoped implementation, phases A–E | Phase 6, per track | source files, co-located unit tests, `implementation-notes.md`, appends `project-memory.md` | none; emits `BLOCKED:` | `sonnet` | `skills/developer/SKILL.md` |
| `devops-engineer` | Branch and worktree setup (Phase A); PR creation, merge, cleanup (Phase B) | Phases 5 and 9 | branches, worktrees, PRs; snapshot `worktree` and `prUrl` | none | `haiku` | `skills/devops-engineer/SKILL.md` |
| `e2e-testing-patterns` | Reference patterns for browser-driven end-to-end tests: login, navigation, forms, REST verification, screenshots | read inline by `test-engineer` Phase B | rows and screenshot labels into `test-results.md` | none | absent | `skills/e2e-testing-patterns/SKILL.md` |
| `epic-planner` | Evaluate whether the spec should split into epics sharing one spec and ADR set. Proposes only | Phase 03 | `epics/_epics-proposal.md`; snapshot `epics` block | hard — never decides unilaterally | `sonnet` | `skills/epic-planner/SKILL.md` |
| `feedback-triage` | Post-launch triage P0–P3, P0 hotfix fast lane, backlog classification | user supplies feedback | `triage-report.md`, appends `docs/backlog.md`, `docs/hotfix-<date>/validated-idea.md`, hotfix worktree | confirm a P0 is reproducible | `haiku` | `skills/feedback-triage/SKILL.md` |
| `git-conflict-resolver` | Semantic auto-resolution of merge conflicts with escalation | a merge or rebase fails | resolved source files, commit, escalation log | escalate on `ESCALATE` | `haiku` | `skills/git-conflict-resolver/SKILL.md` |
| `javascript-testing-patterns` | Unit-testing standard: co-location, mocking, async, ≥70% statement coverage | read inline by `test-engineer` and `code-reviewer` | co-located test files; the unit-test table in `implementation-notes.md` | none; <70% coverage is a BLOCKER upstream | absent | `skills/javascript-testing-patterns/SKILL.md` |
| `parallel-executor` | Orchestrator protocol for N parallel track agents in isolated worktrees with sequential commits | Phase 6 (≥2 tracks), Phase 8, Phase 1 council | worktrees, rsync copies, per-track notes, task-graph updates, commits, merges, cleanup | track-failure prompt: retry / fix manually / abort | absent | `skills/parallel-executor/SKILL.md` |
| `product-strategist` | Stress-test the raw request, produce the validated idea with explicit acceptance criteria | Phase 1 | `validated-idea.md` | presents the built-in-first three-option choice; must ask if ACs cannot be explicit | `sonnet` | `skills/product-strategist/SKILL.md` |
| `review-fixer` | Triage, fix, verify, and update status for one review round | Phase 8c, per round | modified source, issue-file frontmatter updates, `round-summary.md`, snapshot `review` block | none; escalates with `BLOCKED:` | `sonnet` | `skills/review-fixer/SKILL.md` |
| `security-auditor` | OWASP-based audit plus hardcoded-secret detection | Phase 8b | `reviews-001/security_NN.md`, `security-report.md` | CRITICAL/HIGH block the PR, no exceptions | `opus` | `skills/security-auditor/SKILL.md` |
| `sn-built-in-first` | Build-vs-buy advisory grounded in platform docs when a docs MCP is available; always presents three options | read inline by `product-strategist`, `spec-writer`, `technical-architect` | **writes nothing** — returns a recommendation line | hard — always presents the three-option choice and waits | absent | `skills/sn-built-in-first/SKILL.md` |
| `spec-writer` | Clarify, research, choose an approach, draft, and on approval write the spec | Phase 2 | `requirements.md`, `user-stories.md`, `adrs/adr-001.md` | **multiple hard gates** — nothing written before approval; one question per message; approach selection; A/B/C/D draft approval | `sonnet` | `skills/spec-writer/SKILL.md` |
| `task-decomposer` | Codebase enrichment then per-task files and a schema-valid task graph | Phase 4.5 | `codebase-context.md`, `tasks/task_NN.md`, `tasks/task-graph.json` | none | `sonnet` | `skills/task-decomposer/SKILL.md` |
| `technical-architect` | Data model, API, integration design, parallel track split | Phase 4 | `architecture.md`, `implementation-plan.md`, appends `decisions.md` | the built-in-first three-option choice | `opus` | `skills/technical-architect/SKILL.md` |
| `test-engineer` | Test plan, execution, regression, unit-test audit. Owns integration and E2E; does not write unit tests | Phase 7 | `test-plan.md`, `test-results.md` | none; BLOCKED on untested artifacts, <100% unit pass, <70% coverage, or an unreachable environment | `sonnet` | `skills/test-engineer/SKILL.md` |
| `using-git-worktrees` | The standard worktree lifecycle for every code-changing skill | read by any code-changing skill | worktrees, branches, commits; snapshot `worktree` | hard — always asks for the worktree root before creating anything | absent | `skills/using-git-worktrees/SKILL.md` |
| `ux-designer` | User journeys, layouts, states, component mapping, accessibility | Phase 3, parallel with Phase 2 | `ux-wireframes.md` | none | `sonnet` | `skills/ux-designer/SKILL.md` |

`spec-writer` reference files, carried unchanged: `references/adr-template.md`,
`references/question-protocol.md`, `references/requirements-template.md` (all `type: reference`,
no `name:` field by design).

### 4.7 Agents

All 6 files under `agents/`. These are the **only** valid `subagent_type` values besides the
generic one (`commands/setup/agent-spawning-model.md:12-16`).

| Agent | Purpose | Trigger | Tools in source | Outputs | Human gates | `model:` in source | Source |
|---|---|---|---|---|---|---|---|
| `branch-sync` | Resolve a failed automated sync where the current branch is the source of truth | the sync script's merge failed | `Bash, Read, Glob` | merge commit, push, one-line log append | none — headless | `sonnet` | `agents/branch-sync.md` |
| `context-extractor` | Two-phase DRAFT then WRITE extraction of session knowledge into memory | `/stop` when the extract-pending marker exists | `Read, Write, Edit, Glob, Grep, Bash` | `decisions.md`, `skills.md`, `rules.md`, `tokens-consumed.md` — append/create only | **hard** — A/B/C/D approval before any write; spawns 4 read-only sub-agents in parallel | `haiku` | `agents/context-extractor.md` |
| `learning-observer` | Mine human-in-the-loop corrections into structured learning rules across 5 signal types | session end when a signal flag is pending | `Read, Write, Edit, Glob, Grep, Bash` | `pending.md`, `distilled.md`, `corrections.jsonl`, truncates `post-push-queue.md` | none at runtime; retirement only via user commands | `haiku` | `agents/learning-observer.md` |
| `memory-curator` | Consolidate a session log into durable project memory | `/pause` and `/stop` | `Read, Write, Edit, Glob, Grep, Bash` | `project-memory.md`, `decisions.md`, `patterns.md`, `.curated` marker | none — headless | `haiku` | `agents/memory-curator.md` |
| `product-council` | Three parallel perspectives before Phase 1 synthesis | `--council` | `Bash, Read` | `council/product-mind.md`, `council/devils-advocate.md`, `council/security-advocate.md` | none in the agent | `sonnet` | `agents/product-council.md` |
| `verificator` | Isolated spec-vs-artifact verification under a strict isolation contract | Phase 6.5 | `Read, Write, Glob, Grep, Bash` | `verification-report.md`, `verification-result.json`, appends `distilled.md` | none — headless | `sonnet` | `agents/verificator.md` |

Write-scope rules carried into Isaac: the curator writes only `.claude/memory/`; the observer only
`.claude/learnings/`; the extractor only `.claude/memory/` and only after approval; the verificator
only its two result files plus `distilled.md`; `gate-verify` writes nothing at all.

### 4.8 Hooks

`hooks/hooks.json` registers 5 entries across 4 events.

| Event | Matcher | Script | Behaviour | Exit semantics |
|---|---|---|---|---|
| `SessionStart` | `*` | `scripts/session-start.sh` | Backgrounds branch sync and (conditionally) the codebase scan; builds `additionalContext` from the workflow prompt plus memory and learnings files | 0 only; never blocks |
| `UserPromptSubmit` | `*` | `scripts/user-prompt-submit.sh` | Classifies the prompt as teaching / correction / preference; writes `.claude/.learning-pending` | 0 only |
| `PreToolUse` | `Edit\|Write\|MultiEdit\|NotebookEdit\|Bash` | `scripts/pre-tool-use.sh` | Branch guard on protected branches plus a shell-read guard over the codebase index | **2 blocks**, 0 allows |
| `PreToolUse` | `Read` | `scripts/read-guard.sh` | Blocks reading the codebase index always; blocks whole-file reads of the four large artifacts while the story is active | **2 blocks**, 0 allows |
| `Stop` | `*` | `scripts/stop.sh` | Spawns the learning observer headlessly when a signal is pending; never touches the extract-pending marker | 0 only |

`scripts/post-tool-use.sh` exists and is shipped to the adapter overlays but is **not registered**
in `hooks/hooks.json` and is a pure `exit 0` no-op.

Injection order built by the session-start hook (`docs/memory-system.md:53-63`): workflow prompt,
active learned rules, last 70 lines of decision blocks, the project-memory `## State` body, last
40 lines of `skills.md`, last 50 lines of `rules.md`, pending-learnings count.

### 4.9 MCP server — `sdd-index`

`scripts/mcp-index-server.py`. JSON-RPC 2.0 over stdio, newline-delimited, standard library only.
Registered by `.mcp.json`. Server name `sdd-index`. Imports the scanner as a module so filter logic
lives in one place. Root resolved once at import from the project-directory environment variable,
falling back to the process working directory. Default result limit 20, hard cap 50.

| Tool | Parameters | Returns | Errors |
|---|---|---|---|
| `find_files` | `prefix` (string or array), `symbol` (case-insensitive substring, string or array), `language`, `importsOf`, `importedBy`, `testsFor`, `untested` (boolean), `limit` (integer, clamped 0–50, non-integer falls back to 20). `additionalProperties: false` | `{generatedAt, totalMatches, shown, limit, indexBuilt, files:[{path, language, loc, exportedSymbols[], imports[], tests[], summary}]}` | Builds the index on first use and reports it via `indexBuilt` |
| `show_file` | `path` (string, **required**) | the full index entry — `path, hash, language, loc, symbols[{name,kind,exported,line}], imports, tests, summary` — plus `indexBuilt` | `show_file requires 'path'`; `<path> is not in the index` — both returned as a tool result with `isError: true`, not a JSON-RPC error |
| `index_status` | none | `{root, indexExists:false}` or `{root, indexExists:true, generatedAt, fileCount, symbolCount, edgeCount}` | — |
| `refresh_index` | none | `{root, generatedAt, fileCount, reused}` | the only tool that writes unconditionally |

Protocol methods handled: `initialize` (echoes the client protocol version, advertises tool
capability), `notifications/initialized`, `notifications/cancelled`, `ping`, `tools/list` (sorted),
`tools/call`. Error codes: `-32700` parse error, `-32601` unknown method, `-32602` unknown tool.

Per-harness registration (`docs/cross-platform.md:96-120`, `CHANGELOG.md:20-35`): Claude-family
harness auto-loads the plugin-root MCP manifest; Cursor and Codex native manifests point at the
same file with a string pointer; the OpenCode overlay gets a generated `mcp` block; the Codex
overlay gets a generated server table entry, treated as merge-sensitive; Devin supports MCP only at
user level so the overlay ships a copy-pasteable snippet and the installer prints where to merge it.

**Second, optional MCP — `sdd-mcp-tracker` / SDD Brain** (`docs/mcp-tracker-setup.md`,
`docs/CONVENTIONS.md:88-102`). Phase 1 probes the MCP manifest for the server name and sets
`MCP_ENABLED`. Tools named in the source: `list_active_stories`, `get_specs`, `get_story_status`,
`get_decisions`, `get_findings`, `get_rules`, `get_patterns`, `get_verification_summary`,
`upsert_story_snapshot`, `upsert_spec`, `patch_story_status`, `archive_story`. When enabled, every
phase gate writes the snapshot through and the local snapshot is git-excluded.

### 4.10 Memory system

| Path | Contract | Written by | Committed |
|---|---|---|---|
| `.claude/memory/project-memory.md` | `# Project Memory` plus fixed `##` sections Architecture, Modules, Conventions, State. Entries are `- YYYY-MM-DD: <fact>`. `## State` is **replaced**, not appended | `memory-curator`, `developer` (inline), `/memory-save`, `/archive` | yes |
| `.claude/memory/decisions.md` | `## D-YYYY-MM-DD-NN: <title>` blocks with Status, Context, Decision, Alternatives considered, Consequences, Source | `memory-curator`, `context-extractor`, `technical-architect` (inline), `/memory-save` | yes |
| `.claude/memory/patterns.md` | `## P-N: <name>` with Sites observed, Shape, Promote to. Added only at ≥3 instances | `memory-curator` only | yes |
| `.claude/memory/rules.md` | `## Rules Updated <date> (Session <id>)` blocks | `context-extractor` | yes |
| `.claude/memory/skills.md` | `## Session <id> — <date>` blocks recording which phases ran | `context-extractor` | yes |
| `.claude/memory/tokens-consumed.md` | `## Session <id> — <date>` blocks | `context-extractor` | — |
| `.claude/memory/session-log/<id>.jsonl` | one JSON object per line: `session_start`, `user_prompt`, `tool_use`, `session_stop` | hook scripts | **no** |
| `.claude/memory/session-log/<id>.jsonl.state-changes` | one line per state-changing tool call | post-tool hook | no |
| `.claude/memory/session-log/<id>.jsonl.curated` | marker suppressing curator re-spawn | `/pause`, `/stop`, curator | no |
| `.claude/memory/branch-sync.log` | append-only sync outcome log | branch sync script and agent | no |

### 4.11 Learning system

| Path | Contract | Written by | Committed |
|---|---|---|---|
| `.claude/learnings/distilled.md` | `## L-YYYY-MM-DD-NNN [active, confidence: high\|medium]` with Trigger, Rule, Source, Last reinforced | `learning-observer`, `/learn-from`, `verificator` | yes |
| `.claude/learnings/pending.md` | `## L-... [pending]` with Trigger, Candidate rule, Evidence, Suggested confidence, Promote criteria; plus `## CONFLICT:` blocks | `learning-observer` only | no |
| `.claude/learnings/retired.md` | retired rules with a retirement date and reason; never deleted | `/memory-review`, `/memory-prune` — user action only | yes |
| `.claude/learnings/corrections.jsonl` | append-only raw signal log with verbatim prompts | `learning-observer`, `/learn-from` | **no** |
| `.claude/learnings/post-push-queue.md` | `## Push` blocks; truncated after processing, never deleted | a post-push hook writes, the observer consumes | no |

Promotion threshold, verbatim from the source: **2 observations of the same pattern, or 1 explicit
teaching** (`docs/plugin-overview.md:145`, `docs/memory-extension-readme.md:429`). Explicit teaching
lands directly at `confidence: high`.

Signal detection: the prompt hook regex-classifies teaching, correction, and preference signals and
writes a flag file; the stop hook spawns the observer only when that flag exists.

### 4.12 Branch-sync

`scripts/branch-sync.sh` plus `agents/branch-sync.md`. Skips on the default branch or a detached
head. Fetches, merges the default remote branch into the current branch with an "ours" strategy,
and pushes. On merge failure it aborts and spawns the branch-sync agent headlessly. Logs every
outcome to `.claude/memory/branch-sync.log`. Launched from the session-start hook on every session
unless disabled by environment variable. Optional scheduling at 7am and 7pm via
`scripts/install-branch-sync-schedule.sh`, which supports macOS launch agents, Linux cron, and
Windows Task Scheduler under WSL or Git Bash, with a deterministic task name derived from a hash
of the repository path.

### 4.13 Adapters

Generated wholesale by `scripts/build-adapters.py`, which removes and rewrites the whole output
tree on every run and fails loudly if an expected substitution pattern is missing.

| | OpenCode | Codex | Devin |
|---|---|---|---|
| Skills | copied verbatim into the shared skills directory | same | same |
| Commands | native command files, argument placeholder preserved | converted to skills | converted to skills |
| Subagents | native agent files with per-tool booleans | per-agent TOML with developer instructions | **none — instructions run inline** |
| Hooks | a JS plugin bridging the shell scripts | native hooks file, exit 2 denies | native hooks file, no wrapper key, filtered to the supported event set |
| MCP | generated `mcp` block | generated server table entry, merge-sensitive | copy-paste snippet only |
| Shared runtime | scripts directory with broadened tool-name matchers and headless-CLI shim | same | same |

Dropped or degraded by design: slash commands become skills on two harnesses; the spawn tool call
becomes a harness shim note; the plugin-root path variable is rewritten; the turn cap in the legacy
parallel runner is dropped; all hook matchers widen to match everything and filtering moves into
the scripts; OpenCode gets no session-start or prompt-submit bridging; Devin gets no subagents and
no automated MCP install.

### 4.14 Schemas

All six, draft-07, `additionalProperties: false`, timestamps pinned to a strict UTC pattern.

| Schema | Validates | Required top-level fields |
|---|---|---|
| `codebase-index.schema.json` | the generated codebase index | `generatedAt, root, files, edges`. Summary text must not be model-generated — determinism requirement |
| `session-config.schema.json` | `docs/<story-id>/session.json` | `storyId, deployEnv, outputBackend, sessionCreated`. `outputBackend` enum `filesystem \| mcp \| database \| custom` |
| `story-graph.schema.json` | `docs/<story-id>/story-graph.json` | `storyId, updatedAt, artifacts, definitions, edges`. Edge kinds `sourced, contains, realizes, touches, testedBy, verifies, mentions` |
| `story-snapshot.schema.json` | `docs/<story-id>/story-snapshot.json` | `storyId, title, status, startedAt, updatedAt, phases`. Phase id enum includes both `03` and `3` |
| `task-graph.schema.json` | `docs/<story-id>/tasks/task-graph.json` | `storyId, generatedAt, taskCount, tasks`. Track ids must match an uppercase-letter pattern; digits rejected |
| `verification-result.schema.json` | `docs/<story-id>/verification-result.json` | `storyId, runAt, verdict, totals, findings, reportFile`. Finding ids `VF-NNN`; criterion ids copied verbatim from the spec, never renumbered |

### 4.15 Templates

| Template | Purpose | Filled output lands at |
|---|---|---|
| `templates/spec.md` | Functional and non-functional requirements, acceptance criteria, out of scope, open questions | `docs/<story-id>/requirements.md` |
| `templates/story.md` | User story with Given/When/Then scenarios carrying requirement ids | `docs/<story-id>/user-stories.md` |
| `templates/adr.md` | Context, Decision, Consequences, Alternatives Considered | `docs/<story-id>/adrs/adr-NNN.md` |
| `templates/rule.md` | Trigger, Rule, Rationale, Exceptions; status active/deprecated/retired | `.claude/memory/rules.md` |
| `templates/AGENTS.md` | A harness-agnostic drop-in for a repository connected to the cross-story memory server: always-rules, vocabulary, connection, session-start call sequence, question-to-tool routing, pointer-then-fetch reading, memory writing, degraded mode, end-of-session checklist | copied verbatim into a connected repo's root |

### 4.16 Scripts

| Script | Purpose | Writes |
|---|---|---|
| `scripts/lib/resolve-project-dir.sh` | Shared project-root resolver: environment variable, then git toplevel, then working directory | none by contract |
| `scripts/session-start.sh` | Build and emit the session context blob; background the sync and the scan | none directly |
| `scripts/user-prompt-submit.sh` | Classify the prompt, write the learning flag | `.claude/`, `.claude/.learning-pending` |
| `scripts/pre-tool-use.sh` | Branch guard plus shell-read guard over the index | none |
| `scripts/read-guard.sh` | Block index reads always; block whole-file reads of the four large artifacts during an active story and print the table of contents instead | none directly; the printed table-of-contents call rewrites `story-graph.json` |
| `scripts/post-tool-use.sh` | No-op, unregistered | none |
| `scripts/stop.sh` | Spawn the learning observer headlessly when a signal is pending | observer log; removes the learning flag |
| `scripts/branch-sync.sh` | Automated branch sync with escalation | the sync log; git state and the remote |
| `scripts/scan-codebase.py` | Deterministic incremental codebase indexer with `scan`, `query`, `show` verbs | the state directory, the index, a generated ignore marker |
| `scripts/mcp-index-server.py` | The MCP server in §4.9 | the index, via the scanner |
| `scripts/story-graph.py` | Build and query the per-story pointer map; `build`, `show`, `fetch`, `ids` | `story-graph.json` — note that `show`, `fetch`, and `ids` also rewrite it when stale |
| `scripts/story-state.py` | Own every write to the story snapshot; `init`, `phase`, `review`, `verify`, `epics`, `worktree`, `pr`, `status`, `mcp`, `show`, `next` | the story directory and `story-snapshot.json`, schema-validated before landing |
| `scripts/task-graph.py` | Own every write to the task graph; `task`, `track`, `show`, `ready`, `tracks`, `artifacts` | the file named by its argument, schema-validated |
| `scripts/validate-json.py` | Schema validator with a bundled draft-07 fallback; artifact-to-schema map; harness-owned manifests exempted | none |
| `scripts/build-adapters.py` | Generate the three adapter overlays | the whole output tree |
| `scripts/install-adapter.sh` | Copy an overlay into a target project, never overwriting the three merge-sensitive files | the target project tree, plus memory and learnings directories |
| `scripts/install-branch-sync-schedule.sh` | Install or uninstall the twice-daily sync schedule per platform | platform scheduler state, the log directory |
| `scripts/run-headless.sh` | Pick the first available harness CLI and exec it with a prompt and a tool allowlist | none |
| `scripts/claude-start.sh` | Session bootstrap wrapper; flags for review, curator-disable, clean, passthrough | the learnings directory; removes the learning flag under clean |
| `scripts/parallel-run.sh` | Legacy parallel track runner, marked deprecated for interactive sessions | track output dirs, logs, status files, worktrees, commits, merges, a summary file |

### 4.17 Test suite

Run by `tests/run_tests.sh` with an optional substring filter; `tests/harness.sh` provides the
assertion helpers and fixture builders; `tests/generate-report.sh` wraps the run and emits a
self-contained HTML report. Prerequisites are checked and the run aborts if any are missing.

| Suite | Locks in |
|---|---|
| `test_user_prompt_submit.sh` | Prompt classification into exactly three signal types and the flag file contents; neutral prompts write nothing; always exit 0 |
| `test_post_tool_use.sh` | State-changing tools append to the counter file; read-only tools do not; every call appends a tool record |
| `test_session_start.sh` | The continue response, the session-start event, and each injected context section including the pending-learnings nudge |
| `test_stop.sh` | Always returns continue; exactly one stop event; curator only when state changed and not already curated; observer only when the flag exists, then removes it; **must not** touch the extract-pending marker |
| `test_resolve_project_dir.sh` | The drifted-working-directory regression: resolution order, no stray directories in subdirectories, guards still correct from a drifted cwd, every hook sources the shared resolver |
| `test_branch_sync.sh` | Always exits 0 and logs a reason for each of the eight paths, including escalation on merge failure |
| `test_install_schedule.sh` | Path-hash determinism, OS detection, the missing-script prerequisite, and the task-name format |
| `test_skill_validation.sh` | Every skill has a name and description in its first five lines; the reference files exist; no legacy sentinel lines; the named agents and commands exist; the extract-pending marker is handled |
| `test_new_commands.sh` | Content contracts for four commands plus a check against an installed marketplace copy |
| `test_build_adapters.sh` | Generator exits clean; per-harness structure and skill counts; syntax check on every generated script; the critical transforms; a live branch-guard run through a ported script |
| `test_command_references.sh` | Every cross-file reference inside the command tree resolves; the version stamp matches the manifest; the pointer-then-fetch contract holds — no whole-file read of the four large artifacts anywhere |
| `test_sdd_state.sh` | The largest suite: story-graph determinism, section fetch, scoped ids, freshness, corruption refusal; task-graph track derivation and the letter-id rule; scanner determinism and every query filter; both read guards |
| `test_mcp_index_server.sh` | The server over stdio: initialize, tool listing, notification handling, every filter, the hard limit, hit and miss on file lookup, auto-build on a missing index, unknown tool and unknown method errors, byte-identical output on an unchanged index |

Isolation discipline: every suite works inside a temporary directory with a cleanup trap and uses
path-prepended fake binaries, so no real harness CLI is ever invoked.

---

## 5. Isaac Requirements — Model layer

### 5.1 Model abstraction (the governing requirement)

**MR-1.** The model name MUST appear in exactly one configuration value. No command, phase,
protocol, rule, setup module, skill, agent, script, or hook may contain a model name literal.

In `build-agents-sdd` the model name appears in 26 places across 22 files: 20 frontmatter
`model:` fields (14 skills, 6 agents), the per-phase table in the orchestrator, the spawning
module's prose, two phase files, the context-budget command, the cost-report pricing table, and the
sweep skill's implementer parameter. Isaac replaces all of them with references to named tiers
resolved from config at runtime.

**MR-2.** Isaac defines a tier vocabulary that maps to concrete model identifiers in one config
file. The source uses three tiers — a reasoning tier, a default tier, and a cheap mechanical tier.
Isaac keeps three tier *names* and resolves all three through JEV. `[JEV: whether JEV exposes
distinct capability or cost tiers at all, and what their identifiers are, is unverified. If JEV is
a single undifferentiated model, all three tiers resolve to the same identifier and the tier
vocabulary becomes a no-op that preserves the file structure.]`

**MR-3.** The per-spawn override MUST remain a single parameter on the spawn call, set only where
the phase table says so, exactly as the spawning module specifies. Everywhere else the spawn
inherits the session model.

**MR-4.** An environment variable MUST be able to force one tier for every spawn, overriding both
the phase table and the config, matching the source's escape hatch.

**MR-5.** A skill's frontmatter `model:` field MUST continue to have **no effect** on a generic
spawn. The source states this explicitly and a test depends on the behaviour. Isaac keeps the
fields for documentation value and keeps them inert. `[uncertain: skills/*/SKILL.md:4 declare a
model that commands/setup/agent-spawning-model.md:32-34 says is never consulted — the fields are
documentation, not configuration, and Isaac must not silently start honouring them.]`

### 5.2 Tool-call speed

**MR-6.** Phase 6 MUST continue to spawn every track agent as a background call in a **single**
orchestrator message so the tracks run concurrently, then commit each track sequentially as agents
return. Sequential commits prevent index corruption; that constraint is independent of the model.
`[JEV: the maximum number of concurrent agent spawns JEV supports in one message is unverified.
If it is lower than the track count a story produces, the orchestrator needs a batching rule that
does not exist in build-agents-sdd.]`

**MR-7.** Phases 2 and 3 MUST continue to spawn as two parallel agents in one message; Phase 8a and
8b MUST continue to spawn as two parallel agents; council mode MUST continue to spawn three.

**MR-8.** The context-guard threshold MUST remain a single environment variable with a documented
default, not a hardcoded window fraction. The source default assumes a large window and documents
the smaller-window override inline. `[JEV: JEV's context window size is unverified, so the correct
default threshold cannot be set from this repository.]`

**MR-9.** The context-budget denominator used by the context audit command MUST be read from config,
not written into the command text. `[JEV: JEV's context window size is unverified.]`

**MR-10.** The cost-report pricing table MUST move out of the skill body into a config file keyed by
model identifier, and MUST tolerate an unknown identifier by reporting token counts with cost
omitted rather than by failing. `[JEV: JEV's input, output, cache-write, and cache-read prices are
unverified.]`

**MR-11.** The cost report's model-change detection MUST be preserved: when a phase runs under more
than one model identifier, the report MUST say so. This is what lets an auditor confirm a phase did
not drift mid-run.

**MR-12.** The sweep skill's implementer-model parameter MUST keep its name and keep defaulting to
the cheap tier, resolved through the same config. `[JEV: whether a cheaper implementer tier exists
under JEV is unverified.]`

**MR-13.** Isaac MUST NOT specify a thinking budget, effort level, or reasoning-depth instruction
anywhere. `build-agents-sdd` specifies none and Isaac adds none. `[JEV: whether JEV exposes an
effort or thinking-budget parameter, and whether setting it is beneficial, is unverified.]`

**MR-14.** Isaac MUST record, per phase, the wall-clock time and the tool-call count, so goal G2 is
measurable rather than asserted. The cost report already parses tool calls from the session log;
Isaac extends its output with elapsed time per phase. This is a reporting change inside an existing
artifact, not a new feature.

### 5.3 Headless spawns

**MR-15.** The headless shim MUST keep selecting a CLI by name from the environment with a
documented override variable, and MUST NOT name a model on the command line. The source names no
model in any script; Isaac keeps that property.

---

## 6. Isaac Requirements — MCP-only file I/O

### 6.1 The write server

Isaac ships a second MCP server alongside `sdd-index`. Working name for this document:
`isaac-write`. `[uncertain: build-agents-sdd never names an MCP write server — the mcp backend in
commands/setup/output-write-adapter.md:24-30 expects the operator to supply a server and tool name.
Isaac ships its own, so this name is new and is the one place a new identifier is unavoidable.]`

Transport, framing, and error conventions mirror `sdd-index`: JSON-RPC 2.0 over stdio,
newline-delimited, standard library only, root resolved once from the project-directory environment
variable with a working-directory fallback, tool failures returned as a tool result with an error
flag rather than a protocol error.

### 6.2 Tool definitions

All paths are repository-relative. An absolute path, a path escaping the repository root via `..`,
or a symlink whose target escapes the root is rejected.

| Tool | Parameters | Returns | Error cases |
|---|---|---|---|
| `create_file` | `path` (string, required), `content` (string, required), `overwrite` (boolean, default false), `mkdirs` (boolean, default true) | `{path, bytes, created: true, sha256}` | path escapes root; parent missing and `mkdirs` false; file exists and `overwrite` false; path is a directory; write denied |
| `update_file` | `path` (required) and exactly one of: `content` (whole-file replace), or `old_string` + `new_string` + optional `replace_all` (default false), or `start_line` + `end_line` + `content` (range replace). Optional `expected_sha256` for optimistic concurrency | `{path, bytes, replacements, sha256}` | file missing; more than one mode supplied; `old_string` not found; `old_string` not unique and `replace_all` false; line range out of bounds; `expected_sha256` mismatch |
| `delete_file` | `path` (required), `recursive` (boolean, default false, required for a non-empty directory) | `{path, deleted: true, entries_removed}` | path missing; directory non-empty and `recursive` false; path escapes root; path is protected (see 6.4) |
| `move_file` | `from` (required), `to` (required), `overwrite` (boolean, default false), `mkdirs` (boolean, default true) | `{from, to, moved: true}` | source missing; destination exists and `overwrite` false; either path escapes root; destination parent missing and `mkdirs` false |
| `mkdir` | `path` (required), `parents` (boolean, default true) | `{path, created: true \| false}` | path escapes root; a non-directory exists at the path; parent missing and `parents` false |
| `append_file` | `path` (required), `content` (required), `create` (boolean, default true), `newline` (boolean, default true — ensure the existing content ends with a newline before appending) | `{path, bytes_appended, bytes_total, sha256}` | file missing and `create` false; path escapes root; path is a directory |

Cross-cutting requirements:

- **WR-1.** Every tool is idempotent-safe to retry: a repeated `create_file` with identical content
  and `overwrite: true` produces the same result and the same hash.
- **WR-2.** Every write is atomic: write to a temporary file in the same directory, then rename.
  A partial write must never be observable.
- **WR-3.** Every tool returns the resulting content hash so the caller can prove what landed.
- **WR-4.** Text is UTF-8. A binary payload is out of scope; the pipeline writes only text
  artifacts. The one exception is the debug command's screenshots, which the browser automation MCP
  writes directly — see 6.5.
- **WR-5.** The server logs every write to an append-only audit line: timestamp, tool, path, bytes,
  hash. The log itself is written by the server, not by a tool call, and is the one file exempt from
  the "all writes go through a tool" rule.

### 6.3 Enforcement

**WR-6.** A `PreToolUse` hook denies, with the blocking exit code, every call to `Write`, `Edit`,
`MultiEdit`, and `NotebookEdit`. The deny message names the equivalent write-server tool.

**WR-7.** The same hook inspects `Bash` commands and denies any that writes a file: a redirect
(`>`, `>>`) whose target is not a null device, `tee` without a null target, `sed -i`, `cp`, `mv`,
`mkdir`, `touch`, `rm`, `install`, `dd`, and heredoc-to-file constructs.

**WR-8.** The existing branch guard's exemptions are **narrowed, not reused**. In
`build-agents-sdd` the branch guard exempts writes under the harness directory and temporary
directories and only applies on protected branches. Isaac's write ban applies on **every** branch,
so it is a separate rule in the same script, evaluated before the branch check.

**WR-9.** Explicit shell-write exemptions, each justified:

| Exempt | Why | Consequence |
|---|---|---|
| `git` itself (commit, merge, checkout, worktree, branch, push) | Git writes the object store and the working tree as an intrinsic part of its operation. Routing git through a file-write MCP is not possible without reimplementing git. | Git remains the one process allowed to mutate tracked files outside the write server. The staging guard and branch guard continue to constrain it. |
| Writes to the temporary directory | Scratch files that never enter the repository and are not artifacts. | Kept, and the deny rule checks the target path. |
| The null device | Output suppression, not a write. | Kept. |

**WR-10.** Test fixtures are exempt only inside the test harness's temporary directories, and the
test suite MUST include a case asserting that a write attempt outside those directories is denied.

### 6.4 Protected paths

`delete_file` and `move_file` refuse, unconditionally, to operate on: the git directory, the
project manifest, the MCP manifest, the hooks manifest, anything under `schemas/`, and any file the
caller has not first read in the same session. `[uncertain: build-agents-sdd has no equivalent
protection — the archive command's own rule "Never delete story artifacts — only move them" is the
closest analogue and is prose, not enforcement. Isaac makes it enforcement.]`

### 6.5 Feature-by-feature write mapping

Every feature from §4 that writes a file, and the tool it uses. Sorted by owner.

**Commands**

| Feature | Files written | Isaac tool |
|---|---|---|
| `/build-feature` | story directory, loop-contracts directory | `mkdir` |
| `/build-feature` | `.claude/.context-extract-pending` | `create_file` (overwrite) |
| `/quick-feature` | `scope.md`, `implementation-notes.md`, `test-results.md` | `create_file` |
| `/archive` | create the archive directory | `mkdir` |
| `/archive` | move the story tree | `move_file` |
| `/archive` | append a State line to project memory | `update_file` (range or string replace on the `## State` section) |
| `/context-report` | the generator script, the HTML report | `create_file` |
| `/current-context` | — read-only | none |
| `/debug-browser` | five markdown evidence files | `create_file` |
| `/debug-browser` | two screenshots | written by the browser automation MCP directly — already an MCP write, no change |
| `/gate-verify` | — writes nothing by contract | none |
| `/learn-from` | a distilled entry | `append_file` |
| `/learn-from` | a corrections line | `append_file` |
| `/match-figma-design` | components, tokens, assets, manifest | `create_file`, `update_file` |
| `/memory-prune` | rewrite four memory files, write retired | `update_file` (whole-file), `append_file` |
| `/memory-recall` | — read-only | none |
| `/memory-review` | update distilled and pending, append retired | `update_file`, `append_file` |
| `/memory-save` | append to project memory or decisions | `append_file` or `update_file` |
| `/pause` | resume context (overwritten) | `create_file` (overwrite) |
| `/pause` | the curated marker | `create_file` |
| `/pause` | remove the learning flag | `delete_file` |
| `/review-plan` | the plan file, on request | `create_file` |
| `/review-pr` | the review file, on request | `create_file` |
| `/stop` | move resume context into the archive directory | `mkdir` + `move_file` |
| `/stop` | delete the extract-pending marker | `delete_file` |

**Phases** (the artifact each phase contracts)

| Phase | Files | Isaac tool |
|---|---|---|
| 1 | `validated-idea.md`, three council files | `create_file` |
| 1 (tracker mode) | append two patterns to the git exclude file | `append_file` |
| 2+3 | `requirements.md`, `user-stories.md`, `ux-wireframes.md`, `adrs/` | `mkdir` + `create_file` |
| 03 | `epics/_epics-proposal.md`, `epics/_epics.md`, epic directories | `create_file`, `mkdir` |
| 4 | `architecture.md`, `implementation-plan.md` | `create_file` |
| 4 | append to decisions memory | `append_file` |
| 4.5 | `codebase-context.md`, `tasks/task_NN.md` | `mkdir` + `create_file` |
| 5 | branches and worktrees | git, exempt under WR-9 |
| 6 | source files and unit tests | `create_file`, `update_file` |
| 6 | per-track notes | `create_file` |
| 6 | append to project memory | `append_file` |
| 6.5 | `verification-report.md` (overwritten, never appended) | `create_file` (overwrite) |
| 7 | `test-plan.md`, `test-results.md` | `create_file` |
| 8a/8b | issue files and two reports | `mkdir` + `create_file` |
| 8c | issue-file frontmatter updates, round summary, source fixes | `update_file`, `create_file` |
| 9 | the extract-pending marker | `create_file` |

**Skills** — every skill in §4.6 that writes uses the same mapping: a new markdown artifact is
`create_file`; an edit to an existing artifact is `update_file`; a memory append is `append_file`;
a directory is `mkdir`. The one skill that writes an agent definition at runtime uses
`create_file`. The one skill that writes nothing stays writing nothing.

**Agents** — the curator, observer, extractor, and verificator all write memory or learnings files
via `append_file` and `update_file`. The branch-sync agent writes only its log line, which is
`append_file`, and its git operations are exempt. The council personas write three files via
`create_file`.

### 6.6 Hook scripts that write from shell — disposition

This is the hard case. Six shell and python scripts write files today, and they run *inside* the
hook lifecycle, where calling an MCP tool is not available.

| Script | Writes | Disposition | Why |
|---|---|---|---|
| `scripts/user-prompt-submit.sh` | the learning flag file and its parent directory | **Exempt** | Runs as a hook with no MCP client. The file is a single-token internal marker, not an artifact. An MCP round trip on every prompt would add latency to the hot path for no reviewable benefit. |
| `scripts/stop.sh` | the observer log; removes the learning flag | **Exempt** | Same lifecycle constraint. The log is diagnostic output, not an artifact. |
| `scripts/branch-sync.sh` | the sync log; git state; the remote | **Exempt** | Runs detached from any session, often from a scheduler with no harness present. Its git operations are exempt under WR-9 regardless. |
| `scripts/scan-codebase.py` | the state directory, the index, the ignore marker | **Exempt** | The index is generated derived state, gitignored, and rebuilt deterministically. It is also written by the `refresh_index` MCP tool already, which is the reviewable path; the CLI path exists for hooks and CI. |
| `scripts/story-state.py` | the story snapshot | **Route through MCP** | This is pipeline state of record. It is schema-validated, it is the resume point, and every phase writes it. It belongs on the audited path. Isaac exposes the same verbs as MCP tools on the write server and keeps the CLI as a thin client of those tools, so the command lines in every phase file stay byte-identical. |
| `scripts/story-graph.py` | the story graph | **Route through MCP**, with one carve-out | Same reasoning. The carve-out: the read guard invokes the graph's `show` verb, which rewrites the graph as a side effect, from inside a `PreToolUse` hook with no MCP client. Isaac splits that verb so the hook path is strictly read-only and the rewrite happens on the next pipeline call. |
| `scripts/task-graph.py` | the task graph | **Route through MCP** | Same reasoning as the snapshot. |
| `scripts/build-adapters.py` | the whole adapter output tree | **Exempt** | A build-time generator run by a developer or CI, not by an agent during a pipeline run. It also removes and rewrites its whole output tree, which no per-file tool models well. |
| `scripts/install-adapter.sh` | a target project tree | **Exempt** | An installer run by a human against a different repository. |
| `scripts/install-branch-sync-schedule.sh` | platform scheduler state | **Exempt** | Writes outside the repository into OS-level scheduler config. Out of scope for a repository-scoped file server. |
| `scripts/parallel-run.sh` | track logs, status files, worktrees, a summary | **Exempt**, and deprecated | Marked deprecated for interactive sessions in the source. Isaac keeps it for CI parity only and does not route it. |
| `scripts/claude-start.sh` | the learnings directory; removes the flag under clean | **Exempt** | A pre-session wrapper that runs before any harness exists. |

**WR-11.** Every exempt script MUST carry a comment naming this section and the reason, so the
exemption is discoverable at the write site rather than only in this document.

**WR-12.** The three routed helpers MUST keep their exact command-line interfaces. Phase files,
protocols, and skills invoke them by command line dozens of times; changing the interface would
violate NG1 in spirit even though it renames nothing user-facing.

---

## 7. Functional Requirements

Each is traceable to a §4 entry and has a binary acceptance criterion.

| ID | Requirement | Traces to | Acceptance criterion |
|---|---|---|---|
| FR-001 | Isaac ships 17 command files with the names in §4.1 | §4.1 | `ls commands/*.md` returns exactly the 17 names, spelled identically to the source |
| FR-002 | `/build-feature` parses ` resume`, `--force`, `--council`, `--step`, `--loop` and sets the five corresponding variables | §4.1 flags | Each flag, supplied alone, sets its variable and is stripped from the description |
| FR-003 | `/build-feature` halts with the verbatim stale-plugin message when any of the four companion directories is missing | §4.1, `commands/build-feature.md:48-73` | Removing one directory produces the message and no pipeline run |
| FR-004 | The staging guard blocks any run when the index is dirty | §4.5 staging module, core rule 1 | A staged file produces the blocked message and exit before any write |
| FR-005 | Isaac ships 11 phase files with the names in §4.2 | §4.2 | `ls commands/phases/*.md` matches the source list exactly |
| FR-006 | Every phase gate runs the context guard and acts on its output | core rule 8 | A story directory above the threshold produces the triggered branch and a paused snapshot |
| FR-007 | Phase 03 always asks, regardless of the planner's recommendation | §4.2 | Both a split and a no-split recommendation produce the three-option prompt |
| FR-008 | Accepting a Phase 03 split stops the session and never auto-runs an epic pipeline | §4.2 | After accept, no Phase 4 dispatch occurs and one command line per epic is printed |
| FR-009 | Epic mode skips Phases 1, 2+3, and 03 and resolves the six shared inputs against the parent | §4.3 epic-scoping | A story id containing the epic separator starts at Phase 4 with parent-resolved inputs |
| FR-010 | Phase 6 spawns every track in one message and commits sequentially | §4.2, MR-6 | With three tracks, one orchestrator message contains three spawn calls and three commits occur in sequence |
| FR-011 | Phase 6.5 spawns the registered verificator agent, not a skill name | §4.2, §4.7 | The spawn names the registered agent type; a skill name in that slot fails |
| FR-012 | Phase 8 runs at most 3 fix rounds then asks the user | §4.2 | A fourth round never starts without the three-option prompt |
| FR-013 | Phase 9 writes the extract-pending marker containing the story id | §4.2 | After a successful PR, the marker file exists and contains the story id |
| FR-014 | Isaac ships 6 protocol files with the names in §4.3 | §4.3 | Directory listing matches |
| FR-015 | Isaac ships both rule files with all 16 rules verbatim | §4.4 | Text diff against the source rule files is empty apart from the product name |
| FR-016 | Isaac ships 8 setup modules with the names in §4.5 | §4.5 | Directory listing matches |
| FR-017 | Isaac ships 21 skill directories with the names in §4.6 | §4.6 | Directory listing matches; every one has a name and description in the first five lines |
| FR-018 | Isaac ships 6 agent files with the names in §4.7 | §4.7 | Directory listing matches |
| FR-019 | Only the 6 registered agent names are valid spawn types | §4.5 spawning module | A spawn using a skill name as the type fails with a type-not-found error |
| FR-020 | Isaac registers 5 hook entries across 4 events | §4.8 | The hooks manifest lists exactly those entries |
| FR-021 | The read guard blocks reading the codebase index on every branch | §4.8 | A read of the index path returns the blocking exit code and a message naming the query tools |
| FR-022 | The read guard blocks a whole-file read of the four large artifacts while the story is active and prints the table of contents | §4.8 | A read of an over-threshold `requirements.md` in an active story is blocked with a section list |
| FR-023 | The branch guard blocks write tools and history-mutating git on protected branches | §4.8 | On the default branch, an edit attempt is blocked; a checkout is not |
| FR-024 | `sdd-index` exposes exactly the four tools in §4.9 with the stated schemas | §4.9 | Tool listing returns four names; each rejects an unknown parameter |
| FR-025 | `find_files` clamps the limit to at most 50 | §4.9 | A request for 500 returns at most 50 rows |
| FR-026 | `find_files` and `show_file` build the index on first use and signal it | §4.9 | With no index present, the first call succeeds and reports the build flag |
| FR-027 | `refresh_index` is the only tool that rescans on demand | §4.9 | Repeated read calls on an unchanged repo produce byte-identical output |
| FR-028 | Memory files use the structures in §4.10 | §4.10 | A curator run produces entries matching each file's heading contract |
| FR-029 | The project-memory State section is replaced, not appended | §4.10 | After two curator runs, the State section contains one current block |
| FR-030 | Learning promotion requires 2 observations or 1 explicit teaching | §4.11 | A single non-teaching observation stays pending; an explicit teaching lands active at high confidence |
| FR-031 | Background agents write only inside their declared scope | §4.7 | A curator attempt to write outside the memory directory is refused |
| FR-032 | `gate-verify` writes nothing | §4.1 | A full run leaves the working tree unchanged |
| FR-033 | `current-context` and `memory-recall` write nothing | §4.1 | Same |
| FR-034 | Branch sync always exits 0 and logs a reason on every path | §4.12 | Each of the eight paths produces one log line and exit 0 |
| FR-035 | The adapter generator produces three overlays and fails on a missing substitution | §4.13 | Removing an expected pattern fails the build loudly |
| FR-036 | No generated overlay file outside the scripts directory contains the plugin-root path variable | §4.13 | The generator's leak check passes |
| FR-037 | Every SDD-authored JSON validates against its schema before it lands | §4.14 | A malformed write is refused with a non-zero exit and the previous file is unchanged |
| FR-038 | Track ids are uppercase letters; digits are rejected by the schema | §4.14 | A numeric track id fails validation |
| FR-039 | `taskCount` equals the task array length | §4.14 | A mismatch fails validation |
| FR-040 | Verification finding ids and criterion ids are never renumbered | §4.14 | Criterion ids in the result match the spec verbatim |
| FR-041 | Isaac ships the five templates in §4.15 | §4.15 | Directory listing matches |
| FR-042 | The test suite covers every area in §4.17 | §4.17 | Every suite named in §4.17 exists and passes |
| FR-043 | The model name appears in exactly one config value | MR-1 | A repository-wide grep for model identifiers matches only the config file |
| FR-044 | A skill's frontmatter model field has no effect on a generic spawn | MR-5 | Changing it produces no change in the model used |
| FR-045 | An environment variable forces one tier for every spawn | MR-4 | Setting it overrides both the phase table and the config |
| FR-046 | The context-guard threshold is an environment variable with a config default | MR-8 | Setting it changes the trigger point |
| FR-047 | The cost report tolerates an unknown model identifier | MR-10 | An unknown identifier yields token counts with cost omitted, not an error |
| FR-048 | The cost report flags a mid-phase model change | MR-11 | Two identifiers in one phase produce the model-change line |
| FR-049 | Per-phase elapsed time and tool-call count appear in the context report | MR-14 | Both values are present for every completed phase |
| FR-050 | `isaac-write` exposes exactly the six tools in §6.2 | §6.2 | Tool listing returns six names; each rejects an unknown parameter |
| FR-051 | Every write tool is atomic | WR-2 | A killed write leaves the original file intact |
| FR-052 | Every write tool returns the resulting content hash | WR-3 | Every successful call includes a hash field |
| FR-053 | A path escaping the repository root is rejected by every tool | §6.2 | A parent-directory traversal and an absolute path both fail |
| FR-054 | `update_file` in string mode fails when the target is not unique and replace-all is false | §6.2 | Two occurrences without the flag produce an error and no write |
| FR-055 | `update_file` honours an expected-hash precondition | §6.2 | A stale hash fails with no write |
| FR-056 | The hook denies every native write tool on every branch | WR-6, WR-8 | Each of the four tools is blocked on a feature branch as well as the default branch |
| FR-057 | The hook denies file-writing shell commands | WR-7 | Each listed construct is blocked; a null-device redirect is allowed |
| FR-058 | Git operations remain permitted | WR-9 | A commit on a feature branch succeeds |
| FR-059 | Delete and move refuse the protected paths | §6.4 | Each protected path produces a refusal |
| FR-060 | Every exempt script carries an in-file comment naming its exemption | WR-11 | Each of the exempt scripts contains the comment |
| FR-061 | The three routed state helpers keep identical command-line interfaces | WR-12 | Every command line copied verbatim from a source phase file executes unchanged |
| FR-062 | The read-guard path never writes | §6.6 carve-out | A blocked read leaves the story graph file unmodified |
| FR-063 | The write server appends one audit line per successful write | WR-5 | Ten writes produce ten audit lines with tool, path, bytes, and hash |
| FR-064 | The session-start hook injects the seven context sections in order | §4.8 | Each section appears, and an empty source file omits its section |
| FR-065 | The stop hook never touches the extract-pending marker | §4.8, §4.17 | The marker survives a stop-hook run |

---

## 8. Non-Functional Requirements

### 8.1 Versioning

**NFR-1.** The plugin manifest version MUST be bumped on every commit touching commands, skills,
agents, protocols, setup, phases, or rules. The harness caches the installed plugin by that string;
an unchanged version makes an update silently no-op, which is the root cause of the stale-scaffolding
failure the orchestrator's self-check reports. Source: `CLAUDE.md` rule 6.

**NFR-2.** After a bump, the operator MUST be told to run the marketplace update, the plugin update,
and the reload, in that order. The reload is required and is not implied by the updates.

**NFR-3.** The version stamp inside the orchestrator command file MUST equal the manifest version.
A test enforces this today and MUST continue to.

**NFR-4.** Every additional manifest that carries a version — the native plugin manifests for other
harnesses — MUST be kept in sync. The source notes there is no automatic sync. Isaac SHOULD add a
single test asserting all version strings match, since that is enforcement of an existing rule
rather than a new feature. `[uncertain: docs/security-and-extending.md:20 names two bump sites,
CLAUDE.md rule 6 names one, docs/cross-platform.md:151-152 names two more — four sites total, no
single authoritative list.]`

### 8.2 Docs ship with the version

**NFR-5.** A version bump is incomplete until the guides describe that version's behaviour, in the
**same commit**: a new changelog section at the top in Keep a Changelog format matching the bumped
version exactly and describing the behaviour change and the files that carry it; every guide whose
content the change invalidates; and every stale version string found by grepping the previous
version across the readme, docs, commands, and skills.

**NFR-6.** Before a bump is declared done, the commit MUST state which guides were updated and
which were checked and needed no change. Silence reads as forgotten.

**NFR-7.** When the published team guide changes, the summary MUST say so, because it mirrors a
separately published branch.

### 8.3 Cross-harness adapter parity

**NFR-8.** The canonical source is the Isaac plugin tree. Overlays are generated and MUST NOT be
hand-edited.

**NFR-9.** The generator MUST fail loudly when an expected substitution pattern is missing, rather
than emitting a silently wrong overlay.

**NFR-10.** Every script the hooks manifest references MUST be present in every overlay that
registers those hooks. The source violates this: two overlays register a hook pointing at a script
the generator never copies. Isaac MUST ship a generator check and a test asserting that every hook
command path exists in the emitted tree.

**NFR-11.** Known degradations MUST be documented, not hidden: no user-defined subagents on one
harness, no prompt-submit equivalent on another, varying tool names on a third, and manual MCP
registration on one.

**NFR-12.** The memory layout MUST be byte-identical across every harness so one project can be
worked from any of them without splitting its memory.

### 8.4 Test coverage

**NFR-13.** Isaac's suite MUST cover every area the source suite covers — the thirteen suites in
§4.17 — plus new suites for the write server and the write enforcement.

**NFR-14.** The suite MUST be self-contained: no external test framework, no package install. Every
suite works inside a temporary directory with a cleanup trap and uses path-prepended fake binaries
so no real harness CLI is invoked.

**NFR-15.** No suite may depend on a machine-local install path. The source has one suite that
asserts against a path under the user's home directory, which makes the whole run environment
dependent. Isaac MUST make that suite skip cleanly when the path is absent.

**NFR-16.** The suite MUST assert determinism where the source does: the codebase index and the
story graph produce byte-identical output on an unchanged repository, and the index server returns
byte-identical results across two calls.

**NFR-17.** The suite MUST assert the pointer-then-fetch contract: no instruction anywhere in
commands, skills, or agents tells an agent to read one of the four large artifacts whole, and no
legacy sentinel line remains.

### 8.5 Determinism and safety

**NFR-18.** The codebase index MUST remain deterministic, incremental, model-free, network-free,
and MUST never contain file contents. Secrets, keys, certificates, and dependency directories are
hard-denied from the scan.

**NFR-19.** State JSON MUST never be hand-edited. The three helpers own every write and validate
before saving.

**NFR-20.** Prose MUST NOT be wrapped in a JSON string. A value past roughly two hundred characters
belongs in markdown, with the JSON linking to it by id or path.

**NFR-21.** Rendered HTML is output only. It is never a source of truth and is never read back by
an agent as input.

---

## 9. Migration Notes

### What a `build-agents-sdd` user does to move to Isaac

1. **Install Isaac** in place of, or alongside, the existing plugin. Both can be installed at once;
   the command names collide, so running both is not supported. Uninstall the old one first.
2. **Set the model config value.** This is the one mandatory configuration step and has no default
   that can be inferred from this repository. `[JEV: the identifier string to put here is
   unverified.]`
3. **Register the write server.** Isaac's MCP manifest registers both `sdd-index` and
   `isaac-write`. On a harness that auto-loads the plugin-root manifest this is automatic; on the
   others it follows the same per-harness table as the index server.
4. **Re-run the adapter installer** for any non-native harness, so the overlays carry Isaac's
   scripts and the write server's registration.
5. **Re-run the schedule installer** if the twice-daily branch sync was installed, because the task
   name is derived from the repository path and the script path changes.
6. **Expect the first run to rebuild the codebase index.** The index is derived state and lives in a
   product-named state directory. `[uncertain: the state directory name is derived from the product
   name (.build-agents-sdd/) in scripts/scan-codebase.py and docs/CONVENTIONS.md:52. NG1 forbids
   renaming user-facing names; a hidden state directory is arguably not user-facing. If Isaac renames
   it, existing indexes are orphaned and rebuild on first use, which is harmless. If Isaac keeps the
   old name, the directory name no longer matches the product. This is a decision, not a fact.]`

### What carries over unchanged

| Carried over | Why it is safe |
|---|---|
| `.claude/memory/project-memory.md`, `decisions.md`, `patterns.md`, `rules.md`, `skills.md`, `tokens-consumed.md` | Structures are identical; Isaac's curator and extractor append in the same format |
| `.claude/learnings/distilled.md`, `pending.md`, `retired.md`, `corrections.jsonl`, `post-push-queue.md` | Same entry formats and the same promotion threshold |
| `.claude/memory/session-log/` and its markers | Same event shapes |
| Every `docs/<story-id>/` tree, including in-flight stories | Every schema is unchanged, so `story-snapshot.json` resumes and the graph rebuilds |
| `docs/_archived/` | Untouched |
| `knowledge/` files | Read the same way |
| `reports/` | Untouched; new reports gain per-phase timing |
| `scan.config.json` | Same scanner, same config resolution order |

### What does not carry over

- **The `database` and `custom` output backends.** A session config naming either fails validation
  in Isaac, because the enum narrows to `mcp`. A migration note in the release must say so.
  `[uncertain: schemas/session-config.schema.json permits four backends; Isaac's MCP-only mandate
  permits one. Narrowing an enum is a breaking change to an existing artifact, which sits against
  NG1's spirit. The alternative is keeping the enum wide and rejecting the other three at runtime.]`
- **Any local tooling that wrote into `docs/<story-id>/` directly.** It is now denied by the hook.

### Resuming an in-flight story across the move

A story paused under `build-agents-sdd` resumes under Isaac with the same command shape. The resume
protocol asks the state file for the next phase, regenerates the index, and continues. Phases
already complete are not re-run.

---

## 10. Open Questions

### 10.1 JEV markers, collected

| Marker | Where it appears | What must be confirmed |
|---|---|---|
| JEV-1 | G2 | The mechanism by which JEV speeds up tool calls: batching, lower per-call latency, a higher parallel ceiling, or something else. Without this, G2 has no design implication. |
| JEV-2 | MR-2 | Whether JEV exposes distinct capability or cost tiers, and their identifiers. If not, the three tier names collapse to one identifier. |
| JEV-3 | MR-6 | The maximum concurrent agent spawns in one message. If lower than a typical track count, a batching rule is needed that the source has no precedent for. |
| JEV-4 | MR-8 | JEV's context window size, which sets the correct context-guard default. |
| JEV-5 | MR-9 | JEV's context window size, which sets the context-audit denominator. |
| JEV-6 | MR-10 | JEV's input, output, cache-write, and cache-read prices. |
| JEV-7 | MR-12 | Whether a cheaper implementer tier exists for the sweep skill's split. |
| JEV-8 | MR-13 | Whether JEV exposes an effort or thinking-budget parameter, and whether setting it helps or hurts. |
| JEV-9 | Migration step 2 | The exact model identifier string to write into the config. |

### 10.2 Source contradictions

Each needs an authoritative answer before the corresponding Isaac file is written.

| # | Contradiction | Sides |
|---|---|---|
| C-1 | **Phase count.** Four different numbers for the same pipeline. | "9-phase": `README.md:5`, `.claude-plugin/plugin.json:5`, `.claude-plugin/marketplace.json:11`, `CLAUDE.md`, `docs/loop-mode/LOOP_MODE_FEATURE.md:218`. "10-phase": `commands/setup/command-fit-check.md:39`, `commands/setup/final-report-template.md:17`, `docs/plugin-overview.md:153`, `docs/plugin-guide.md:88`. "14 phases": `commands/setup/session-initialization.md:35`. The phase table itself has 11 rows: `commands/build-feature.md:160-172`. |
| C-2 | **Base branch name.** The story branch created at step 0c is never matched by the Phase 5 gate glob or by the archive cleanup glob, so the main story branch is never cleaned up. | `commands/setup/staging-guard-and-worktree.md:46` vs `commands/phases/05-branch-setup.md:34` and `commands/archive.md:76` |
| C-3 | **Track branch name.** Phase 6 records a branch name Phase 5 never created. | `commands/phases/05-branch-setup.md:24` vs `commands/phases/06-implementation.md:138` |
| C-4 | **Worktree ownership.** Two owners and two locations for the same per-track worktree. | `commands/phases/05-branch-setup.md:24-26` vs `commands/phases/06-implementation.md:43-50,123`; further inconsistent with `skills/using-git-worktrees/SKILL.md` (human-gated root) and `skills/feedback-triage/SKILL.md` (a third location) |
| C-5 | **Secret-file blocking.** The security guide claims hooks hard-block reading secret files. Neither guard script contains any such logic; a grep for the patterns returns only the shebang lines. | `docs/security-and-extending.md:9` vs `scripts/pre-tool-use.sh`, `scripts/read-guard.sh` |
| C-6 | **Memory curator trigger.** One set of docs says the curator fires on the stop hook; another says no hook writes a session log and the curator runs only where a person can see it. | `docs/plugin-overview.md:142-145`, `docs/plugin-guide.md:122,134` vs `docs/memory-system.md:33-37`, `CHANGELOG.md:81-83` |
| C-7 | **Session JSONL logging.** Docs describe a session log that the changelog says was removed and that the memory doc says no hook writes — yet the tests assert hooks write it. | `docs/plugin-guide.md:909,918` and `tests/test_session_start.sh` vs `docs/memory-system.md:33`, `CHANGELOG.md:292-294` |
| C-8 | **Hook inventory.** One doc says four hooks including a tool-logging hook; the manifest registers five entries over four events and has no tool-logging hook. | `docs/install.md:29`, `docs/plugin-guide.md:54` vs `hooks/hooks.json`, `docs/plugin-structure.md:72` |
| C-9 | **Counts of skills, commands, agents.** Skills stated as 19, 21, and 17; actual 21. Commands stated as 16, 17, and 11; actual 17. Agents stated as 6 and 4; actual 6. | `README.md:39`, `docs/install.md:26-28`, `docs/plugin-structure.md:16,36,64`, `docs/plugin-overview.md:45,89-93`, `docs/plugin-guide.md:51-53` vs the directories on disk |
| C-10 | **Missing memory imports.** The project instructions import three memory files that do not exist in the working tree. | `CLAUDE.md` imports `.claude/learnings/distilled.md`, `.claude/memory/skills.md`, `.claude/memory/rules.md`; only `project-memory.md`, `decisions.md`, `patterns.md` exist |
| C-11 | **Commands that do not exist.** The project instructions present seventeen slash commands as available; they exist only as prompt blocks in a guide and have no command file. The same instructions omit the flagship command. | `CLAUDE.md` "Workflow conventions" vs `commands/` and `docs/development-workflow.md:126-825` |
| C-12 | **Dangling adapter hook.** Two overlays register a hook pointing at a script the generator never copies into the overlay. | `hooks/hooks.json` PreToolUse Read entry vs `scripts/build-adapters.py:195-199` include list |
| C-13 | **Read guard writes.** A `PreToolUse` read guard mutates the repository as a side effect of blocking a read. | `scripts/read-guard.sh:87` invoking `scripts/story-graph.py`, whose freshness check writes at `:618,630,634` |
| C-14 | **Read verbs write.** The story graph's `show`, `fetch`, and `ids` verbs all rewrite the graph; only `build` is documented as writing. | `scripts/story-graph.py:605-608,618,630,634` vs `commands/setup/session-initialization.md` |
| C-15 | **Whole-file read contradiction.** One skill lists two of the four guarded artifacts as whole-file reads while other skills and a test say the guard refuses exactly that during an active story. | `skills/review-fixer/SKILL.md` inputs vs `skills/developer/SKILL.md`, `tests/test_command_references.sh` |
| C-16 | **Council persona names.** The agent's own description names three personas; its body defines three different ones, and the output filenames match the body. | `agents/product-council.md:5` vs `:40,75,111` |
| C-17 | **Council tool list.** The council agent declares read-only tools yet instructs three file writes. | `agents/product-council.md` frontmatter vs body |
| C-18 | **Spawn tool naming.** Two names are used for the same mechanism across command files. | `commands/setup/agent-spawning-model.md:10`, `commands/build-feature.md:131` vs `commands/pause.md:68,98`, `commands/stop.md:62,143` |
| C-19 | **Agent count in `/stop`.** The command says four agents will extract; it invokes one, which itself spawns four read-only sub-agents. | `commands/stop.md:139-141` vs `:143-150`, `agents/context-extractor.md` |
| C-20 | **Bare version-control CLI.** Two commands instruct the bare CLI while the repository's own rule forbids it and mandates a wrapper. Plausibly intentional for downstream users, but unresolved inside this repo. | `commands/review-plan.md`, `commands/review-pr.md` vs `CLAUDE.md` GitHub Account Binding rule 1 |
| C-21 | **Adapter location claim.** One command sends the reader to a file for a block that lives in a different file. | `commands/quick-feature.md:266` vs `commands/setup/output-write-adapter.md` |
| C-22 | **Context guard duplication.** The protocol says the check is never a separate call and the file need not be re-read, yet two commands carry full standalone copies run as their own step. | `commands/protocols/context-guard.md:10-12` vs `commands/quick-feature.md:39-48`, `commands/debug-browser.md:25-34` |
| C-23 | **Quick-feature terminal state.** The commit phase has no completion call, so a quick story never reaches a terminal state. | `commands/quick-feature.md` phase marking vs `commands/phases/09-pr.md:65-66` |
| C-24 | **Duplicate step number.** Two different steps share a number, and a later step refers ambiguously to "the numbered list from step 7". | `commands/stop.md:161` and `:174` |
| C-25 | **Model pin with no source.** One command states a default model for the whole plugin that nothing else establishes. | `commands/current-context.md:13` vs `commands/setup/agent-spawning-model.md:35-38` |
| C-26 | **Legacy state file.** One skill writes a plain-text progress file and spawns sub-processes by a mechanism another skill explicitly forbids. | `skills/git-conflict-resolver/SKILL.md` vs `skills/parallel-executor/SKILL.md` |
| C-27 | **Missing reference file.** One skill references a templates file four times; the file does not exist. | `skills/contract-loop/SKILL.md` vs the contents of its directory |
| C-28 | **Unterminated code fence.** One skill opens a fenced block that is never closed, swallowing the rest of a step. | `skills/epic-planner/SKILL.md` |
| C-29 | **Naming collision in learnings.** The sweep skill writes files named for memory files but into the learnings directory. | `skills/contract-loop/SKILL.md` output paths vs `.claude/memory/` contract |
| C-30 | **Changelog ordering.** A released section sits above the unreleased section, inverting the stated format. | `CHANGELOG.md:8` vs `:52` |
| C-31 | **Tracker tool count.** One doc claims a tool count that no doc enumerates; two others list ten and five names. | `docs/plugin-overview.md:210` vs `docs/CONVENTIONS.md:92-94`, `docs/memory-system.md:81-82` |
| C-32 | **Test suite totals.** One doc says eleven suites; another lists twelve; the directory holds more, and neither doc lists one of them. | `README.md:40` vs `docs/testing.md:19-31` vs `tests/` |
| C-33 | **Stale product name in an install step.** One doc's install step still names the pre-rename repository. | `docs/plugin-overview.md:32` vs `CHANGELOG.md:539-541` |
| C-34 | **Error handling in the index server.** The tool-error handler catches ordinary exceptions but not the system-exit the loader raises when the index file vanishes between a check and a load, which would kill the server process instead of returning an error. | `scripts/mcp-index-server.py:226` vs `scripts/scan-codebase.py:622` |
| C-35 | **Blunt redirect regex.** The branch guard blocks on any redirect character anywhere in a command string, including inside quotes. Isaac's broader write ban inherits this and makes it more consequential. | `scripts/pre-tool-use.sh:178` |
| C-36 | **Dead comments in the session wrapper.** The wrapper's header claims a session log and catch-up curation; it does neither. | `scripts/claude-start.sh` header vs body |
| C-37 | **Headless write scope.** The stop hook's "may only write learnings" constraint is prompt text; the spawned agent gets unrestricted write tools. | `scripts/stop.sh:44-46` |
| C-38 | **Unattended push.** The sync script pushes to the remote unattended after an "ours" merge, launched from every session start unless disabled. Isaac must decide whether to keep an unattended remote write. | `scripts/branch-sync.sh:86`, `scripts/session-start.sh:21` |

### 10.3 Isaac decisions that are not facts

| # | Decision needed |
|---|---|
| D-1 | The write server's name. `isaac-write` is a placeholder; the source has no precedent. |
| D-2 | Whether the state directory name changes with the product name, orphaning existing indexes (harmless, they rebuild) or stays, mismatching the product. |
| D-3 | Whether the session-config backend enum narrows to one value (breaking existing configs) or stays wide with a runtime rejection. |
| D-4 | Whether the four routed state helpers expose their MCP tools on the same server as the file tools or a third server. |
| D-5 | Whether `git` stays exempt from the write ban, or whether Isaac reimplements the small set of git writes the pipeline performs as MCP tools. This PRD assumes exempt; the alternative is a much larger surface. |
| D-6 | Which of the 38 contradictions in 10.2 Isaac fixes at port time versus carries forward for parity. NG3 says carry forward by default; each fix is an explicit exception. |

---

## 11. Appendix A — Command name preservation table

Every command in `commands/` and `commands/phases/`. Isaac's name is identical in every row; that
is the point of the table.

### Slash commands

| build-agents-sdd | Isaac | Source file |
|---|---|---|
| `/archive` | `/archive` | `commands/archive.md` |
| `/build-feature` | `/build-feature` | `commands/build-feature.md` |
| `/context-report` | `/context-report` | `commands/context-report.md` |
| `/current-context` | `/current-context` | `commands/current-context.md` |
| `/debug-browser` | `/debug-browser` | `commands/debug-browser.md` |
| `/gate-verify` | `/gate-verify` | `commands/gate-verify.md` |
| `/learn-from` | `/learn-from` | `commands/learn-from.md` |
| `/match-figma-design` | `/match-figma-design` | `commands/match-figma-design.md` |
| `/memory-prune` | `/memory-prune` | `commands/memory-prune.md` |
| `/memory-recall` | `/memory-recall` | `commands/memory-recall.md` |
| `/memory-review` | `/memory-review` | `commands/memory-review.md` |
| `/memory-save` | `/memory-save` | `commands/memory-save.md` |
| `/pause` | `/pause` | `commands/pause.md` |
| `/quick-feature` | `/quick-feature` | `commands/quick-feature.md` |
| `/review-plan` | `/review-plan` | `commands/review-plan.md` |
| `/review-pr` | `/review-pr` | `commands/review-pr.md` |
| `/stop` | `/stop` | `commands/stop.md` |

### Phase modules

| build-agents-sdd | Isaac | Source file |
|---|---|---|
| Phase 1 — Discovery | Phase 1 — Discovery | `commands/phases/01-discovery.md` |
| Phase 2+3 — Spec + UX | Phase 2+3 — Spec + UX | `commands/phases/02-spec-ux.md` |
| Phase 03 — Epic Decomposition | Phase 03 — Epic Decomposition | `commands/phases/03-epic-decomposition.md` |
| Phase 4 — Technical Architecture | Phase 4 — Technical Architecture | `commands/phases/04-architecture.md` |
| Phase 4.5 — Task Decomposition | Phase 4.5 — Task Decomposition | `commands/phases/04b-task-decomposer.md` |
| Phase 5 — Branch Setup | Phase 5 — Branch Setup | `commands/phases/05-branch-setup.md` |
| Phase 6 — Implementation | Phase 6 — Implementation | `commands/phases/06-implementation.md` |
| Phase 6.5 — Spec Verification | Phase 6.5 — Spec Verification | `commands/phases/06b-verificator.md` |
| Phase 7 — Testing | Phase 7 — Testing | `commands/phases/07-testing.md` |
| Phase 8 — Review + Audit + Fix Loop | Phase 8 — Review + Audit + Fix Loop | `commands/phases/08-review-loop.md` |
| Phase 9 — Pull Request | Phase 9 — Pull Request | `commands/phases/09-pr.md` |

### Supporting modules (also name-preserved)

| Kind | Names | Source |
|---|---|---|
| Protocols | `blocker-escalation`, `context-guard`, `epic-scoping`, `human-in-the-loop`, `phase-transition`, `resuming` | `commands/protocols/` |
| Rules | `core-rules`, `loop-mode-rules` | `commands/rules/` |
| Setup | `agent-spawning-model`, `command-fit-check`, `final-report-template`, `flags-and-modes`, `output-write-adapter`, `preflight-questions`, `session-initialization`, `staging-guard-and-worktree` | `commands/setup/` |
| Skills | `code-reviewer`, `context-reporter`, `contract-loop`, `developer`, `devops-engineer`, `e2e-testing-patterns`, `epic-planner`, `feedback-triage`, `git-conflict-resolver`, `javascript-testing-patterns`, `parallel-executor`, `product-strategist`, `review-fixer`, `security-auditor`, `sn-built-in-first`, `spec-writer`, `task-decomposer`, `technical-architect`, `test-engineer`, `using-git-worktrees`, `ux-designer` | `skills/` |
| Agents | `branch-sync`, `context-extractor`, `learning-observer`, `memory-curator`, `product-council`, `verificator` | `agents/` |

---

## 12. Appendix B — File-write map

Every file path `build-agents-sdd` writes at runtime, and the Isaac tool that writes it. Paths are
shown with their source placeholders.

### Story artifacts

| Path | Isaac tool | Written by |
|---|---|---|
| `docs/<story-id>/` | `mkdir` | session-initialization, `story-state.py init` |
| `docs/<story-id>/loop-contracts/` | `mkdir` | session-initialization when LOOP_MODE |
| `docs/<story-id>/session.json` | `create_file` | preflight-questions |
| `docs/<story-id>/story-snapshot.json` | routed helper (see §6.6) | `story-state.py` |
| `docs/<story-id>/story-graph.json` | routed helper | `story-graph.py` |
| `docs/<story-id>/validated-idea.md` | `create_file` | product-strategist; council prepends via `update_file` |
| `docs/<story-id>/council/product-mind.md` | `create_file` | product-council |
| `docs/<story-id>/council/devils-advocate.md` | `create_file` | product-council |
| `docs/<story-id>/council/security-advocate.md` | `create_file` | product-council |
| `docs/<story-id>/requirements.md` | `create_file` | spec-writer |
| `docs/<story-id>/user-stories.md` | `create_file` | spec-writer |
| `docs/<story-id>/adrs/` | `mkdir` | spec-writer |
| `docs/<story-id>/adrs/adr-NNN.md` | `create_file` | spec-writer, technical-architect |
| `docs/<story-id>/ux-wireframes.md` | `create_file` | ux-designer |
| `docs/<story-id>/epics/_epics-proposal.md` | `create_file` (overwrite on re-invoke) | epic-planner |
| `docs/<story-id>/epics/_epics.md` | `create_file` | Phase 03 on accept |
| `docs/<story-id>/epics/epic-NN-<slug>/` | `mkdir` | Phase 03 on accept |
| `docs/<story-id>/architecture.md` | `create_file` | technical-architect |
| `docs/<story-id>/implementation-plan.md` | `create_file` | technical-architect |
| `docs/<story-id>/codebase-context.md` | `create_file` | task-decomposer |
| `docs/<story-id>/tasks/` | `mkdir` | task-decomposer |
| `docs/<story-id>/tasks/task_NN.md` | `create_file` | task-decomposer |
| `docs/<story-id>/tasks/task-graph.json` | routed helper | `task-graph.py` |
| `docs/<story-id>/implementation-notes.md` | `create_file` | developer, single-track Phase 6 |
| `docs/<story-id>/tracks/06-track-<T>/implementation-notes.md` | `create_file` | per-track developer agents |
| `docs/<story-id>/tracks/<phase-key>-track-N/output.log` | exempt (deprecated runner) | `parallel-run.sh` |
| `docs/<story-id>/tracks/<phase-key>-track-N/status.txt` | exempt (deprecated runner) | `parallel-run.sh` |
| `docs/<story-id>/tracks/<phase-key>-summary.txt` | exempt (deprecated runner) | `parallel-run.sh` |
| `docs/<story-id>/parallel-prompts/<phase-key>-track-*.md` | `create_file` | parallel-executor |
| `docs/<story-id>/verification-report.md` | `create_file` (overwrite) | verificator |
| `docs/<story-id>/verification-result.json` | `create_file` | verificator |
| `docs/<story-id>/test-plan.md` | `create_file` | test-engineer |
| `docs/<story-id>/test-results.md` | `create_file` | test-engineer |
| `docs/<story-id>/reviews-<NNN>/` | `mkdir` | Phase 8 |
| `docs/<story-id>/reviews-<NNN>/code_NN.md` | `create_file` | code-reviewer |
| `docs/<story-id>/reviews-<NNN>/security_NN.md` | `create_file` | security-auditor |
| `docs/<story-id>/reviews-<NNN>/round-summary.md` | `create_file` | review-fixer |
| `docs/<story-id>/review-report.md` | `create_file` | code-reviewer |
| `docs/<story-id>/security-report.md` | `create_file` | security-auditor |
| `docs/<story-id>/loop-contracts/<phase-N>-contract.md` | `create_file` | phase-transition Step 2a |
| `docs/<story-id>/loop-contracts/<phase-N>-evidence.md` | `create_file` | phase-transition Step 2b/2c |
| `docs/<story-id>/loop-contracts/<phase-N>-failure.md` | `create_file` | phase-transition Step 2d |
| `docs/<story-id>/loop-report.md` | `create_file` | final-report-template |
| `docs/<story-id>/scope.md` | `create_file` | quick-feature |
| `docs/<story-id>/triage-report.md` | `create_file` | feedback-triage |
| `docs/<story-id>/<filename>.ref` | `create_file` | output-write adapter |
| `docs/_archived/` | `mkdir` | archive |
| `docs/_archived/<story-id>/` | `move_file` | archive |
| `docs/backlog.md` | `append_file` | feedback-triage |
| `docs/hotfix-<date>/validated-idea.md` | `create_file` | feedback-triage |
| `docs/pr-<number>-plan.md` | `create_file` | review-plan on request |
| `docs/pr-<number>-review.md` | `create_file` | review-pr on request |
| `docs/debug-<ts>/bug-report.md` | `create_file` | debug-browser |
| `docs/debug-<ts>/evidence-raw.md` | `create_file` | debug-browser |
| `docs/debug-<ts>/root-cause.md` | `create_file` | debug-browser |
| `docs/debug-<ts>/fix-notes.md` | `create_file` | debug-browser |
| `docs/debug-<ts>/verification.md` | `create_file` | debug-browser |
| `docs/debug-<ts>/screenshots/*.png` | browser automation MCP (already MCP) | debug-browser |

### Memory and learnings

| Path | Isaac tool | Written by |
|---|---|---|
| `.claude/memory/project-memory.md` | `append_file`; `update_file` for the State section | memory-curator, developer, `/memory-save`, `/archive` |
| `.claude/memory/decisions.md` | `append_file` | memory-curator, context-extractor, technical-architect, `/memory-save` |
| `.claude/memory/patterns.md` | `append_file` | memory-curator |
| `.claude/memory/rules.md` | `append_file` | context-extractor |
| `.claude/memory/skills.md` | `append_file` | context-extractor |
| `.claude/memory/tokens-consumed.md` | `append_file` | context-extractor |
| `.claude/memory/session-log/<id>.jsonl` | exempt (hook) | hook scripts |
| `.claude/memory/session-log/<id>.jsonl.state-changes` | exempt (hook) | post-tool hook |
| `.claude/memory/session-log/<id>.jsonl.curated` | `create_file` from a command; exempt from a hook | `/pause`, `/stop`, memory-curator |
| `.claude/memory/.curator-<id>.log` | exempt (headless spawn output) | curator spawn |
| `.claude/memory/branch-sync.log` | exempt (WR-9) | `branch-sync.sh`, branch-sync agent, scheduler |
| `.claude/learnings/distilled.md` | `append_file` | learning-observer, `/learn-from`, verificator |
| `.claude/learnings/pending.md` | `append_file`, `update_file` | learning-observer |
| `.claude/learnings/retired.md` | `append_file` | `/memory-review`, `/memory-prune` |
| `.claude/learnings/corrections.jsonl` | `append_file` | learning-observer, `/learn-from` |
| `.claude/learnings/post-push-queue.md` | `update_file` (truncate) | learning-observer consumes |
| `.claude/learnings/.observer-last.log` | exempt (hook) | `stop.sh` |
| `.claude/learnings/loop/contract.md`, `plan.md`, `state.md`, `flagged.md`, `baseline.md` | `create_file` | contract-loop |
| `.claude/learnings/loop/evidence/REQ-XXX/` | `mkdir` + `create_file` | contract-loop |
| `.claude/learnings/<objective>-report.md` | `create_file` | contract-loop |
| `.claude/agents/loop-implementer.md` | `create_file` | contract-loop |

### Flags and markers

| Path | Isaac tool | Written by |
|---|---|---|
| `.claude/.learning-pending` | exempt (hook) | `user-prompt-submit.sh`; removed by `stop.sh` and the clean wrapper flag |
| `.claude/.context-extract-pending` | `create_file`; `delete_file` to clear | Phase 9 writes, `/stop` clears |
| `.claude/.resume-context.md` | `create_file` (overwrite) | `/pause` |
| `.claude/.resume-archive/<ts>_<disposition>.md` | `mkdir` + `move_file` | `/stop` |

### Generated state, reports, and scratch

| Path | Isaac tool | Written by |
|---|---|---|
| `<state-dir>/state/codebase-index.json` | `refresh_index` MCP tool; CLI path exempt | scanner, index server |
| `<state-dir>/state/` | created by the scanner; exempt | scanner |
| `<state-dir>/.gitignore` | exempt | scanner |
| `<state-dir>/project.json` | `create_file` | cross-story memory setup |
| `reports/<story-id>/phase-<N>-context-<ts>.html` | `create_file` | context-reporter |
| `reports/<story-id>/session-full-context-<ts>.html` | `create_file` | context-reporter |
| `/tmp/context_report_gen.py` | exempt (temporary directory, WR-9) | context-reporter |
| `/tmp/test-coverage/**` | exempt (temporary directory) | contract-loop |
| `/tmp/parallel-worktrees/<story-id>-06-track-<T>/` | exempt (git worktree, WR-9) | parallel-executor |
| `$(git rev-parse --git-path info/exclude)` | exempt (git metadata, WR-9) | Phase 1 in tracker mode |
| `dist/**` | exempt (build-time generator) | `build-adapters.py` |
| target project tree | exempt (installer) | `install-adapter.sh` |
| platform scheduler state | exempt (outside the repository) | `install-branch-sync-schedule.sh` |
| source files and co-located tests | `create_file`, `update_file` | developer, review-fixer, contract-loop implementer, git-conflict-resolver |
| git branches, commits, merges, worktrees, pushes | exempt (WR-9) | devops-engineer, parallel-executor, branch-sync, archive |
