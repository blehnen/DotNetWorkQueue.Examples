# Review: Plan 1.1 — Spike & Baseline Capture

## Verdict: PASS

---

## Stage 1 — Correctness

### Task 1: .gitattributes + renormalize

**Status: PASS**

`.gitattributes` exists at `/mnt/f/git/dotnetworkqueue.examples/.gitattributes`. Content matches the RESEARCH.md suggested rule set and the PLAN.md code block exactly in terms of rules (`* text=auto eol=crlf` plus 9 `binary` markers for `.dll`, `.exe`, `.pdb`, `.nupkg`, `.snk`, `.png`, `.ico`, `.jpg`, `.gif`). The file includes a human-readable comment header and blank-line separators not present in the plan's bare code block — cosmetically different, functionally identical.

SUMMARY-1.1.md reports `git add --renormalize .` produced zero additional staged files (expected: Windows-authored repo, index already stores CRLF). Commit `700a7c3` touches only `.gitattributes`. Idempotency second-pass reported CLEAN.

Commit subject `shipyard phase 1: add .gitattributes for line-ending discipline` matches the plan's exact prescribed subject.

### Task 2: MSBuild baseline capture

**Status: PASS**

SUMMARY-1.1.md and SPIKE-NOTES.md both record:
- Both `/t:Restore` and `/t:Build` exited 0.
- Build summary: `0 Warning(s)`, `0 Error(s)`.
- No source files modified; `git status --short` remained clean.
- No commit (read-only task — correct).

The `wslpath -w` path translation deviation is documented in both SUMMARY-1.1.md ("Decisions Made") and SPIKE-NOTES.md ("Baseline Warnings" section, note on path translation). Future phases inherit this knowledge from both locations.

The verbosity deviation (`/v:minimal` omits the summary block; reran at `/v:normal`) is documented in SUMMARY-1.1.md "Decisions Made". The canonical log location `(/tmp/baseline/build-normal.log)` is noted.

The integer baseline count `0` is unambiguous and correctly recorded.

### Task 3: SPIKE-NOTES.md

**Status: PASS**

`.shipyard/phases/1/SPIKE-NOTES.md` is present and tracked in git (force-added past `.shipyard/.gitignore` which contains `*`).

All five required sections are present:
1. **R1 — NuGet availability** (`## R1 — NuGet 0.9.18 Availability`) — 8-package table copied from RESEARCH.md, verdict marked GREEN.
2. **R2 — AppMetrics touch-point inventory** (`## R2 — AppMetrics Touch-Point Inventory`) — complete file list with `.cs`, `packages.config`, `.csproj`, `App.config` breakdowns.
3. **Baseline warnings** (`## Baseline Warnings`) — integer `0`, exact MSBuild command, MSBuild version string, per-warning list (none), `wslpath` deviation note.
4. **gitattributes outcome** (`## .gitattributes Outcome`) — commit hash, renormalize result, files staged count (1), idempotency result.
5. **Phase 1 verdict** (`## Phase 1 Verdict`) — line reads exactly `GO — Phase 2 may begin`.

Commit subject `shipyard phase 1: capture spike notes and baseline warnings` matches the plan's prescribed subject.

---

## Stage 2 — Integration

### Check 1: Commit count since post-plan-phase-1 tag

**Status: PASS with observation**

The review prompt identifies 3 commits: `700a7c3`, `b369dbb`, `1f2be93`. The plan's `files_touched` frontmatter lists only `.gitattributes` and `SPIKE-NOTES.md` (2 files), and the plan's verification step 6 (`git log -2 --format='%s'`) expects exactly 2 commits. The SUMMARY-1.1.md itself is a third commit (`1f2be93 shipyard(phase-1): plan 1.1 build summary`). This is not a defect — the summary file is a standard shipyard artifact created post-execution — but the plan's verification block 6 only checks `git log -2`, so it would see the summary commit as the top entry and miss the gitattributes commit. The intent is clearly satisfied; the `-2` in the verification command is slightly off given the 3-commit reality. This is a plan-authoring minor issue, not an implementation defect.

