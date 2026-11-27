# Agent-Tight

> A budgeted, auditable, circuit-breaker-protected TDD workflow for coding agents, built to catch hallucinated test results.

## 📌 Overview

Coding agents fail in predictable ways:

- They read without bound, opening a hundred or more files on a single ticket — slower every round, until the context window collapses.
- When repairing review findings they re-read the source and re-verify each finding, looping in place.
- They produce false greens: the tests "pass" because the agent has talked itself into believing they pass, and bugs ship.
- Unable to find a module seam, they widen the search indefinitely and stall in the exploration phase.
- Once the session context is compressed and lost, they restart by re-reading the entire codebase — forever.

**Agent-Tight is a production-grade closed-loop skill specification, not a prompt.**

Through **budgeted throttling, checklist-driven work, circuit-breaker protection, a two-run trap gate, and an end-to-end audit trail**, it constrains the agent's discretion and makes its behavior auditable and traceable, addressing the three root causes: runaway exploration, repair loops, and false greens.

Core design principle:

> Plan before reading; checklist before editing; evidence before sign-off; stop at the limit; mark every assumption.

✅ What changes:

- **Exploration** — from 100+ unbounded file reads to a hard cap of 15 whole-file reads and 3 searches per ticket, with targeted reads first.
- **Repair** — from re-reading source and re-verifying findings to treating `fixes.md` as the single source of truth, with hunk-level reads and no re-verification of reviewed findings.
- **Hallucination guard** — from self-judged green to a two-run gate (main suite plus trap suite), where the exit code is the sole criterion, checked by an independent blind-review agent.
- **Stall fallback** — from an unbounded search for a seam to marking it `assumed` after two rounds of targeted exploration, with that assumption reviewed at the review stage.
- **Context loss** — from re-reading the whole codebase to reading the on-disk record (`plan.md` / `notes.md` / `fixes.md`) to restore state without touching the source.

## 🚀 Getting started

### 1. Supported platforms

Trae (recommended), Cursor, and Windsurf. Language- and framework-agnostic: pytest, jest, junit, go test, and any other runner.

### 2. Installation

1. Download the four skill directories under `skills/` in this repository — `tdd/`, `implement/`, `code-review/`, `test-hallucination-guard/` — each containing a `SKILL.md` and its supporting files.
2. Import the skill package through your agent's configuration panel.
3. Enable the workflow at the start of a ticket or task.

> Optional: enable it globally so every coding ticket is constrained by default.

### 3. First run: a minimal validation ticket

Create a simple requirement, for example:

> Add non-null input validation to a utility function, with positive and negative test cases.

Submit the ticket and observe:

1. The agent first writes `plan.md`, naming the files it intends to read and why.
2. Each read is logged to `notes.md`.
3. Findings are consolidated into `fixes.md` before repair begins.
4. On completion the agent runs the two-run gate: main suite plus trap suite.
5. Only after both pass does `code-review` audit whether the workflow stayed within budget and within the rules.

### 4. Artifacts (generated automatically)

```
.scratch/<ticket-id>/
├── plan.md      # Exploration plan: read targets, dependencies, seam hypotheses
├── notes.md     # Audit log: every file read and search, plus any over-budget justification
├── fixes.md     # Repair checklist: defect, file, line, fix (the single source of truth for repair)
├── main.xml     # Main suite report
├── trap.xml     # Trap (negative-case) suite report
└── runs.txt     # Test run log
```

## ⚙️ Tuning the thresholds

> All thresholds are production-tested defaults. Adjust them to project size and codebase scale in the rule sections of the corresponding skill files.

| Parameter | Default | Meaning | Tuning guidance |
| --- | --- | --- | --- |
| Whole-file reads per ticket | 15 | Maximum whole-file reads during a ticket's exploration phase; exceeding it requires a recorded justification, or review sends the work back | Small projects / microservices: lower to 8~10. Large monolithic codebases: raise to 20 (not above 25) |
| Search count | 3 | Maximum grep / code searches during exploration | Simple business logic with clear interfaces: 2. Legacy systems: 4 |
| Reads per finding | 2 | Maximum hunk-level reads for each finding during repair | Fixed; do not change — it keeps a single finding from spreading |
| Seam exploration rounds | 2 | At most two rounds of targeted search for a seam before the fallback marks it `assumed` | Leave as is; raise to 3 only for tightly coupled legacy frameworks |
| Circuit-breaker threshold | 3 | Three consecutive findings whose fix fails its test stops the ticket and escalates to a human | Prefer stability: 2. Allow more retries: 4 |
| Trap-case ratio | ≥30% | Negative / trap cases must be at least 30% of the total, as a defense against hallucinated tests | Core business logic: raise to 40%. Utility functions: keep at 30% |

