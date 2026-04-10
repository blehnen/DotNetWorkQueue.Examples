# Phase 1 Plan — Feasibility Critique

**Date:** 2026-04-10  
**Plan:** PLAN-1.1.md  
**Reviewer:** Verification Agent

## Verdict
**READY** — All feasibility checks pass. No blocking issues. Proceed to build with awareness of documented cautions.

---

## Per-task findings

### Task 1: Create `.gitattributes` and renormalize working tree

**File path existence:**  
✓ PASS — `.gitattributes` does not yet exist at repo root (confirmed via `test -f .gitattributes` → not found). This is correct; Task 1 creates it.

**Command validity:**  
✓ PASS — `git add --renormalize` is valid and documented. Verified: `git version 2.43.0` includes `--renormalize` in help output, described as "renormalize EOL of tracked files (implies -u)". The command is supported.

**Quoting and escaping:**  
✓ PASS — No special characters in the planned commit message that would require escaping. Task correctly staggers: `git add .gitattributes`, then `git add --renormalize .` (separate operations).

**Hidden risk — CAUTION:**  
⚠ **WSL git + Windows-authored repo normalization deltas.** RESEARCH.md flags this at line 129: "may trigger real normalization changes via `git add --renormalize .`. If it does, those normalized changes are a one-shot commit that must land on `main` alongside `.gitattributes`."

Task 1's verification step (line 85) checks: `git status --short | grep -v '^?? CLAUDE.md$' | grep -v '^?? .shipyard/' | wc -l` → expects 0 after `git add --renormalize .` runs. This is correct: if renormalize produces staged deltas (e.g., 160+ files with CRLF→LF changes), they MUST be included in the commit at line 78. The plan correctly treats this as atomic ("do not split") and the verification confirms no lingering untracked/unstaged files post-commit.

**Atomic commit handling:**  
✓ PASS — Plan explicitly warns at line 45: "Medium risk. Task 1 `git add --renormalize .` may surface a large number of real CRLF/LF deltas. If so, they MUST be committed in the same atomic commit as `.gitattributes`." Task 1 correctly calls `git commit` once (line 78), and lines 45–46 state "never split, because a tree where `.gitattributes` is committed but normalization hasn't been applied is an invalid intermediate state." This is load-bearing discipline and the plan enforces it.

---

### Task 2: Build probe with MSBuild 18 — Restore + Build Release

**File path existence:**  
✓ PASS — `Source/Examples/Examples.sln` exists (confirmed via `test -f Source/Examples/Examples.sln` → found). This is the correct build target.

**MSBuild executable path:**  
✓ PASS — `/mnt/c/Program Files/Microsoft Visual Studio/18/Community/MSBuild/Current/Bin/MSBuild.exe` exists (confirmed via `test -f '/mnt/c/...'` → found). Quoting is correct in the plan (line 97).

**Command structure:**  
✓ PASS — Two-step invocation: `/t:Restore` then `/t:Build`, both with `/p:Configuration=Release /v:minimal`. This is correct. RESEARCH.md (line 106) confirms: "MSBuild 18 handles `packages.config` restore via `/t:Restore`" (built-in since MSBuild 17.3). No separate `nuget.exe` needed.

**Logging:**  
✓ PASS — Both MSBuild invocations pipe to `/tmp/baseline-*.log` files via `tee`, allowing Task 3 to parse them. Lines 102–106 extract: (1) integer warning count, (2) full list of unique warning codes.

**Failure handling:**  
✓ PASS — Lines 108–109 explicitly state: "If either MSBuild invocation exits non-zero, STOP the phase. Do not attempt fixes. Record the failure mode (exit code, last 40 log lines) and escalate to the user." This is correct escalation discipline for a spike phase.

**Verification block:**  
✓ PASS — Task 2's verify step (lines 111–115) checks: `/tmp/baseline-build.log` exists, contains `0 Error(s)` and `[0-9]+ Warning(s)` lines, and `git status --short` shows zero changes. This correctly confirms the build succeeded and no source files were touched.

**Hidden risk:**  
⚠ **Read-only assumption.** Task 2 is declared "do NOT modify any source file" (line 93). The verification correctly confirms this by checking `git status --short`. However, if the pre-freeze solution does NOT build cleanly against DNWQ 0.8.0 + net472, the builder must halt per lines 108–109. Plan is sound; execution risk is in the repo state, not the plan itself.

---

### Task 3: Create `.shipyard/phases/1/SPIKE-NOTES.md`

**File path existence:**  
✓ PASS — `.shipyard/phases/1/` directory exists (confirmed via `test -d .shipyard/phases/1` → found). Task 3 writes `SPIKE-NOTES.md` into this directory.

