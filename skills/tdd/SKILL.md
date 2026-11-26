---
name: tdd
description: Test-driven development. Use when the user wants to build features or fix bugs test-first, mentions "red-green-refactor", or wants integration tests.
---

# Test-Driven Development

TDD is the red → green loop. This skill is the reference that makes that loop produce tests worth keeping: what a good test is, where tests go, the anti-patterns, and the rules of the loop. Every section applies on every cycle: consult them before and during the loop, not after.

When exploring the codebase, read `GLOSSARY.md` (if it exists) so test names and interface vocabulary match the project's domain language, and respect ADRs in the area you're touching.

## What a good test is

Tests verify behavior through public interfaces, not implementation details. Code can change entirely; tests shouldn't. A good test reads like a specification: "user can checkout with valid cart" tells you exactly what capability exists, and it survives refactors because it doesn't care about internal structure.

See [tests.md](tests.md) for examples and [mocking.md](mocking.md) for mocking guidelines.

## Seams: where tests go

A **seam** is the public boundary you test at: the interface where you observe behavior without reaching inside. Tests live at seams, never against internals.

**Test only at pre-agreed seams.** Before writing any test, write down the seams under test and confirm them with the user. No test is written at an unconfirmed seam. You can't test everything, so agreeing the seams up front is how testing effort lands on the critical paths and complex logic instead of every edge case.

Ask: "What's the public interface, and which seams should we test?" Give each proposed seam a one-line note on what it catches and what it misses.

When the shape of that interface is itself in question (how deep the module is, where the seam belongs, what the interface should expose), call the Skill tool with "codebase-design" for the vocabulary. It is the shared source of the module, interface, depth, seam, adapter, leverage and locality terms, and it is a reference to consult, not a session to run.

**Seam fallback.** If two rounds of targeted exploration have not produced a definitive seam, stop exploring. Propose the most reasonable seam from the module's public surface, mark it `assumed` in the trap manifest (see `skills/test-hallucination-guard/SKILL.md`, red-phase section), and proceed to the red phase. Assumptions are validated in code review; an unbounded search for a "perfect" seam is a bug, not diligence.

## Anti-patterns

- **Implementation-coupled**: mocks internal collaborators, tests private methods, or verifies through a side channel (querying the database instead of using the interface). The tell: the test breaks when you refactor but behavior hasn't changed.
- **Tautological**: the assertion recomputes the expected value the way the code does (`expect(add(a, b)).toBe(a + b)`, a snapshot derived by hand the same way, a constant asserted equal to itself), so it passes by construction and can never disagree with the code. Expected values must come from an independent source of truth: a known-good literal, a worked example, the spec.
- **Horizontal slicing**: writing all tests first, then all implementation. Bulk tests verify _imagined_ behavior: you test the _shape_ of things rather than user-facing behavior, the tests go insensitive to real changes, and you commit to test structure before understanding the implementation. Work in **vertical slices** instead: one test → one implementation → repeat, each test a **tracer bullet** that responds to what the last cycle taught you.

## Rules of the loop

- **Red before green.** Write the failing test first, then only enough code to pass it. Don't anticipate future tests or add speculative features.
- **One slice at a time.** One seam, one test, one minimal implementation per cycle.
- **Refactoring is not part of the loop.** It belongs to the review stage (see the `code-review` skill), not the red → green implementation cycle.

## Red-phase exploration budget

Before reading any source file, write `.scratch/<feature>/plan.md` containing:

1. A one-sentence restatement of what the ticket requires.
2. The expected affected files (≤ 5), each with a note on what to look for in it.
3. The files needed to confirm the seam (≤ 3) and exactly what to check in each.

Then read only files on that list. Confirm interface shapes with targeted grep (`grep -n "def symbol" src/…`), not by reading whole files. Append one line per read to `.scratch/<feature>/notes.md` in the form `read #N: <path> — what it confirmed`; this log is the audit trail for the budget.

Budget: at most 15 whole-file reads and 3 searches per ticket. Reaching the cap does not mean reading faster — it means writing the plan and moving to the red phase. If the cap must be exceeded, first append two lines to `plan.md` explaining why and what will be read, then continue. Exploration stays strictly within the modules the ticket names; historical tickets and previous guard artifacts are not reference material.

## Guarding the green

At the end of each red phase, Read `skills/test-hallucination-guard/SKILL.md` for its red-phase section only: author the trap cases, register the `trap` and `negative` markers, record `scope` + `hash_algo` + `tree_hash` in the manifest, report the running negative ratio. Seams and assertion quality stay this skill's business. The green-phase gate is not yours to trigger — it belongs to `/implement`'s completion criterion.
