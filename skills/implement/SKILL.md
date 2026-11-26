---
name: implement
description: "Implement a piece of work based on a spec or set of tickets."
disable-model-invocation: true
---

Implement the work described by the user in the spec or tickets.

Use /tdd where possible, at pre-agreed seams.

## Exploration boundaries

Plan before reading. Before opening any source file, produce `.scratch/<feature>/plan.md`: the ticket restated in one sentence, the expected affected files (≤ 5) with what to look for in each, and the ≤ 3 files needed to confirm the seam. Read only files on that plan; use targeted grep to confirm interface shapes instead of whole-file reads. After planning, freeze the plan into a 5-line scope summary and work from that summary — do not re-read raw tickets or reference files to re-confirm what the summary already states.

Append one line per read to `.scratch/<feature>/notes.md` (`read #N: <path> — what it confirmed`). Hard cap: 15 whole-file reads and 3 searches per ticket. Hitting the cap means move to implementation, not keep reading. If the cap must be exceeded, first record why and what in `plan.md`, then continue. Exploration is limited to the modules the ticket names; do not read historical tickets, previous guard artifacts, or unrelated directories for reference. A missing or stale `plan.md` means the exploration phase is not done.

## Fix loop protocol

After `/code-review` hands back `fixes.md`, repair is driven by that file — one
finding at a time, never by re-establishing the whole picture first.

- **One finding at a time.** Pick the first unchecked line in `fixes.md`. Locate the
  fix site with grep; read only the surrounding hunk (`sed -n 'X,Yp'`, or an
  offset/limit read), never the whole file. Fix exactly that finding.
- **Verify per finding.** Run the single test file that covers the fix site; the
  runner's exit code decides, as everywhere else. Append one line to `fixes.md`:
  `- [x] F1 — fixed, <test> exit 0`. Then move to the next finding.
- **Batching, only by mechanical criteria.** Findings may be fixed in one batch only
  when ALL hold: the findings live in the same file, and their fix hunks are
  disjoint — the line ranges recorded in `fixes.md` do not overlap or touch.
  Disjoint hunks cannot interfere, so batch the edits, run the covering test file
  once, then check off each line in `fixes.md`. Overlapping hunks, cross-file
  findings, or any finding whose fix changes what a disjoint hunk reads go back to
  one-at-a-time. Disjointness is read off the recorded line ranges — it is never a
  judgment call about whether the changes "feel independent".
- **Circuit breaker.** Stop and escalate instead of grinding when either holds:
  three consecutive findings whose fix failed verification, or the same test file
  failing twice in a row on the same finding. On break, append to `fixes.md` what
  was tried and why it failed, leave the remaining lines unchecked, and report the
  ticket back to the user with the record — do NOT auto-revert, do NOT keep trying
  alternative fixes. Direction-level doubt is a user decision, not an agent one.
- **No batch pre-reading.** Do NOT read "all files to be changed" up front.
  Batch pre-reading is the exploration anti-pattern in a second costume: the reads
  crowd the context, the session compresses, and the state is lost. Each finding
  already names its site — use it.
- **Recovery after compression.** If the session's history was compressed mid-fix
  loop, recover by reading `fixes.md` (and `notes.md` for background) — never by
  re-reading implementation files to "recall" what was done. The files on disk are
  the record.
- **Per-finding budget.** At most 2 reads and 1 test run per finding. If a finding
  needs more, append two lines to `fixes.md` explaining why and what will be read,
  then continue. The whole repair phase stays within findings × 2 reads — countable,
  auditable by `/code-review`.
- **Closing.** When every line is checked, run the gate once (the same single
  two-run gate as completion — never per finding), confirm the six artifacts exist,
  and only then report the work as done.

Run typechecking regularly, and single test files regularly. At the end, run the guard's two invocations — main suite and trap suite — instead of one ad-hoc full-suite run.

Before reporting tests as passing, Read `skills/test-hallucination-guard/SKILL.md` and run its gate. The work is not finished until `.scratch/<feature>/main.<ext>`, `.scratch/<feature>/trap.<ext>`, and `.scratch/<feature>/runs.txt` exist, with both exit codes recorded in `runs.txt`; a missing artifact means incomplete, not green. `plan.md` and `notes.md` from the exploration phase are part of the record: `code-review` reads them as the audit trail for scope and the read budget. `fixes.md` is part of the record after a review round: every finding line checked off, or the work is not finished.

The one mechanical exemption: if the full diff touches no executable code — only
documentation, comments, or user-facing copy strings — the gate is skipped and the
cycle is marked `no-code-change`. This is decided by the diff's contents, never by judging the ticket "small"; a ticket with any code change, however trivial, runs the
full gate.

Once done, use /code-review to review the work.

Commit your work to the current branch.
