# Agent Guidelines

Rules and expectations for all AI agents working in this repository tree.
These guidelines apply to every project unless a project-level `Agents.md`
explicitly overrides a section.

## Git convention

## Title conventions
- Under 70 characters
- Format: `type(scope): description`
- Types: `fix`, `feat`, `refactor`, `docs`, `test`, `chore`
- Use description for details, not the title

## Description rules
- Lead with WHY, not WHAT
- List all changed files/endpoints/components in a structured table if the PR is large
- Explicitly list what was NOT changed and why (scope boundaries)
- If there are pre-existing issues in untouched files, note them as out of scope
- Link to spec if one exists
- Update the description when the PR changes — stale descriptions cause review friction


---

# Spec

## Micro Spec Convention

Every non-trivial task begins with a micro spec written **before** any code.

### File location

```
<project>/docs/<YYYYMMDD>_<short-task-slug>.md
```

### Required sections

| Section | Purpose |
|---|---|
| **Goal** | One sentence — what problem this solves and why |
| **Scope** | What is in and explicitly out of scope |
| **Interfaces** | Public API / data shapes / events exposed by this work |
| **Behaviour** | Numbered acceptance criteria, written as observable facts |
| **Error cases** | How each failure mode is detected and surfaced |
| **Test plan** | List of unit tests required; maps 1-to-1 to Behaviour items |
| **Open questions** | Assumptions that need a decision before or during implementation |

### Rules

- No code is written until the micro spec exists and is committed.
- The spec is the source of truth. If code diverges, update the spec first
  and get acknowledgement before continuing.
- In markdown docs, do not manually hard-wrap prose lines; keep paragraph text on a single line and let the editor handle visual wrapping.
- Acceptance criteria drive the test plan — every criterion maps to at least
  one test. Missing test → missing criterion or vice versa.
- Interfaces in the spec are expressed in language-neutral pseudocode or
  plain English unless the project has an established language convention.

---

# testing

## Unit Test Standard

**Target: 97 % line coverage, 100 % branch coverage on public interfaces.**

### What must be tested

- Every public function / method / endpoint.
- Every error path listed in the micro spec's *Error cases* section.
- Every boundary value (zero, one, max, overflow where relevant).
- Every state transition if the module is stateful.

### What is explicitly excluded from the 97 % target

- Auto-generated code — mark with the project's coverage-skip mechanism.
- Third-party adapters that are thin pass-throughs with no logic.
- Entry-point bootstrapping that wires the dependency graph.

Exclusions must be listed in the micro spec under *Scope > Out of scope*.

### Test structure

```
tests/
```

Each test file mirrors its source file in path and name.

### Red / Green / Refactor cycle

Each acceptance criterion from the micro spec is implemented through exactly
one cycle:

1. **Red** — write the test, run the suite, read the failure output.
   The failure must be an assertion failure against the behaviour under test,
   not a setup or import error. Fix setup issues until the failure is clean,
   then stop.
2. **Green** — write the smallest production change that turns that test
   green. Run the full suite to confirm no regressions.
3. **Refactor** — improve the code while the suite stays green.

An agent must never write production code before observing a red failure.
Skipping the red step invalidates the test as a safety net.

### Test anatomy (AAA)

```
test <behaviour>_<condition>_<expected outcome>:
    // Arrange
    set up inputs and dependencies

    // Act
    result = call the unit under test

    // Assert
    verify result matches expectation
```

- One logical purpose per test.
- Tests are deterministic: no randomness, no wall-clock time, no network —
  inject or stub these at the boundary.
- Tests never share mutable state across test functions.

### Coverage gate

CI must fail if coverage drops below 97 % on any PR touching production code.
Enforce this through the project's standard coverage tooling.

---

# design conventions

##  SOLID Design Principles

This project applies all five SOLID principles:

| Abbr | Full name | One-line rule |
|---|---|---|
| **SRP** | Single Responsibility | One reason to change per module, class, or function |
| **OCP** | Open / Closed | Open for extension, closed for modification |
| **LSP** | Liskov Substitution | Subtypes must be drop-in replacements for their base type |
| **ISP** | Interface Segregation | Many narrow interfaces over one wide one |
| **DIP** | Dependency Inversion | Depend on abstractions, not concretions |

