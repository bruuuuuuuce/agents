# Agent Guidelines

Rules and expectations for all AI agents working in this repository tree. These guidelines apply to every project unless a project-level `Agents.md` explicitly overrides a section.

---

## 1. Micro Spec Convention

> Design, steering, and micro-spec guidelines — how to write them: `@design/SPECS.md`

- Every non-trivial task begins with a micro spec written **before** any code.
- No code is written until the micro spec exists, is committed, and is approved by the reviewer or human (§5 Two phases).
- Spec is the source of truth. Code to its intended target — do not rewrite it to match the code.
- The spec is not a status tracker: no `DONE`/`Status:`/`REVERTED` markers or edit history in spec prose — state the settled contract only (`design/SPECS.md` § The spec is not a tracker).
- Spec statements follow SOLID, open/closed in particular: state what the change adds or does, not the module's full inventory (`design/SPECS.md` § Objective).
- Acceptance criteria are guides for groups of testing, not micro-detail inventories: each derives at least one test — usually more — at implementation; leave them as generalizations where appropriate and never treat or present them as the ceiling of testing (`design/SPECS.md` § Acceptance criteria are guides, not inventories).
- Resolve gaps by size:
  - **In-scope gap** → apply conventions, record the assumption, continue. Never ask.
  - **Ambiguous** (conventions conflict or none applies) → ask.
  - **Small drift** (naming, local structure) → update spec, continue.
  - **Large divergence** (new approach, changed contract, added/dropped requirement) → ask, at the end of a turn that delivers everything not depending on the answer. Do not self-approve by editing the spec.
  - Unsure if large → treat as large, ask.
- Fill in-scope gaps with the established conventions:
  - Security → [security-principles](reference/security-principles.md):
    - Fail-closed — deny on absent/empty/unknown.
    - Secure by query — the query is the gate, not fetch-and-filter.
    - Least privilege — narrowest scope that works.
  - Design → SOLID + [design-principles](reference/design-principles.md) + [design-patterns](reference/design-patterns.md):
    - One responsibility.
    - One owner per contract.
    - No speculative abstraction.
  - Standards → §4 + [`CONVENTIONS.md`](CONVENTIONS.md) + [`style/`](style/) (per-language):
    - Enums for closed sets.
    - No primitive obsession.
- Then check the result against [`reference/anti-patterns.md`](reference/anti-patterns.md):
  - The *negative* set — things to avoid, not conventions to apply.
  - E.g. gold plating, primitive obsession, smuggler.
- Acceptance criteria drive the test plan — every criterion maps to at least one test.
- In markdown docs, do not manually hard-wrap prose lines.

### Negative example — inventing a requirement

- **Task:** read table `records` via credential `C`, scoped by org. One credential is the data boundary; one preflight check is the gate.
- **Failure:** agent added a per-row permission check nobody asked for — access predicate, batch checker, post-fetch filter — a permission tier with no credential, no ACL column, no way to configure it.
- Adding a per-row gate is a *new access model*, not an in-scope gap → was a large divergence → stop and ask.
- Fail-closed hardens the *one* defined gate; it never means invent a second gate → the per-row layer is [Gold Plating](reference/anti-patterns/gold-plating.md) + a §4 C6 violation (no consumer).
- Post-fetch filtering is fetch-then-check — the anti-pattern the single query gate already avoided.
- When flagged, the agent defended the layer and pushed to spread it "for consistency" instead of deleting it.
- The unrequested machinery dwarfed the actual task.
- **Tell:** adding a check/credential/gate the spec never named — per row, per field, per call — is a large divergence, not a gap. Stop and ask.
- **Rule:** if your defense of a layer is your own reasoning, not a spec line, delete it — do not defend it.

---

## 2. Unit Test Standard

> Full details: `@unittesting/GENERAL.md` | Pre-submit: `@unittesting/CHECKLIST.md`