**`.shipyard/.gitignore` handling:**  
✓ PASS — `.shipyard/.gitignore` contains a single line: `*` (confirmed via Read). This ignores everything under `.shipyard/`. Task 3 correctly calls `git add -f` (line 134) to force-add `SPIKE-NOTES.md` past this blanket ignore. Without `-f`, the file would silently fail to stage and the commit would be empty. Plan explicitly calls this out at line 131: "`.shipyard/.gitignore` blocks everything under `.shipyard/` by default, so force-add:" followed by the `-f` flag. This is correct.

**Content requirements:**  
✓ PASS — Task 3 specifies five required sections (lines 123–129):
1. R1 — NuGet availability table (from RESEARCH.md, verbatim, mark GREEN)
2. R2 — AppMetrics touch-point file list (from RESEARCH.md, verbatim)
3. Baseline warnings — integer count + full per-warning list + exact MSBuild command for reproducibility
4. gitattributes outcome — whether `git add --renormalize .` produced deltas, file count
5. Phase 1 verdict — single-line GO/STOP/ESCALATE statement

Plan is explicit and unambiguous. Verification block (lines 139–146) confirms all five sections are present via grep for section headers and verdict line.

**Verdict line discipline:**  
✓ PASS — Lines 129 and 144 state the verdict reads "GO — Phase 2 may begin" if and only if R1 GREEN, R2 captured, Task 1 clean, Task 2 succeeded; otherwise "STOP" or "ESCALATE". This is correct phase-gating logic.

**Commit messaging:**  
✓ PASS — Task 3 commit (line 135) reads "shipyard phase 1: capture spike notes and baseline warnings". Verification (line 145) confirms `git log -1 --format='%s' | grep -q '^shipyard phase 1: capture spike notes'`. Message is clear and audit-traceable.

---

## Cross-task dependency audit

**Sequential ordering:**  
✓ PASS — Three tasks are strictly sequential:
- **Task 1 → Task 2:** Task 1 commits `.gitattributes` and renormalizes the working tree, producing a clean tree. Task 2 then builds against this clean baseline. Dependency is sound: Task 2's verification confirms zero source file changes, implying Task 1 completed.
- **Task 2 → Task 3:** Task 2 writes `/tmp/baseline-build.log`. Task 3 reads this file (line 127: "Record the integer warning count from Task 2 and the full per-warning list") and consumes it for `SPIKE-NOTES.md`. Dependency is explicit and load-bearing.
- **No parallel opportunities:** All three tasks are sequential with tight dependencies. No reordering is possible without breaking correctness.

**No external plan dependencies:**  
✓ PASS — Phase 1 has only one plan (PLAN-1.1.md). No forward references to other plans. The plan is self-contained.

**Shared files check:**  
✓ PASS — Files touched:
- `.gitattributes` (Task 1 creates and commits)
- `.shipyard/phases/1/SPIKE-NOTES.md` (Task 3 creates and commits)
- `/tmp/baseline-restore.log`, `/tmp/baseline-build.log` (Task 2 writes; Task 3 reads; not committed)

No conflicts. No task touches the same file in incompatible ways.

---

## Overall notes

**Caution items (non-blocking):**

1. **WSL git normalization deltas (CAUTION — documented in plan):** The plan correctly flags this at lines 45–46 and RESEARCH.md line 129. If `git add --renormalize .` stages 100+ files, the builder must include them in Task 1's atomic commit. The plan handles this correctly. A builder reading the plan will see this warning and proceed with appropriate care.

2. **Pre-freeze solution build success assumption (CAUTION — documented in plan):** Task 2 assumes the repo builds clean today. If it doesn't, the plan explicitly halts and escalates (lines 108–109). This is correct spike discipline. No remediation needed in the plan.

3. **Temp files in `/tmp` (minor):** Task 2 writes logs to `/tmp/baseline-*.log`. These are temporary and not committed. Task 3 reads them during execution. Lifespan is within a single plan run. No risk.

**Plan quality:**

- Verification blocks are concrete and runnable. Every command is a shell one-liner that can be executed post-build to validate success.
- Failure modes are documented (lines 45–46, 108–109, 131).
- Atomic commit discipline is enforced (Task 1).
- The plan is defensively written: it flags cautions and escalation points.
- No ambiguities in file paths, command syntax, or success criteria.

**Recommendation:**

Plan is production-ready. Proceed to build phase. Builder should be aware of the two cautions (normalization deltas, build success assumption) but no plan revision is needed.

---

## Summary

All file paths exist or are correctly specified as to-be-created. Commands are valid and correctly quoted. Dependencies are sequential and load-bearing. Cautions are documented in the plan itself. No blocking issues found.

**READY to proceed.**