Call the principle out by abbreviation in code review comments and micro
specs (e.g. "this violates SRP — split the parse and persist steps").

### SRP — Single Responsibility

A module, class, or function has exactly one reason to change. When a unit
does two things — parsing *and* persisting, validating *and* formatting —
split it. The name of each unit should make its single responsibility
obvious without needing to read the body.

### OCP — Open / Closed

New behaviour is added by extending the system, not by editing existing
code. Achieve this through composition, strategy objects, and well-defined
extension points. A change request that requires editing an existing class
body is a signal the abstraction boundary is in the wrong place.

### LSP — Liskov Substitution

Every subtype must honour the contract of its base type. A caller that holds
a reference to the base type must observe identical behaviour regardless of
which concrete subtype it receives. Violations: a subtype that throws where
the parent does not, returns a narrower type, or silently ignores a method.

### ISP — Interface Segregation

No implementor should be forced to define methods it does not use. Split
wide interfaces into focused ones. Clients depend only on the slice of the
interface they actually call.

### DIP — Dependency Inversion

High-level policy modules do not import low-level detail modules. Both
depend on a shared abstraction. Concrete implementations are injected at
the composition root — never constructed inside business logic.

---

## Engineering Quality Standards

### Code style

- Follow the language's idiomatic style guide.
- Maximum function length: **30 lines** of logic (excluding blank lines and
  comments). If longer, extract a helper with a clear name.
- Maximum file length: **400 lines**. Larger files signal more than one
  concept in the file.
- No commented-out code committed. Use version control instead.

### Naming

- Names are pronounceable, unambiguous, and domain-specific.
- No abbreviations unless universally understood in the domain.
- Boolean names start with `is_`, `has_`, `can_`, or `should_`.

### Error handling

- Never silently swallow exceptions. Log and re-raise, or convert to a
  typed domain error with context.
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

  <body — why, not what; optional>
  ```
  Types: `feat`, `fix`, `refactor`, `test`, `docs`, `chore`.
- No merge commits on feature branches; rebase onto main before merging.
- Branch names: `<type>/<short-slug>`.

### Pull / merge requests

- PR description links to the micro spec doc.
- Checklist before requesting review:
  - [ ] All acceptance criteria from the micro spec are met.
  - [ ] Coverage gate passes locally.
  - [ ] No new lint warnings introduced.
  - [ ] Micro spec updated if scope changed during implementation.
  - [ ] CHANGELOG entry added for user-visible changes, if the repository keeps a changelog. Never create one.

---

## Agent Workflow

When an AI agent picks up a task it **must** follow this order:

1. **Read** the relevant micro spec (or create one if absent).
2. **Read** existing code in the affected area before writing anything.
3. **Write or update** the micro spec if the task is new or scope changes.
4. **Red** — write one test, run the suite, confirm that test fails for the
   right reason. A test that cannot be seen to fail proves nothing.
5. **Green** — write the minimum production code needed to make that test
   pass. No more.
6. **Repeat** steps 4–5 for each acceptance criterion in the micro spec.
7. **Refactor** — with all tests green, clean names, split large functions,
   remove duplication. Run the suite after every refactor step.
8. **Commit** in atomic commits following the git hygiene rules above.
9. **Do not deploy** unless the user explicitly says so.

An agent must stop and surface an open question rather than guess when:
- A spec section is ambiguous.
- An interface from another module is missing or contradicts the spec.
- The 97 % coverage target cannot be reached without unreasonable stubbing.

### Response style

- Keep responses short and to the point.
- Prefer direct answers over long explanations.
- Include only the detail needed for the current task unless the user asks for more.

---

## What Agents Must Never Do

### Scope creep — the hardest rule

An agent works only on what was explicitly asked for. When in doubt, do less
and ask. Violations erode trust and create hidden regressions.

- **Do not add features** that were not in the current task or micro spec,
  even if they seem obviously useful.
- **Do not refactor code** outside the files directly touched by the task.
- **Do not rename** symbols, files, or directories unless renaming is the
  explicit task.
- **Do not add logging, metrics, or instrumentation** beyond what the spec
  requires.
- **Do not add comments or documentation** to code that was not part of the
  change, even to "improve" it.
- **Do not change formatting** in lines that were not otherwise modified.