- **Target**: 97% line coverage, 100% branch coverage on public interfaces
- **Cycle**: Red → Green → Refactor. Never write production code before seeing a red test.
- **Red means committed red**: the failing test is added to the real suite and committed *while it fails*, before any production-code change. A "temporary" test that is run once and never committed is not a red light — it is fabricated evidence. The commit history must show the red-test commit preceding the fix commit. Hooks that run the suite will reject the red commit by design — that is the one sanctioned use of `--no-verify` (§5 step 6).
- **Pattern**: AAA (Arrange, Act, Assert). One assertion per test. No shared mutable state.
- **Structure**: `tests/unit/` (every save), `tests/integration/` (on PR), `tests/e2e/` (on merge). Mirror source paths.
- **Deterministic**: no randomness, no wall-clock time, no network — stub at the boundary
- **CI gate**: coverage below 97% fails the PR
- **Automation**: unit tests, lints, and formatting are enforced by deterministic tools via CI and pre-commit hooks — never run manually or by the agent. If a check can be automated, it must be.
- **Scratch checks stay out of the suite.**
  - Verify however you like — a one-off probe or script that only explores behaviour is not committed and is not evidence.
  - Every committed test traces to an acceptance criterion or a red-light claim (§5, §6).
  - Size tests like the neighbouring test files — roughly one focused test per stated behaviour.
  - A scratch check does not become a permanent test file.

---

## 3. SOLID Design Principles

This project applies all five SOLID principles:

| Abbr | Full name | One-line rule |
|---|---|---|
| **SRP** | Single Responsibility | One reason to change per module, class, or function |
| **OCP** | Open / Closed | Open for extension, closed for modification |
| **LSP** | Liskov Substitution | Subtypes must be drop-in replacements for their base type |
| **ISP** | Interface Segregation | Many narrow interfaces over one wide one |
| **DIP** | Dependency Inversion | Depend on abstractions, not concretions |

Call the principle out by abbreviation in review comments and specs (e.g. "this violates SRP — split the parse and persist steps").

SOLID applies to statements too — docs, specs, PR descriptions, commit messages, chat. Open/closed is the usual failure: `style/DOT_POINT_SRP.md` § SOLID statements.

### SRP — Single Responsibility

- One reason to change per module, class, or function.
- A unit doing two things — parse *and* persist, validate *and* format — is split.
- The name states the single responsibility; you should not need to read the body.

### OCP — Open / Closed

- Add behaviour by extending, not by editing existing code.
- Achieve it with composition, strategy objects, and defined extension points.
- Needing to edit an existing class body signals the abstraction boundary is wrong.

### LSP — Liskov Substitution

- Every subtype honours its base type's contract.
- A caller holding the base type observes identical behaviour for any subtype.
- Violations: a subtype that throws where the parent does not, returns a narrower type, or silently ignores a method.

### ISP — Interface Segregation

- No implementor defines methods it does not use.
- Split wide interfaces into focused ones.
- Clients depend only on the slice they call.

### DIP — Dependency Inversion

- High-level policy modules do not import low-level detail modules.
- Both depend on a shared abstraction.
- Concrete implementations are injected at the composition root — never constructed inside business logic.

---

## 4. Engineering Quality Standards

### Leverage compiler safety

- In strongly-typed languages (TypeScript, Rust, Go, etc.), exploit the compiler to catch bugs at build time, not runtime.
- **Use `as const` for literal types.** When a value must be a specific string or number (e.g. an ID in a discriminated union, a route name, an action type), assert it with `as const` so the compiler narrows the type to the literal rather than widening to `string`.
  ```typescript
  // Bad — id is typed as string, typos compile silently
  { id: "setting", label: t("settings") }

  // Good — id is the literal type "setting", mismatches are compile errors
  { id: "setting" as const, label: t("settings") }
  ```
