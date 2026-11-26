---
name: test-hallucination-guard
description: "Gate the claim \"tests pass\" — exit-code evidence, a separate trap run, an isolated verifier. Read at the end of a red phase for trap authoring, and by the implementer as its completion criterion."
disable-model-invocation: true
---

# Test Hallucination Guard

A companion gate for `/tdd`, `/implement`, and `/code-review`. It does not teach testing — `/tdd` owns what a good test is. This skill answers one question: **is the claim "tests pass" backed by evidence, or did the agent talk itself into it?**

Each rule below is labelled **Mechanism** (the harness enforces it) or **Convention** (the agent is asked to hold it). Only mechanisms get called non-negotiable; a convention that breaks is a finding, not a violation.

| Rule | Kind | What actually enforces it |
| --- | --- | --- |
| Exit code from the harness | Mechanism for how it is produced; Convention for authenticity | The runner process supplies it; a typed line is indistinguishable |
| Two separate runs | Mechanism | Two invocations, trap dir excluded from the main one |
| Machine-readable reports | Mechanism | `--junitxml` / `--json --outputFile`; no report, no verdict |
| Trap fails by assertion, not by error | Mechanism where the report splits `failure`/`error` (junit); elsewhere message match — see rule 4 | The report's per-case status, or a match against the manifest's `expects_message` |
| Trap integrity | Mechanism | The verifier rehashes the trap tree against the manifest, using the formula the manifest states |
| Implementer can't read traps | Mechanism where the harness enforces read scope; Convention otherwise | Directory permissions, or a manifest-only view (ids and expectations, no source) |
| Isolated verifier | Mechanism | A fresh sub-agent with a whitelisted brief |
| Test write-lock | Convention | The implementation brief; a diff check at review |
| Negative-case quota | Convention, countable | A `negative` marker; the verifier counts the report or a collect-only listing |
| Manifest format comes from this document only | Mechanism (the format is fully specified here); Convention (refraining from reading historical artifacts) | This document specifies the format end to end; nothing else is authoritative |
| Unconfirmed seam is marked `assumed` in the manifest | Mechanism (manifest field exists); Convention (marking honestly instead of silently proceeding) | The manifest carries a `seam` field; whether it is marked `assumed` rests on the author |
| **Gate attendance** | Convention — the weak link | `/implement` cannot finish without both report files |

## When it runs

| Phase | Who reads this skill | What happens |
| --- | --- | --- |
| **Red** — after the test is written | `/tdd` | Author traps, register the `trap` and `negative` markers, record the tree hash, report the running negative ratio. No exit-code gate yet: there is no implementation to run. |
| **Green** — when implementation is claimed done | `/implement`'s completion criterion | The full gate: two runs, both reports, tree hash, isolated verifier. |

The green-phase trigger has exactly one owner: `/implement`. `/tdd` stops at the red phase — it never calls the gate, so there is no second trigger point to disagree with.

Nothing else about the **red** phase is this skill's business — seams, assertion quality, and coverage belong to `/tdd`.

## Who runs what

The **orchestrator** — the session driving `/implement`, which may be the main agent — runs both commands. It is not the implementation sub-agent, which is barred from the trap directory, and not the verifier, which reads reports and may only list tests (collection executes nothing).

The orchestrator's exit-code lines are **appended by the shell**, labelled by run, never typed:

```
pytest -m "not trap" --junitxml=.scratch/<feature>/main.xml; echo "main EXIT:$?" >> .scratch/<feature>/runs.txt
pytest -m trap --confcutdir=tests/traps --junitxml=.scratch/<feature>/trap.xml; echo "trap EXIT:$?" >> .scratch/<feature>/runs.txt
```

`--confcutdir` is the pytest half of isolation: without it, `tests/conftest.py` and the root `conftest.py` load into the trap run automatically, and a fixture the implementer can edit reaches the traps unimpeded. Other runners need their equivalent — a separate config or root for the trap directory, or `setupFiles` that don't climb out of it.

jest takes a path argument and a JSON report instead of a marker:

