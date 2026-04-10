---
phase: spike-and-baseline
plan: 1.1
wave: 1
dependencies: []
must_haves:
  - .gitattributes committed with line-ending discipline
  - Baseline Release warning count captured as integer + per-warning list
  - SPIKE-NOTES.md committed consolidating R1, R2, baseline, and Phase 1 verdict
files_touched:
  - .gitattributes
  - .shipyard/phases/1/SPIKE-NOTES.md
tdd: false
risk: medium
---

# Plan 1.1: Phase 1 Spike & Baseline Capture

## Context

This is the sole plan for Phase 1 ("Spike & Baseline Capture") of the `examples-freeze` project. Phase 1 locks in the inputs the rest of the freeze project diffs against: package availability, AppMetrics touch-points, a clean working tree with stable line endings, and a frozen baseline warning count.

Two of the four Phase 1 research questions are already resolved and recorded in `.shipyard/phases/1/RESEARCH.md`:

- **R1 — NuGet 0.9.18 availability for all 7 DNWQ packages:** GREEN. No builder task required; the RESEARCH.md table is the deliverable.
- **R2 — AppMetrics touch-point enumeration:** complete. The file list in RESEARCH.md is the deliverable.

What remains for the builder is the concrete, repo-touching work: install a permanent line-ending policy, capture the baseline Release warning count from MSBuild 18, and consolidate everything into `SPIKE-NOTES.md` as the frozen snapshot later phases diff against.

Late-binding decisions from `.shipyard/phases/1/CONTEXT-1.md`:
- The CRLF fix is permanent (`.gitattributes`), not a one-shot reset.
- `git checkout -- .` has already been run pre-planning, so the working tree is clean except for the untracked `CLAUDE.md` from an earlier `/init`.
- MSBuild path from RESEARCH.md — quoting is load-bearing because the path contains spaces:
  `"/mnt/c/Program Files/Microsoft Visual Studio/18/Community/MSBuild/Current/Bin/MSBuild.exe"`
- MSBuild 18 handles `packages.config` restore via `/t:Restore`; no separate `nuget.exe` is needed.

## Dependencies

None. This is the first plan of the first phase. Nothing in the project precedes it.

## Risk

**Medium.** Two specific failure modes to watch:

1. Task 1 `git add --renormalize .` may surface a large number of real CRLF/LF deltas. If so, they MUST be committed in the same atomic commit as `.gitattributes` — never split, because a tree where `.gitattributes` is committed but normalization hasn't been applied is an invalid intermediate state.
2. Task 2 assumes the pre-freeze solution builds clean today against DNWQ 0.8.0 + net472. If MSBuild fails, Phase 1's assumption is invalidated and the builder must **stop and escalate**, not attempt repairs — Phase 1 is a spike, not a fix-up.

## Tasks

<task id="1" files=".gitattributes" tdd="false">
  <action>
Create `.gitattributes` at the repo root with the starter rule set from RESEARCH.md:

```
* text=auto eol=crlf
*.dll   binary
*.exe   binary
*.pdb   binary
*.nupkg binary
*.snk   binary
*.png   binary
*.ico   binary
*.jpg   binary
*.gif   binary
```

Then stage and renormalize:

```
git add .gitattributes
git add --renormalize .
git status --short
```

If `git add --renormalize .` stages additional files beyond `.gitattributes`, those are real normalization deltas and MUST be included in the same commit — do not split. Commit atomically:

```
git commit -m "shipyard phase 1: add .gitattributes for line-ending discipline"
```
  </action>
  <verify>
test -f .gitattributes && \
git log -1 --format='%s' | grep -q '^shipyard phase 1: add .gitattributes' && \
git add --renormalize . && git diff --cached --quiet && \
git status --short | grep -v '^?? CLAUDE.md$' | grep -v '^?? .shipyard/' | wc -l
  </verify>
  <done>
`.gitattributes` exists at repo root with the documented rule set. `git log -1 --name-only` shows the commit touches `.gitattributes` (plus any renormalized files, if deltas existed). Rerunning `git add --renormalize .` afterward produces zero staged changes. `git status --short`, filtered for the expected untracked `CLAUDE.md` and any `.shipyard/` scratch, returns 0 lines.
  </done>
</task>

<task id="2" files="" tdd="false">
  <action>
Read-only build probe. Do NOT modify any source file. Invoke MSBuild 18 with the quoted Windows path, Restore first then Build, both Release:

```
MSBUILD='/mnt/c/Program Files/Microsoft Visual Studio/18/Community/MSBuild/Current/Bin/MSBuild.exe'
"$MSBUILD" Source/Examples/Examples.sln /t:Restore /p:Configuration=Release /v:minimal 2>&1 | tee /tmp/baseline-restore.log
"$MSBUILD" Source/Examples/Examples.sln /t:Build   /p:Configuration=Release /v:minimal 2>&1 | tee /tmp/baseline-build.log
```

From `/tmp/baseline-build.log`, extract:
1. The integer warning count from the final MSBuild summary line matching `XXX Warning(s)`.
2. The full list of unique warning codes plus a short message for each (e.g. `CS0618 'X' is obsolete`) for later Phase 3 / Phase 7 diffing.