- **Prefer const enums / union types over plain strings** for finite sets of values (action types, status codes, mode names).
- **Enable strict compiler flags** (`strict: true` in tsconfig, `-Wall -Werror` in C/C++, `clippy` in Rust). Never weaken them to fix a build — fix the code instead.
- **Let the type system replace runtime guards.** If a check can be expressed as a type constraint, do that instead of writing an `if` that throws at runtime.
- **Never use `any` unless absolutely necessary.** `any` disables type checking and defeats the purpose of a type system. Use `unknown` when the type is genuinely not known — it forces callers to narrow before use. If a third-party API returns `any`, wrap it and type the return at the boundary. The only acceptable uses of `any` are interop with untyped legacy code where a proper type is infeasible — and these must include a `// eslint-disable-next-line @typescript-eslint/no-explicit-any` comment explaining why.

### Fail-closed defaults

- Any code that filters, permits, or gates defaults to deny.
- **Present-but-empty means deny.** An empty permission set, an empty filter list, or an empty role array grants nothing — never treat empty the same as absent.
- **Absent may mean "legacy, allow" only if the spec documents it.** If there is no documented legacy exception, absent also means deny.
- **Missing declarations exclude.** An item without a permission or type declaration is excluded from results, not included by default.
- **State the posture in the spec.** Every gate or filter in a micro spec must say what happens when input is absent, empty, or unrecognized.

### Booleans over string arrays for fixed permission sets

- Closed, compile-time-known permission sets are a record of booleans (`{ read: true, write: false }`), not a string array (`["read"]`).
- Booleans cannot be misspelled, are self-documenting, and the compiler catches missing keys.

### No metadata in data namespaces

- System fields (timestamps, version markers, internal IDs) never share a namespace with user-supplied data.
- System fields live under a single reserved envelope key, written after user data so they cannot be shadowed.
- The reserved key is stripped from user input at the boundary.

### Cursor pagination, not offset pagination

- All paginated endpoints and list queries use cursor-based pagination keyed on a stable record ID — never offset/page-number.
- Offset pagination breaks under concurrent writes (skipped/duplicated rows) and degrades at depth (`OFFSET 10000` scans and discards 10,000 rows).
- The cursor is an opaque token derived from the last record's ID (or a composite key when sort order requires it).
- Response shape: `{ items, nextCursor, hasMore }`. No `page`, `totalPages`, or `offset` fields.
- The underlying query uses a `WHERE id > :cursor ORDER BY id LIMIT :size` pattern (or equivalent for the ORM/database).
- If a UI needs a page-number display, the frontend synthesises it from cursor state — the API never exposes offset semantics.

### No compatibility shims for internal code

- No backwards-compatibility wrappers, re-exports, or adapter layers for internal callers.
- When an internal interface changes, update every caller.
- Compat shims are only justified at published public API boundaries.

### Code style

- Follow the language's idiomatic style guide.
- Maximum function length: **30 lines** of logic (excluding blank lines and comments). If longer, extract a helper with a clear name.
- Maximum file length: **400 lines**. Larger files signal more than one concept in the file.
- No commented-out code committed. Use version control instead.

### Naming

- Names are pronounceable, unambiguous, and domain-specific.
- No abbreviations unless universally understood in the domain.
- Boolean names start with `is_`, `has_`, `can_`, or `should_`.

### Error handling

- Never silently swallow exceptions. Log and re-raise, or convert to a typed domain error with context.
- Every public function that can fail has a documented failure contract.
- Errors are typed — avoid bare catch-all error types.

### Dependencies

- No new dependency is added without a note in the micro spec justifying it.
- Prefer standard library over third-party where the effort is comparable.
- Pin all dependency versions; no floating version ranges in lock files.

### Git hygiene

- Commits are atomic: one logical change per commit.
- Commit message format:
  ```
  <type>: <imperative short summary>

  <body — why, not what; optional, max 3 lines>
  ```
  Types: `feat`, `fix`, `refactor`, `test`, `docs`, `chore`.
- **Keep commit messages terse.** The summary line is under 72 characters. The body, if present, is 1–3 lines explaining *why* — not a changelog, not a list of every file touched, not a paragraph restating the diff. If the diff is self-explanatory, the body is omitted entirely.
- No merge commits on feature branches; rebase onto main before merging.
- **Do not squash** unless explicitly instructed. A squash is a one-time operation — after completing it, return to pushing incremental commits. Do not squash on every subsequent fix after an authorised squash.
- Branch names: `<type>/<short-slug>`.