```
jest --testPathIgnorePatterns=tests/traps --json --outputFile=.scratch/<feature>/main.json; echo "main EXIT:$?" >> .scratch/<feature>/runs.txt
jest tests/traps --json --outputFile=.scratch/<feature>/trap.json; echo "trap EXIT:$?" >> .scratch/<feature>/runs.txt
```

Its isolation is weaker than pytest's: `setupFiles` and `globalSetup` still run for the trap run, so they must not import anything the implementer owns.

The label, not the line order, identifies the run — retries and reruns can append freely without a "first line is main" guess.

Provenance is a convention, not a mechanism: a typed `main EXIT:0` looks exactly like an appended one, and the verifier holds no way to tell them apart. So this rule says where the number is *supposed* to come from, and the verifier records it as an assumption of the same trust domain rather than pretending to verify it.

The verifier's independence rests on its **blind brief**, not on being the executor: it sees no implementation diff and no authoring rationale. What the verifier must never do is take someone's word for a number it can read from a report.

Attendance is this skill's honest weak point: whether the gate runs at all depends on `/implement` reaching its completion criterion. That criterion is therefore stated as an artifact requirement — both report files, both exit codes recorded — so a skipped gate fails the work rather than passing silently.

One residual risk this skill cannot close: the orchestrator runs the commands, so nothing at the skill layer stops it from re-running with a doctored setup and producing a legal-looking report. The tree hash covers the trap files, and the verifier cross-checks counts against statuses, but "a valid run of the wrong thing" needs harness-level isolation — the runner writing reports the orchestrator can only read. Until then, treat orchestrator and verifier as the same trust domain.

## Three principles

1. **The runner judges, not the agent.** Pass/fail is the native runner's exit code. Agent interpretation is never evidence.
2. **Writer does not judge, judge does not write.** Whoever wrote the tests never signs them off; whoever wrote the implementation never edits them.
3. **Every green needs an artifact.** Exit code, failing-case list, trap result — one machine-readable trace per claim.

## The two runs

The gate breaks whenever trap cases live inside the main suite: a deliberate failure poisons the exit code, so "exit = 0" and "traps fail" can never both hold. The suite therefore runs **twice**, as two separate invocations.

| Run | What it is | Required result |
| --- | --- | --- |
| **Main run** | The real suite, trap directory excluded | exit code = 0, report `main.<ext>` |
| **Trap run** | Only the seeded trap cases, isolated from ancestor config | runner's *tests failed* code, every trap `failed` (or `expects_message` match), report `trap.<ext>` |

Each run writes a machine-readable report; a run without one is unverified, not green.

| Runner | Report flag | Negative count comes from |
| --- | --- | --- |
| pytest | `--junitxml=.scratch/<feature>/<run>.xml` | `--collect-only -q -m "negative and not trap"` |
| jest | `--json --outputFile=.scratch/<feature>/<run>.json` | the report's `assertionResults[].fullName` matching the naming tag — `--listTests` lists files only, never cases |
| other | the runner's structured-report flag | the runner's structured report |

**Adapting a new runner.** The rules above are runner-agnostic; any runner fits by
answering three questions, in this order:

1. **Exclusion flag** — how to run the suite without the trap directory, and the
   trap directory alone. Look for a marker filter, a path ignore pattern, or a
   second config file. Verify: run both ways and check the case lists in the
   reports do not overlap.