Hold these two pieces of data in scratch memory / a tmp file for Task 3 to consume. Do not commit the log files themselves.

If either MSBuild invocation exits non-zero, STOP the phase. Do not attempt fixes. Record the failure mode (exit code, last 40 log lines) and escalate to the user — Phase 1 explicitly assumes the pre-freeze solution builds clean today, and a failing baseline is a surprise that needs user input before Phase 2 can be scoped.
  </action>
  <verify>
test -s /tmp/baseline-build.log && \
grep -E '[0-9]+ Warning\(s\)' /tmp/baseline-build.log && \
grep -E '0 Error\(s\)' /tmp/baseline-build.log && \
git status --short | grep -v '^?? CLAUDE.md$' | grep -v '^?? .shipyard/' | wc -l
  </verify>
  <done>
Both `/t:Restore` and `/t:Build` exited 0. `/tmp/baseline-build.log` contains a `0 Error(s)` line and an `N Warning(s)` line where N is a non-negative integer. The builder has recorded N and the per-warning list for Task 3. `git status --short` (filtered for expected untracked) shows zero changes — no source file was touched. If the build failed, the builder has halted and escalated rather than continuing to Task 3.
  </done>
</task>

<task id="3" files=".shipyard/phases/1/SPIKE-NOTES.md" tdd="false">
  <action>
Create `.shipyard/phases/1/SPIKE-NOTES.md` as the frozen Phase 1 snapshot that later phases diff against. Required sections:

1. **R1 — NuGet availability.** Copy the 7-package availability table from RESEARCH.md verbatim and mark the verdict GREEN.
2. **R2 — AppMetrics touch-point inventory.** Copy the file list from RESEARCH.md verbatim.
3. **Baseline warnings.** Record the integer warning count from Task 2 and the full per-warning list (code + short message). Note the exact MSBuild command used so future phases can reproduce the measurement.
4. **gitattributes outcome.** Record whether `git add --renormalize .` produced any deltas beyond `.gitattributes` itself, and how many files were touched by the atomic commit from Task 1.
5. **Phase 1 verdict.** A single line reading `GO — Phase 2 may begin` if and only if R1 is GREEN, R2 is captured, Task 1 committed cleanly, and Task 2's build succeeded. Otherwise `STOP` or `ESCALATE` with a one-line reason.

`.shipyard/.gitignore` blocks everything under `.shipyard/` by default, so force-add:

```
git add -f .shipyard/phases/1/SPIKE-NOTES.md
git commit -m "shipyard phase 1: capture spike notes and baseline warnings"
```
  </action>
  <verify>
test -f .shipyard/phases/1/SPIKE-NOTES.md && \
git ls-files --error-unmatch .shipyard/phases/1/SPIKE-NOTES.md && \
grep -q '^## R1' .shipyard/phases/1/SPIKE-NOTES.md && \
grep -q '^## R2' .shipyard/phases/1/SPIKE-NOTES.md && \
grep -qE 'Baseline|baseline' .shipyard/phases/1/SPIKE-NOTES.md && \
grep -qE 'GO — Phase 2 may begin|STOP|ESCALATE' .shipyard/phases/1/SPIKE-NOTES.md && \
git log -1 --format='%s' | grep -q '^shipyard phase 1: capture spike notes'
  </verify>
  <done>
`.shipyard/phases/1/SPIKE-NOTES.md` is tracked in git (force-added past `.shipyard/.gitignore`) and contains all five required sections: R1 verdict, R2 file list, baseline warning integer + list, gitattributes outcome, and a Phase 1 verdict line. The last commit on `main` has subject `shipyard phase 1: capture spike notes and baseline warnings`. The verdict line reads `GO — Phase 2 may begin` if and only if all upstream checks passed.
  </done>
</task>

## Verification

Run all of the following from the repo root after the plan is executed. Every command should succeed and produce the expected output.

```bash
# 1. .gitattributes exists at repo root
test -f .gitattributes && echo "gitattributes OK"

# 2. Working tree clean modulo expected untracked CLAUDE.md
git status --short | grep -v '^?? CLAUDE.md$' | wc -l   # expect 0

# 3. Renormalize is a no-op (proves gitattributes already applied)
git add --renormalize . && git diff --cached --quiet && echo "renormalize clean"

# 4. SPIKE-NOTES.md is tracked
git ls-files .shipyard/phases/1/SPIKE-NOTES.md   # expect non-empty

# 5. SPIKE-NOTES.md has the required sections and a verdict line
grep -c '^## ' .shipyard/phases/1/SPIKE-NOTES.md                       # expect >= 5
grep -E 'GO — Phase 2 may begin|STOP|ESCALATE' .shipyard/phases/1/SPIKE-NOTES.md

# 6. Last two commits on main are the two Phase 1 spike commits
git log -2 --format='%s'
# expect:
#   shipyard phase 1: capture spike notes and baseline warnings
#   shipyard phase 1: add .gitattributes for line-ending discipline

# 7. Baseline build log exists and shows zero errors
grep -E '0 Error\(s\)'     /tmp/baseline-build.log
grep -E '[0-9]+ Warning\(s\)' /tmp/baseline-build.log
```