### Pull / merge requests

- PR description links to the micro spec doc.
- Use the repo's PR template (`.github/pull_request_template.md` or `.github/PULL_REQUEST_TEMPLATE/`): fill it in, keep its sections, checkboxes, and links intact. Fall back to `~/agents/DEFAULT_PR.md` only when the repo has none.
- PR description updates are surgical: fetch the current body, change only the lines that change, leave the rest byte-for-byte, and verify afterwards that no link or section was lost. A wholesale rewrite is not an update.
- Checklist before requesting review:
  - [ ] All acceptance criteria from the micro spec are met.
  - [ ] Coverage gate passes locally.
  - [ ] No new lint warnings introduced.
  - [ ] Micro spec updated if scope changed during implementation. Any large divergence from the spec's intent was raised with the user and approved before proceeding — not self-approved by editing the spec (see §1).
  - [ ] CHANGELOG entry added for user-visible changes.

---

## 5. Agent Workflow

- An agent picking up a task **must** follow this order.
- Every gate demands an artifact — instructions that produce nothing get skipped; instructions whose absence is visible cannot.
- No agent claims work complete until every applicable gate has a recorded artifact.

### Working style

- When you have enough information to act, act.
  - Do not re-derive facts already established in the conversation.
  - Do not re-litigate a decision the user has already made.
  - Do not narrate options you will not pursue.
  - Weighing a choice → give a recommendation, not a survey.
- Say in one line what you are about to do before starting; give brief updates while you work.
- Batch tool calls: privately list what you need next, then request every item that does not depend on another's result in the same response.
- **Edit surgically.** Files, docs, specs, PR descriptions: change the lines that change; a whole-artifact rewrite that produces the same result is waste.
  - Tokens spent editing are best minimised.
  - A wholesale rewrite silently drops content that was not in view — links, template sections, reviewer edits.
- **Finish the whole task.** A turn ends at the phase gate below, or when blocked on input the user has to provide.
  - Reversible actions that follow from the task → proceed without asking.
  - Ending a turn to ask permission for work the task already covers, to report a step, or because the session is long leaves the task undone.
  - Offering follow-ups after the work is fine; asking permission before doing requested work is not.
  - A per-item halt (cascade, `UNPROVEN`, `NEEDS_REVIEW`, a large divergence on one item) is reported and the rest of the work continues; it does not end the turn.
- **Two phases, one gate between them.**
  - Phase 1 — spec: write or update the micro spec (`design/SPECS.md`), pass the process gates (P1–P3), commit, push, then end the turn asking for approval. This turn ends on a question.
  - Phase 2 — code: on reviewer/human approval, implement every acceptance criterion (red → green → commit → push), run the submission gates and review triage, report. Work continues to completion.
  - Phase 2 starts on an approved spec.
  - Phase 1 reopens mid-Phase 2 for a large divergence (§1) — ask, at the end of a turn that delivers everything not depending on the answer.
- **Token scope in long-run loops (Phase 2, review, triage).** Everything produced in one reply — reasoning, drafting, and the reply itself — counts toward one output limit; a cut-off reply is a restart.
  - Reason in the reasoning space; write the deliverable once, in the output space — a file, diff, or report is drafted once, not in full as reasoning and again as the reply.
  - Spend the reasoning on understanding the request, checking the inputs the answer depends on, and settling structure and hard decisions; then write.
  - Large deliverables land in pieces: one file, one commit, one report section per step, each written once.
  - Phase 1 (spec) is exempt: deliberate as long as the design needs.
- Before ending a turn, check the last paragraph.
  - A plan, a question, a list of next steps, or a promise about undone work ("I'll…", "let me know when…") → do that work now with tool calls.
  - That includes retrying after errors and gathering missing information yourself.
  - End the turn when the task is complete or blocked on input the user has to provide.