### Tuning principles

1. Lower thresholds converge faster and consume fewer tokens, but complex work is more likely to trigger an assumption or the circuit breaker.
2. Higher thresholds give the agent more room, but invite the old problem of diffuse, slow behavior.
3. Tune the whole-file read cap and the search cap first; leave the circuit-breaker threshold and the seam rounds alone unless there is a reason to touch them.

### Exemption rule (fixed; not recommended to change)

The full two-run gate is skipped automatically when the diff contains no executable code change — comments, documentation, or README edits only.

## 📖 The workflow at a glance

1. **TDD exploration** — write the plan → targeted search and reads → record each read in `notes.md`; hard budget; mark `assumed` after two rounds without a seam.
2. **Implementation** — build to the plan; add positive cases and trap cases.
3. **Code review** — emit a structured `fixes.md`; audit the exploration budget and read behavior for compliance.
4. **Repair** — work strictly from `fixes.md`. Findings in the same file with non-overlapping line ranges may be batched; everything else is serial. Do not re-verify reviewed findings.
5. **Guard gate** — two-run tests, `tree_hash` verification of the trap files, independent blind review; the exit code is the sole criterion.
6. **Circuit breaker** — on reaching the failure threshold, automated repair stops and a standardized report goes to a human.

## ⚠️ Known limitations and risks

1. The gate runs by default, so even line-level changes (a constant, a copy string) trigger the full two-run cycle. That is expensive; a lightweight guard branch can be added as an optional extension.
2. There is no built-in model integration. Any model — GLM, DeepSeek, Qwen, or a private API — works provided it supports tool calling and streaming inference.
3. This skill is a behavioral constraint specification. It cannot eliminate a model's underlying hallucination; it reduces the probability substantially through process, evidence, and verification.
4. A seam marked `assumed` requires human review at the review stage. It cannot be delegated entirely to the agent.

## 🧩 Provenance and originality

This skill set draws on two sources. Keep this notice when using or redistributing it:

| Skill | Origin | Nature |
| --- | --- | --- |
| `test-hallucination-guard` | **Original work** | Original. Its core mechanisms are the two-run gate, exit-code evidence, `tree_hash` trap-integrity verification, an independent blind-review agent, a negative-case quota, and trap isolation |
| `tdd` | Matt Pocock | From Matt Pocock's skill collection, **adapted** |
| `implement` | Matt Pocock | From Matt Pocock's skill collection, **adapted** |
| `code-review` | Matt Pocock | From Matt Pocock's skill collection, **adapted** |

**`test-hallucination-guard` is original work.** It turns the claim "the tests pass" into a gate that can be evidenced and audited. That is the differentiating core of this workflow and the only guard added on top of the three upstream skills.

`tdd`, `implement`, and `code-review` originate with Matt Pocock and were adapted from their original versions as follows:

- **Exploration budget** — a `plan.md` written before any read, a `notes.md` read-audit log, and hard caps on whole-file reads and searches.
- **Repair phase** — a checklist-driven `fixes.md` repair loop, with a per-finding read budget, a disjoint-hunks batching rule, and a circuit breaker.
- **Gate integration** — explicit invocation points, artifact-based completion criteria, a verdict line at review, and the exemption declaration.
- **Review audit** — `code-review` gains disposal rules for exploration-budget, repair-budget, and read-behavior compliance.
- **Seam fallback** — when no seam can be confirmed, mark it `assumed` and proceed, leaving it to human review.

## 🤝 Contributing and feedback

Issues are welcome:

- Threshold-tuning feedback
- Adaptations for additional test frameworks
- Rule-logic bugs and edge-case improvements

Pull requests are welcome: lightweight guard extensions, adapter scripts, and demo projects.

---

> Stop the agent from wandering. Make coding agents auditable, bounded, and trustworthy.
