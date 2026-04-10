# SUMMARY-1.1.md — Phase 1 Spike & Baseline Capture

**Status:** complete

**Plan:** PLAN-1.1.md
**Phase:** 1 — Spike & Baseline Capture
**Branch:** main
**Date:** 2026-04-10

---

## Tasks Completed

### Task 1: .gitattributes + renormalize

- Created `.gitattributes` at repo root with Windows-CRLF rule set (`* text=auto eol=crlf`) and binary exclusions for `.dll`, `.exe`, `.pdb`, `.nupkg`, `.snk`, `.png`, `.ico`, `.jpg`, `.gif`.
- Staged with `git add .gitattributes`.
- Ran `git add --renormalize .` — produced **zero additional staged files**. The existing index already stores CRLF (Windows-authored repo), so no normalization delta existed.
- Committed atomically: `700a7c3 shipyard phase 1: add .gitattributes for line-ending discipline`. Files in commit: **1** (`.gitattributes` only).
- Second-pass idempotency check: CLEAN.

### Task 2: MSBuild baseline capture (read-only)

- Ran `/t:Restore` then `/t:Build` with `Configuration=Release`.
- Both invocations exited 0.
- Build summary: **0 Warning(s), 0 Error(s)**.
- No source files were modified; `git status --short` remained clean after build.
- No commit required (read-only task).

### Task 3: SPIKE-NOTES.md

- Created `.shipyard/phases/1/SPIKE-NOTES.md` containing all five required sections.
- Force-added past `.shipyard/.gitignore`: `git add -f .shipyard/phases/1/SPIKE-NOTES.md`.
- Committed: `b369dbb shipyard phase 1: capture spike notes and baseline warnings`.
- Verdict line: `GO — Phase 2 may begin`.

---

## Files Modified

| File | Action |
|---|---|
| `.gitattributes` | Created (new file) |
| `.shipyard/phases/1/SPIKE-NOTES.md` | Created (new file, force-added) |
| `.shipyard/phases/1/results/SUMMARY-1.1.md` | Created (this file, force-added) |

---

## Decisions Made

**Path translation for MSBuild (deviation from plan instructions):**
The plan specified passing the WSL path `/mnt/f/git/.../Examples.sln` to MSBuild.exe. On first attempt this produced `MSBUILD : error MSB1001: Unknown switch` because MSBuild.exe is a Windows binary that treats the `/mnt/...` string as a switch, not a path. Resolution: used `wslpath -w` to obtain the Windows-native path `F:\git\dotnetworkqueue.examples\Source\Examples\Examples.sln` and passed that instead. This is the correct and required invocation pattern for all future phases.

**Verbosity for warning capture:**
The plan's grep pattern for `N Warning(s)` did not match `/v:minimal` output because MSBuild minimal verbosity omits the summary block entirely. Reran build at `/v:normal` to obtain the summary line. Canonical baseline log: `/tmp/baseline/build-normal.log`.

**Renormalize produced no deltas:**
The pre-planning `git checkout -- .` (CONTEXT-1.md) had already restored the working tree to CRLF. Renormalize was a no-op. Task 1 locked in the permanent rule only.

---

## Issues Encountered

**MSBuild path:** WSL path (`/mnt/...`) must be translated to a Windows-native path (`F:\...`) before passing to MSBuild.exe. Every future phase that calls MSBuild from WSL must apply this translation.

**WSL renormalize quirk:** Did NOT occur. Working tree was already clean CRLF. A fresh clone in WSL without `git checkout -- .` may still surface the 160-file phantom flip; the `.gitattributes` + renormalize approach will handle it correctly as a one-shot delta commit.

---

## Verification Results

All 7 plan verification checks passed:

1. `.gitattributes` exists at repo root: PASS
2. Working tree clean (0 unexpected lines): PASS
3. Renormalize is no-op: PASS
4. SPIKE-NOTES.md tracked in git: PASS
5. SPIKE-NOTES.md has 5 sections + `GO` verdict: PASS
6. Last two commits match expected subjects: PASS
7. Baseline build log shows `0 Error(s)` and `0 Warning(s)`: PASS