- Do not stop, summarise, or suggest a new session because the context or session is long.
- Exception: the user is describing a problem, asking a question, or thinking out loud rather than requesting a change → the deliverable is the assessment. Report findings and stop; do not apply a fix until asked.
- Before a command that changes system state (restart, delete, config edit), check the evidence supports that specific action — a signal that pattern-matches a known failure may have a different cause.
- **Delegate and keep working.** Independent subtasks go to subagents; the lead does not block on them.
  - Keep working on the next item — or the user's next update — while they run.
  - Integrate each result as it arrives; a delegated task is complete when its result is integrated and reported.
  - Do not redo a delegated task, predict a pending result, or end the whole task with a delegation outstanding.
  - Intervene if a subagent goes off track or lacks context.

### Before code

1. **Read the bug report or task fully.** If the task references a review issue, read the entire `<ID>.md` detail file — description, evidence, fix guidance, and reverify steps. Do not skim summaries or titles. Do not make decisions, push back, or categorise an issue without reading the full detail file first.
2. **Read** the relevant micro spec (or create one if absent, per `design/SPECS.md`).
3. **Read** existing code in the affected area before writing anything.
4. **Write or update** the micro spec if the task is new or scope changes.

**Process gates — Phase 1 ends here; no implementation until all pass and the spec is approved:**

| # | Gate | Evidence |
|---|------|----------|
| P1 | **Spec-with-matrix exists.** For any feature that filters, permits, gates, or falls back: the spec contains the full input-state × behavior table — with absent and present-but-empty as separate rows — and no cell reads TBD. | The table in the spec doc, committed before implementation. |
| P2 | **Acceptance criteria are executable.** Every AC names a test file/case that fails before implementation and passes after. Prose-only ACs are invalid. | Red run before, green run after — both captured, and the red test is a committed suite file (commit sha), not a temporary/uncommitted file. |
| P3 | **Design is checked against the anti-pattern catalog before coding.** Named check of `anti-patterns/CHECKLIST.md` sections relevant to the design (smuggler for any new field on shared objects, primitive-obsession for any new string, boat-anchor for anything speculative). | Pass/fail/N-A list in the spec. |

### During implementation

5. **Red** — write one test in the real suite (committed file paths, not scratch/temp files), run the suite, confirm that test fails for the right reason, and **commit the failing test** with its raw red output referenced in the commit message. A test that cannot be seen to fail proves nothing; a red run with no committed test is unverifiable and does not count.
6. **Green** — write the minimum production code needed to make that test pass. No more. The fix is a separate commit after the red-test commit, so history proves the test failed before the code changed.
   - A pre-commit hook that runs the suite (see §2 Automation) will reject a red commit by design. **In this case only, `--no-verify` is sanctioned**: the redness is the proof being committed, and the coder knows it. This is the sole permitted use of `--no-verify` — red-light commits are standing policy, pre-authorized by the user.
   - Before using it, run the suite and confirm the only failures are the red-light test(s) being committed plus already-tracked red-light tests (their `redlight:` records). Any other failure is unrelated breakage — fix that first; `--no-verify` never smuggles it through.
   - The green stage is likewise not blocked by other issues' still-red tests: when committing a fix, the fixed test must pass, and hook failures caused solely by tracked red-light tests do not force clearing them first — commit with `--no-verify` and continue the cycle.
7. **Repeat** steps 5–6 for each acceptance criterion in the micro spec.
8. **Refactor** — with all tests green, clean names, split large functions, remove duplication. Run the suite after every refactor step.

**Code gates — every new or changed symbol must satisfy all that apply:**