2. **Structured report** — the flag that writes per-case machine-readable results
   to a file (`--junitxml`, `--json --outputFile`, or the runner's equivalent).
   No such flag means the runner cannot pass this gate; do not fall back to
   parsing stdout.
3. **Status vocabulary and exit codes** — how the report spells "assertion
   failure" versus "error" (import failure, collection error), and which exit
   code means "tests ran and failed". Where the report does not split the two,
   `expects_message` substring matching carries the judgement; where the exit
   code does not distinguish failure from interruption, require exit ≠ 0 plus
   rule 4.

Record all three answers in the trap manifest as comments, so the verifier checks
against the same vocabulary instead of re-deriving it. The pytest and jest blocks
above are worked examples of these three answers, not a closed list.

The cycle is green only when all five hold:

1. Main run exit code = 0.
2. Trap run exit code is the runner's *tests failed* code — pytest: `1`, not `2` (interrupted / collection error). Where the runner does not distinguish (go test, cargo test), require exit ≠ 0 and not a build/compile failure, and let rule 4 carry the judgement.
3. The trap run's case list reconciles with the trap manifest one-for-one: every id present. A manifest `id` is the case's name exactly as the report spells it — pytest's `name` attribute in junit, jest's `title`/`fullName` — not a label the author invents. A parameterized trap expands to one manifest entry per parameter, `trap_x[a]`, `trap_x[b]`, so the count stays honest.
4. Every trap case reports `failed` — the report's assertion-failure status. Where the report separates the two, use it: junit `<failure>` versus `<error>` (import failure, missing fixture, collection error). Where the report does not — jest reports both as `failed`, go and cargo do not split at all — the trap must fail with the message recorded in the manifest as `expects_message`, and the verifier checks that it appears as a substring of the reported failure text. Substring, not equality: reports wrap the message in stack traces and paths. A trap that errors for any other reason does not count: it proves the file breaks, not that the suite's assertions can fail.
5. Neither run is skipped, uncollected, or "0 tests ran", and both reports exist.

A trap reported passed, a trap absent from the run, a suite that collected zero tests, or a missing report invalidates the whole result regardless of the main exit code. Both runs go through the project's native runner (Mechanism); nothing in the agent path edits tests, comments out assertions, or rewrites exit-code handling mid-run.

Exclusion is a runner feature, not a promise: a marker (`pytest -m "not trap"` / `-m trap`), a path pattern (`--testPathIgnorePatterns=tests/traps`), or a second config file. Registering the `trap` and `negative` markers edits runner config — under `--strict-markers` an unregistered marker fails collection — and that edit belongs to the **red** phase, by the Test Author. The green-phase lock freezes it along with the test files.

## Exit-code evidence

The code comes from the harness, never from the transcript.

- Read the runner tool's **exit-code field**. Never infer pass/fail from pasted stdout.
- If the harness surfaces only text, the run isn't over until the code is printed on its own line and appended to `runs.txt` rather than retyped — `pytest …; echo "main EXIT:$?" >> .scratch/<feature>/runs.txt` (POSIX) or `pytest …; "main EXIT:$LASTEXITCODE" >> .scratch/<feature>/runs.txt` (PowerShell).
- Filtered, truncated, or summarized output is not evidence. Keep the raw log path.

## Trap cases

Trap cases are seeded failures — invalid input, a deliberately wrong expected value, a known-broken path. They exist to prove the suite can fail.

- **Count**: 2–5 per feature suite, written during the red phase of the feature's first slice.
- **Seam**: a trap rides the same pre-agreed seam as the suite it guards — at a confirmed (or assumed, see below) seam. It is an extra case at that seam, never a new seam.
- **Location**: traps live outside the implementation agent's read scope — their own directory (e.g. `tests/traps/`), excluded from the main run, never named or pasted into the implementation brief. Where the harness cannot restrict reads, downgrade to **manifest-only**: the implementer sees ids and expected statuses, never case bodies. The verifier can read them in either mode.
- **Self-contained**: the trap directory carries its own `conftest.py` and fixtures, and the trap run is cut off from ancestors (`--confcutdir`, or the runner's equivalent). Without that cut, pytest loads every `conftest.py` above the trap directory, so a root fixture the implementer can edit reaches the traps without any import — the exact hole a shared fixture would open. Trap cases also take no fixture parameters from outside their own directory.
- **Integrity**: the hash covers the whole trap tree — every file under it, `conftest.py` and fixtures included — not one file per trap. The manifest stores a `tree_hash`; the verifier recomputes it at sign-off. A mismatch means the trap directory changed and the run is invalid.

  The formula is fixed so both sides produce the same number:

  ```
  tree_hash = sha256( concat, over files sorted by relative path,
                      of  f"{rel_posix_path}\t{sha256(raw_file_bytes)}\n" )
  ```

  Paths are relative to the trap directory, POSIX separators, compared as UTF-8 bytes. File bytes are hashed raw — no line-ending normalization. Every entry ends with `\n`, including the last. Any other formula is a mismatch by construction, so state the one used in the manifest's `hash_algo` line.
- **Manifest** — the Test Author writes `expects` and `tree_hash` at authoring time. The orchestrator runs the commands and records their exit codes in `.scratch/<feature>/runs.txt`. The verifier fills `actual` and confirms the recorded codes against the reports:

```yaml
# .scratch/<feature>/traps.yaml
scope: tests/traps/
hash_algo: relpath\t+sha256(content), sorted, newline-joined
tree_hash: sha256:3f9a…           # taken at authoring time; verifier rehashes
cases:
  - id: trap_negative_quantity
    seam: cart.checkout
    expects: failed               # the report vocabulary: junit <failure> / jest failed
    expects_message: "total must be 15"   # required where the report can't split failed/error
    actual:                       # verifier only, from trap.<ext>
```

```
# .scratch/<feature>/runs.txt — appended by the shell, one labelled line per run
main EXIT:0
trap EXIT:1
```

- **Reveal**: traps surface only in the verification step, never during implementation.

**Manifest format is defined by this document alone.** The example above is the authoritative format. Do not read artifacts from previous tickets or features to "confirm" the format — this document is the only reference, and reading historical guard artifacts is an exploration-cost bug, not a safety step.

**Seam fallback.** If two rounds of targeted exploration have not produced a definitive public seam, propose the most reasonable seam from the module's public surface, write it into the manifest's `seam` field, and mark it `assumed`:

```yaml
seam: cart.checkout  # assumed — unconfirmed, see notes.md read log
```

Proceed to write the red-phase tests against the assumed seam. Do not expand the search scope chasing a "perfect" seam; the assumption is validated later in code review. Trap cases that ride on an assumed seam must be flagged in the verifier brief, so the verifier knows the seam itself is unconfirmed.

## Negative-case quota

At least 30% of a feature's test cases are negative: boundary values, null/empty input, error paths, permission denials, invalid operations.

Negative cases carry a `negative` marker — a pytest marker, a jest tag in the test name (`test_checkout_rejects_empty_cart`), or whatever the runner's report can list. The verifier counts them from the main run's report, or from a collect-only listing it runs itself where the report omits the marker — never from a ratio someone reports. When it runs the listing, it keeps it at `.scratch/<feature>/collect.txt` (`pytest --collect-only -q -m "negative and not trap"`).

Measured **per feature suite, not per cycle** — `/tdd` writes one test per slice, so no single red phase can carry the quota. Report the running ratio at the end of each red phase, with an early warning at **20%**: under it, the suite is on course to miss the bar and the next slices should add negative cases. Gate on ≥ 30% only at sign-off. A happy-path-only suite fails verification.

## Roles

Three roles, three contexts. Separation is real only when the contexts are real, so verification is a **sub-agent dispatch**, not a change of hat.

1. **Test Author** (`/tdd`) — writes tests at confirmed seams. Runs a new test only to confirm it is red; never signs off the green.
2. **Implementation Agent** (`/implement`) — writes production code; reads tests read-only; never edits tests or runner config.
3. **Verification Agent** — a fresh sub-agent spawned with a single `Agent` call, handed nothing but:
   - the two structured reports (`.scratch/<feature>/main.<ext>`, `trap.<ext>`),
   - the run record (`.scratch/<feature>/runs.txt`) with its labelled exit lines (`main EXIT:`, `trap EXIT:`),
   - the trap manifest and read access to the trap directory (for the tree hash),
   - a note in the brief of any trap case riding on a seam marked `assumed`.

   Exclusion is checked on the reports, not on faith: the main report must contain no trap id, and the trap report must contain every manifest id. A main run that "forgot" `-m "not trap"` shows up as trap cases inside `main.<ext>`.

   It never receives the implementation diff, the authoring rationale, the spec, or any claim about what should pass. It may run a collect-only or list-tests command — those execute nothing — where the report omits the marker. Its scope is evidence only: do the exit codes, statuses, hashes, and counts hold? It does not judge coverage or test quality — that belongs to `/code-review`'s Spec and Standards axes. It returns a verdict — green / not green, quoting the numbers — and the main agent reports that verdict rather than overruling it.

   For high-stakes changes, spawn a second verifier alongside the first, on a different model if the platform allows. Disagreement fails closed: any verifier saying not-green means not-green.

## What this catches

Detection is this skill's job; the underlying test-quality rules stay in `/tdd` and are not restated here.

- **Self-judging** — one agent writes, runs, and signs off. → the sub-agent dispatch.
- **Simulated green** — a pass reported with no exit code, or inferred from output text. → the evidence rule.
- **Trap immunity** — a suite that cannot fail, so green proves nothing. → the trap run.
- **Trap dilution** — a trap failing for the wrong reason (import error, missing fixture). → rule 4 of the two runs.
- **Trap weakening** — a trap file, its conftest, or its fixtures edited so it still fails but proves less. → the tree hash.
- **Happy-path bias** — nominal inputs only, no failure paths. → the negative marker and count.
- **Summary inflation** — "all tests pass" when a subset ran, or skips counted as passes. → rule 5 of the two runs.

Drift in the non-trap test files is out of scope here: this gate holds no baseline for them. `/code-review`'s diff check catches it.

## Integration

- **`/tdd`** — end of red phase only: author traps in their own directory, register the markers, record `scope` + `hash_algo` + `tree_hash`, report the running negative ratio. Nothing at the end of a green phase; see `/implement`.
- **`/implement`** — owns the green-phase trigger: run both runs, dispatch the verifier, and treat the artifacts as its completion criterion. The implementation agent works under the write-lock convention and runs the suite only through the native runner. Implementation is not finished until `main.<ext>`, `trap.<ext>`, and `runs.txt` exist; a missing artifact means the work is incomplete, not green. The gate has exactly one exemption, mechanical and diff-based: a diff touching no executable code (docs, comments, copy strings) is marked `no-code-change` and skips the gate. There is no "light mode" — the ticket's size never decides the gate; only the diff's contents do.
- **`/code-review`** — the guard emits a **verdict**, not a third axis. State it as one line above the review's `## Standards` / `## Spec` headings, and file any test-integrity finding under the axis it belongs to (coverage gap → Spec, test quality → Standards). The two axes stay unmerged and unreranked. The review is also where a missing verdict becomes visible: if `runs.txt` or either report is absent, the verdict is **unverified**, filed as a Spec-axis finding — never left blank.

## Per-cycle checklist

1. Red phase: self-contained trap directory written, markers registered, manifest (`scope`, `hash_algo`, `tree_hash`) written, negative cases marked, running ratio reported (warn under 20%).
2. Implementation: test files and runner config untouched, no test skipped.
3. Main run: native runner, labelled `main EXIT:` line appended by the shell, **exit = 0**, `main.<ext>` written, no trap id inside it.
4. Trap run: separate invocation, isolated from ancestor config (`--confcutdir` or the equivalent), labelled `trap EXIT:` line appended, exit = the runner's *tests failed* code (where the runner doesn't distinguish: exit ≠ 0 and not a build failure), every trap `failed` rather than `error` — or, where the report can't split them, matching `expects_message` — and `trap.<ext>` written.
5. Integrity: recomputed tree hash matches the manifest.
6. Sign-off: negative ratio ≥ 30% for the feature suite, counted from the report or a collect-only listing.
7. Verification: verdict from the isolated sub-agent, quoting both exit codes, the trap statuses, and the hash check.
8. Only then is the cycle green.

If any of 1–7 is unverified, report the cycle as **unverified**, not passing.