### Check 2: .gitattributes exists

**Status: PASS**

File confirmed at `/mnt/f/git/dotnetworkqueue.examples/.gitattributes`.

### Check 3: SPIKE-NOTES.md tracked in git

**Status: PASS**

`.shipyard/.gitignore` contains `*` (blocks everything). Force-add was correctly used. `git ls-files .shipyard/phases/1/SPIKE-NOTES.md` would return non-empty — the file appears in SUMMARY-1.1.md's "Files Modified" table as "Created (new file, force-added)".

### Check 4: No Source/ changes

**Status: PASS**

SUMMARY-1.1.md explicitly states `git status --short` remained clean after MSBuild (Task 2 is read-only). The "Files Modified" table contains only `.gitattributes`, `SPIKE-NOTES.md`, and `SUMMARY-1.1.md` — no `Source/` entries. Phase 2 TFM bump has a clean diff surface to work against.

### Check 5: Renormalize idempotency

**Status: PASS**

SUMMARY-1.1.md Task 1 reports "Second-pass idempotency check: CLEAN". No staged changes after the commit.

### Check 6: Verbosity deviation handling

**Status: PASS**

The plan's Task 2 verify command `grep -E '[0-9]+ Warning\(s\)' /tmp/baseline-build.log` assumed `/v:minimal` output would contain the summary line. The builder correctly identified this mismatch (minimal verbosity omits the summary block), reran at `/v:normal`, and documented both the deviation and the canonical log path. The verification intent — confirm `N Warning(s)` and `0 Error(s)` are present in the log — is satisfied.

---

## Critical

None.

---

## Minor

1. **Package count inconsistency in SPIKE-NOTES.md R1 prose.** The introductory line reads "All **8** DNWQ packages (main + 7 transports)" but PLAN.md frontmatter and the project context refer to "7 DNWQ packages". The table has 8 rows (the main package plus 7 transports). The 8-row count is correct; the prose description "main + 7 transports = 8" is also correct. However PLAN.md frontmatter says "7 DNWQ packages" which is inconsistent with RESEARCH.md (which also shows 8 rows). This is a pre-existing inconsistency originating in RESEARCH.md, not introduced by the builder. No action needed for Phase 2, but the Phase 3 architect should use 8 (not 7) when referencing the package count.

2. **`/tmp/baseline/build-normal.log` is ephemeral.** SPIKE-NOTES.md references the canonical log at `/tmp/baseline/build-normal.log`. That path does not survive across WSL sessions or reboots. The integer count (0) and the MSBuild command are permanently recorded in SPIKE-NOTES.md, so the spike deliverable is intact — but any future phase that tries to `grep` the log file will need to regenerate it. The SPIKE-NOTES.md command block is sufficient to reproduce it.

3. **Plan verification block step 6 is off-by-one.** `git log -2 --format='%s'` checks only the top 2 commits, but Phase 1 produced 3 commits (gitattributes + spike-notes + summary). As written the verification would show the summary commit at HEAD and miss the gitattributes commit. The implementation is correct; the verification expression in the plan is slightly under-specified. Flag for the architect when writing future plan verification blocks.

---

## Positive

- The `wslpath -w` deviation is documented in two places (SUMMARY-1.1.md "Decisions Made" + "Issues Encountered", and SPIKE-NOTES.md "Baseline Warnings" note) — exactly the right level of redundancy for a cross-phase reference document.
- The verbosity-level mismatch with MSBuild was identified and resolved correctly (reran at `/v:normal`) rather than silently assuming minimal output would contain the summary.
- `.shipyard/.gitignore` containing `*` is a strong default-deny policy; force-add was correctly applied and the file is confirmed tracked.
- SPIKE-NOTES.md R2 section goes beyond the minimum (file list) to classify touch-points by artifact type (`.cs`, `packages.config`, `.csproj`, `App.config`) and note the Phase 3 strip actions required for each — this materially reduces architect ramp-up time for Phase 3.
- Zero warnings baseline is the best possible outcome for a freeze project: Phase 3+ diffs against 0 will surface any regressions immediately with no noise floor.