| # | Gate | Evidence |
|---|------|----------|
| C1 | **Fail closed, stated explicitly.** Every gate declares its posture in the spec: absent → documented compat or deny; empty → deny; undeclared item → excluded. Fail-open requires a written justification. | Posture declaration in the spec for every gate. |
| C2 | **No closed set as a raw string.** Every finite value set is a named union/enum at every layer it crosses. | `grep` new fields for bare `string` types — zero hits. |
| C3 | **Invalid states unrepresentable.** Flags are booleans, not membership arrays; domain types over primitives. `{ read: true }` cannot typo; `["raed"]` can. | Type definitions in the diff use records/enums, not string arrays. |
| C4 | **System metadata never shares a namespace with user data.** One reserved envelope key, written after user data, stripped from user input at the boundary. | Smuggler checklist against the diff. |
| C5 | **One owner per contract.** A type crossing N boundaries is declared once and imported, or each copy carries a `KEEP-IN-SYNC` reference to the master, and a test pins the wire shape. | Single declaration site, or `KEEP-IN-SYNC` references plus a shape-pinning test. |
| C6 | **Only functional code.** No field, param, shim, or fallback without a current consumer named in the spec. Reviewer suggestions are proposals — they get scope-checked against objectives, not implemented by default. | Every new symbol has a caller in the diff; spec lists no unused additions. |

### Before commit

**Truth gates — claims in the diff must match the code:**

| # | Gate | Evidence |
|---|------|----------|
| T1 | **Every prose claim is verified in the same pass that touches the behavior.** Comments, docblocks, test names, spec assertions — if the claim describes behavior, either point it at a test or re-verify it when the behavior changes. A claim that cannot be checked gets deleted. | No reviewer-bait items survive the diff review. |
| T2 | **Report failures verbatim.** Failing tests, skipped steps, and unverified paths are stated plainly, never smoothed over. | Raw output included — no editorialised summaries of failures. |

9. **Commit** in atomic commits following the git hygiene rules above.

### Before push / claiming complete

**Submission gates — no push until all pass:**

| # | Gate | Evidence |
|---|------|----------|
| S1 | **The published checklists actually execute, with artifacts.** `~/agents/review/REVIEW_METHOD.md` every PR; `~/agents/review/SECURITY_REVIEW.md` check groups whenever the diff touches APIs/auth/credentials; `~/agents/reference/anti-patterns/CHECKLIST.md` against the diff. Each produces a filled item → pass/fail/N-A → `file:line` record. No artifact = didn't happen. | Checklist output files with `file:line` evidence for every item. |
| S2 | **Findings map to objectives before they map to fixes.** Every review finding is classified on-objective / robustness-layer / out-of-scope before any code is written; robustness layers default to rejected pending user decision. | Classification tag on each finding before implementation begins. |
| S3 | **Fresh state before verdicts.** Reviews and fixes run against the current HEAD after fetch — never against a stale checkout. | `git fetch` + `HEAD` SHA recorded before each review or fix pass. |

### Reporting findings

- Report bugs, issues, and findings **to the user in chat** using the finding grammar in [`~/agents/review/REVIEW_METHOD.md` § Finding Grammar](review/REVIEW_METHOD.md#finding-grammar): one line per finding, `- [<x| >] <ID> - <STATUS> [<SEVERITY>] - <title> \`file:line\``.
- Use the status tokens only (`OPEN`, `NEEDS_REVIEW:coder`, `CLOSED verified:<yyyy-mm-dd>`, …). **Never** describe a finding with a loose adjective like "present", "resolved", "done", or "handled".
- A fix that is written but not yet verified is `OPEN`, not `CLOSED verified:` — "verified" requires a passing reverify command, test, or trace, not merely that the code is present.
- Do **not** write to `review.md` or the `~/reviews/<repo>-pr-<number>/` directory. That persisted store is the reviewer/orchestrator's job (see `~/agents/review/ISSUE_TRACKING.md`). Your report is the in-chat list.
  - **Exception 1:** the review-claim triage and red-light procedure below. When executing it, the coder records and updates the claims it is processing in the review store per ISSUE_TRACKING.md.
  - **Exception 2:** `task.md` — the coder owns the file, the user owns the content (grammar: ISSUE_TRACKING.md § Task file). Never update it without the user's approval: propose the exact text in chat, write it only once approved, and keep the user's words near-verbatim — correcting only grammar, spelling, and shorthand. `# DECISIONS` entries are gated: only major actions and direction shifts qualify, each tagged with a short name (e.g. `record-not-resource`) — never task steps or work narration.
- Ground every claim: audit each progress claim against a tool result from this session before reporting it.
  - Report only work you can point to evidence for.
  - Say explicitly what is not yet verified.
- Lead with the outcome: the first sentence answers "what happened" or "what was found"; detail after.
- The final message stands alone — a reader who sees only it gets what was found, what was done, and what is next.
  - Drop working shorthand, arrow chains, and labels coined mid-task.
  - Give each file, commit, or flag its own plain clause saying what it is or what changed.
- Remove all mannered prose. Say what you mean; when a literal phrase is available, use it.
- Keep output short by selecting what to include — drop details that do not change what the reader does next — not by compressing into fragments or jargon.
- Chat replies and reports follow `style/DOT_POINT_SRP.md`: list-first, markdown-only quoting.

### Review-claim triage and red-light (mandatory after any review)

- **Run the `review-triage` skill (`~/agents/skills/REVIEW_TRIAGE.md`) for every claim a review produces** — self-review, reviewer findings, PR comments, feedback ingress. The skill is the procedure; this section is the mandate and the invariants.
- All store writes go through `~/agents/skills/review_triage.py` — it enforces the gate order (record → cascade pre-check → scope gates → trace against current HEAD → red proof or green disproof → fix → close), full IDs, and required evidence. Do not hand-edit the review store, and do not work around a script refusal — a refusal means the procedure is being violated.
- Invariants the skill enforces:
  - A claim is a hypothesis, not a fact — no verdict without reading the current code at the claimed location. The claim's analysis and framing are equally untrusted: re-derive the problem from the code yourself; reviewer prose never decides fix-vs-document or picks the remedy.
  - Every claim resolves with a **committed test in code** — red proves it, green disproves it — never agent speculation. `UNPROVEN` is only for claims where neither is constructable. Only **doc-claims and style-claims** skip the test — claims against a non-executable artifact (spec prose, comments, naming) or style-only rewrites where no committed test can distinguish pre-fix from post-fix. Kind decides, not size: a doc-claim that rewrites a whole design section is still a doc-claim. They route from the scope gate straight to verify-and-fix — no trace, no red-light, never `UNPROVEN`.
  - **A reviewer's suggested fix is untrusted input** — reviewers routinely flag their own suggestions as broken on the next pass. Design the fix from your own trace; audit any part of the suggestion you keep for compounding issues, design failures, out-of-scope bloat, and over-engineering against the happy path — especially when the fix targets an edge case, worst of all an edge case of an edge case.
  - No fix until the defect class is fully understood and its mechanism exposed by the trace. Every close records a **justification** in the issue's detail file: why this fix resolves the class fully and will not compound.
  - `NEEDS_REVIEW:cascade` items are never fixed without the user's explicit approval.
  - A cascade means a prior fix's justification was wrong — re-read that justification, explain where it failed, follow each cascade link to its root, and redesign the fix from the ground up. Never patch the patch.
  - Fixes never edit the proving test; a reverted fix's red test stays committed and red.
  - Report each step in chat as it completes — a step with no report did not happen.

10. **Push** to the PR branch after each completed change. Do not batch up commits — push proactively so the PR stays up to date.
11. **Do not merge** PRs. Merging is done by the user. Do not expect to be involved in the merge process.
12. **Do not deploy** unless the user explicitly says so.

An agent surfaces an open question rather than guessing — at the end of a turn that delivers everything not depending on the answer — when:
- A spec section is ambiguous.
- An interface from another module is missing or contradicts the spec.
- The 97 % coverage target cannot be reached without unreasonable stubbing.

---

## 6. What Agents Must Never Do

### No stashes — commit instead

- **Never use `git stash`.** WIP is committed to its branch (`wip:` prefix is fine) — commits are visible, attributable, durable, and pushable; stashes are none of those.
- A stash blocks history rewrites (squash-rebase fail-closes on stashes), survives invisibly across sessions, and loses its branch context.
- Found an existing stash? Do not apply, drop, or pop it silently — surface it, then preserve it as a commit on a branch (`git stash branch`) with the user's approval.

### No phantom tests — red evidence is committed evidence

- **Never fake the red light with a temporary test.** A test written in a scratch file, run once, and deleted, reverted, or left uncommitted is not red-light evidence — it is untraceable and unverifiable, and claiming it as a red run is a false completion claim (violates T2).
- Every red test lands in the real suite and is committed while failing, before the fix commit (§2, §5 step 5).
- This applies to red-lighting review findings, not just new features: the probe that proves a bug **is** the regression test for its fix. Commit it red, keep it in the suite, let the fix turn it green. "Temporary probes, since reverted" means the findings have no evidence and the fixes will have no regression guard.
- A red-light results table (🔴 verdicts) is only valid if every RED row cites a committed test `file:case` and its commit sha. If running the suite right now shows no failures, nothing is red-lighted — reporting it as confirmed is fabricated evidence.
- If a test used to prove a bug turns out not to belong in the suite, that decision is the user's — surface it, do not silently delete it.

### No side workspaces — work on the PR branch

- All code changes happen on the PR branch checkout. Using worktrees is good too. **Never** apply fixes in a separate clone, or "review workspace".
- A patch that exists only in a side workspace does not exist: it is unverifiable by others, not on the PR, and will be lost. Reporting such a patch as "addressed" is a false completion claim (violates T2).
- Reviewers propose; the fix lands on the branch via the normal red → green → commit → push cycle (§5), or it is reported as an OPEN finding — never as done.

### Scope creep

- The micro spec is the scope. Every change, test, and review fix traces to a spec line (`R<id>`, `A<id>`, an acceptance criterion) or an in-scope red-lighted claim — nothing else lands.
- Scope-check feedback first and fast: every review claim, PR comment, or suggestion is checked against the micro spec before any trace or fix (§5 triage).
  - In scope → proceed.
  - Out of scope → `OUT_OF_SCOPE` with the reason, listed as a follow-up in the summary; not fixed in this change.
  - A claim is not admitted by widening the spec — a fix that needs a new requirement is a large divergence (§1).
- An agent works only on what was explicitly asked for. When in doubt whether something is in scope, leave it out and say so — finish the in-scope work rather than stopping to ask.
- Violations erode trust and create hidden regressions.
- **Do not add features** that were not in the current task or micro spec, even if they seem obviously useful.
- **Do not refactor code** outside the files directly touched by the task.
- **Do not rename** symbols, files, or directories unless renaming is the explicit task.
- **Do not add logging, metrics, or instrumentation** beyond what the spec requires.
- **Do not add comments or documentation** to code that was not part of the change, even to "improve" it.
- **Do not change formatting** in lines that were not otherwise modified.
- A pre-existing bug, performance concern, or behaviour the task does not mention → do not fix, optimise, or extend it unless the requested behaviour cannot work without it; report it as a follow-up in the summary.
- An in-scope gap (§1) → implement the reading the wording and surrounding code most directly support, state the assumption in the summary, and do not build for the other readings too.
- No error handling, fallbacks, or validation for scenarios that cannot happen — trust internal code and framework guarantees; validate at system boundaries only.
- This governs extras: every behaviour the task asks for is still implemented completely.

---

## 7. Skills

- `squash-rebase`: `~/agents/skills/SQUASH_REBASE.md`
- `/squash-rebase`: `~/agents/skills/SQUASH_REBASE.md`
- `rebase-squash`: `~/agents/skills/SQUASH_REBASE.md`
- `/rebase-squash`: `~/agents/skills/SQUASH_REBASE.md`
- `clean-check`: `~/agents/skills/CLEAN_CHECKS.md`
- `/clean-check`: `~/agents/skills/CLEAN_CHECKS.md`
- `update-main`: `~/agents/skills/MAIN_UPDATE.md`
- `/update-main`: `~/agents/skills/MAIN_UPDATE.md`
- `review-triage`: `~/agents/skills/REVIEW_TRIAGE.md`
- `/review-triage`: `~/agents/skills/REVIEW_TRIAGE.md`
