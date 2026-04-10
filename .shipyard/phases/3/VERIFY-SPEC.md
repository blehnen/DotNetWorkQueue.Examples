# Phase 3 Plan — Spec Compliance Review

**Date:** 2026-04-10  
**Reviewer:** Verification Agent (spec-compliance mode)

## Verdict

**PASS**

Plan 3.1 comprehensively covers all requirements (R6–R11 + NFR baseline discipline) with robust verification gates, clear task decomposition (3 tasks), and explicit atomic-commit enforcement. The plan is ready for builder execution.

---

## Coverage Analysis

| Req | Delivery | Evidence |
|-----|----------|----------|
| **R6** | Pin DNWQ at 0.9.18 | Task 1 (packages.config transform step 3), Task 3 (verify presence in packages/ folder). Explicit grep: `grep -rl 'DotNetWorkQueue\.0\.9\.18'` expected > 0 |
| **R7** | Remove Polly v7 stragglers | Task 1 (csproj transform step 5, packages.config step 4), Task 3 (Polly v7 API check: `grep -rE '^using Polly(\.Caching\.Memory\|\.Contrib)'`). RESEARCH.md identified 37 entries across 16 files — all removal paths explicit |
| **R8** | Remove AppMetrics pkg refs | Task 1 (csproj steps 1–2, packages.config steps 1–2, App.config step 1). Verify: `grep -rl 'App\.Metrics' Source/Examples/ --include='*.csproj'` + `--include='packages.config'` expect empty. 102 csproj blocks + 102 packages.config entries targeted |
| **R9** | Strip AppMetrics code | Task 2 (8 .cs files listed explicitly). Verify: `grep -rE 'App\.Metrics\|IMetricsRoot\|DotNetWorkQueue\.AppMetrics' Source/Examples/ --include='*.cs'` expect empty. RESEARCH.md scoped ~53 lines total — load-bearing file (SharedCommands.cs ~25 lines) identified |
| **R11** | Absorb non-metrics breakage | Task 3, Step 5 (ABSORPTION DECISION GATE). Explicit budget: ≤10 files, ≤30 min, mechanical fixes only. Structural examples listed with rollback criteria. `ERR_FILES > 10` forces automatic rollback |

**Verdict:** All five core requirements explicitly mapped to tasks with measurable verification steps.

---

## Task Count

**Specification:** ≤3 tasks per Shipyard rule.  
**Plan:** 3 tasks (IDs: 1, 2, 3).  
**Status:** ✓ PASS

---

## Atomic Commit Invariant

**Specification:** All XML + .cs edits + package cache regeneration in ONE commit. Tasks 1–2 stage only; Task 3 commits after restore+build success.

**Plan Evidence:**
- **Task 1** (line 96–100): `git add` XML files after transform, **no commit**. String: "All XML changes are staged (visible in `git diff --cached --stat`) but **no commit has been made**."
- **Task 2** (line 188–198): `git add` .cs files, **no commit**. String: "All 8 `.cs` files are staged but **not yet committed** (Task 3 handles the atomic commit)."
- **Task 3, Step 7** (line 357–362): Explicit commit after clean build. String: `git commit -m "shipyard(phase-3): pin DotNetWorkQueue at 0.9.18 and strip AppMetrics"`

**Verdict:** ✓ PASS — Atomic-commit rule is enforced end-to-end.

---

## Pre-Commit Verification Gate

**Specification:** Task 3 must include `/t:Restore` exit check, `/t:Build` exit check, warning count ≤ Phase 1 baseline (0), explicit rollback on failure.

**Plan Evidence:**

1. **Restore exit check** (Task 3, Step 3, lines 267–278):
   ```bash
   RESTORE_EXIT=$?
   echo "restore exit: $RESTORE_EXIT"
   if [ "$RESTORE_EXIT" -ne 0 ]; then
     echo "RESTORE FAILED — hard rollback"
     git reset HEAD Source/Examples/
     git checkout -- Source/Examples/
     exit 1
   fi
   ```
   ✓ Explicit exit code capture, failure message, hard rollback.

2. **Build exit check** (Task 3, Step 4, lines 281–286):
   ```bash
   BUILD_EXIT=$?
   echo "build exit: $BUILD_EXIT"
   ```
   ✓ Captured for downstream ABSORPTION DECISION GATE.

3. **Baseline warning discipline** (Task 3, Step 6, lines 342–355):
   ```bash
   WARN_COUNT=$(echo "$WARN_LINE" | grep -oE '[0-9]+' | head -1)
   ERR_COUNT=$(echo "$ERR_LINE" | grep -oE '[0-9]+' | head -1)
   if [ "${ERR_COUNT:-1}" -ne 0 ] || [ "${WARN_COUNT:-1}" -gt 0 ]; then
     echo "BASELINE VIOLATION — rolling back"
     git reset HEAD Source/Examples/
     git checkout -- Source/Examples/
     exit 1
   fi
   ```
   ✓ Explicit parse of warning/error counts. Rollback if ERR_COUNT ≠ 0 OR WARN_COUNT > 0. Phase 1 baseline = 0 (SUMMARY-2.1.md: "0 warnings / 0 errors post-build").

4. **Rollback procedure** (Task 3, Step 2 + Step 3 + Step 6):
   - Hard reset: `git reset HEAD Source/Examples/` + `git checkout -- Source/Examples/`
   - Cache purge: `rm -rf Source/Examples/packages/` (CONTEXT-3.md decision)
   - Exit code 1 to stop the plan

**Verdict:** ✓ PASS — All four verification elements present with explicit rollback paths.

---

## Absorption Budget Clarity

**Specification:** Plan must quote CONTEXT-3.md budget (≤10 files, ≤30 min, mechanical-only) with concrete examples of "mechanical" vs "structural" so the builder can decide in the field.

**Plan Evidence:**

- **Budget statement** (Task 3, Step 5, lines 300–303):
  ```
  Per CONTEXT-3.md, the absorption budget is:
  - ≤10 example files touched
  - ≤30 minutes of mechanical edits
  - Mechanical fixes only
  ```

- **Mechanical examples** (lines 305–312):
  - Method rename (e.g. `CreateAsync` → `CreateQueueAsync`)
  - Parameter reorder
  - Namespace move
  - Property rename
  - Return type changed but caller ignores return value
  - Type alias / enum value rename

- **Structural examples / hard rollback** (lines 314–321):
  - Required constructor parameter added
  - Interface contract changed
  - Whole subsystem removed/replaced
  - DI registration changes affecting `QueueContainer<TInit>`
  - Error requiring design judgment
  - Cascades into >10 files
  - **`ERR_FILES > 10` — automatic rollback**

- **Absorption procedure** (lines 332–340):
  - Examine `/tmp/phase3/build-errors.txt`
  - Confirm mechanical per the list
  - Confirm `ERR_FILES <= 10`
  - Start 30-minute timer
  - Max 3 rebuild iterations before timer forces rollback
  - Record every fix in SUMMARY-3.1.md with file, error code, one-line description

**Verdict:** ✓ PASS — Budget is quoted, mechanical/structural taxonomy is explicit, rollback criteria include file count threshold and timer guardrails. Builder can use this to decide in field whether to absorb or escalate.

---

## Testable Acceptance Criteria

**Specification:** Each task's acceptance criteria (the `<done>` blocks) must be runnable shell checks, not aspirational prose.

**Task 1 `<done>` block** (lines 135–137):
- 7× `grep` checks with "CLEAR" sentinels ✓
- 2× `wc -l` count checks (expect > 0) ✓
- Transform log summary print ✓
- Staged changes visible via `git diff --cached --stat` ✓

**Task 2 `<done>` block** (lines 229–231):
- 3× `grep` checks with "CLEAR" sentinels ✓
- 1× `wc -l` file count (expect 8) ✓
- Staged files check via `git diff --cached --name-only | grep .cs$ | wc -l` ✓

**Task 3 `<done>` block** (lines 387–389):
- `ls Source/Examples/packages/` checks for 0.9.18 presence and 0.8.0 absence ✓
- No AppMetrics package folders ✓
- Build log tail + baseline summary ✓
- Git log check for exact commit subject ✓
- Working tree clean check ✓

**Verify block** (lines 394–415):
- 5× `grep` checks with "CLEAR" sentinels ✓
- Count check for 0.9.18 references ✓
- Rebuild determinism check ✓

**Verdict:** ✓ PASS — All acceptance criteria are concrete shell checks with measurable outputs, not prose assertions.

---

## Scope Discipline

**Specification:** Plan must NOT touch:
- `SharedAssemblyInfo.cs` (Phase 4)
- HintPath cleanup for non-DNWQ packages (only DNWQ HintPaths bump)
- README (Phase 6)
- CI workflow (Phase 5)
- LiteDb namespace typo (Phase 4)

**Plan Evidence:**

1. **SharedAssemblyInfo.cs** — Not listed in `files_touched` (lines 12–22). Phase 4 scope.
2. **Non-DNWQ HintPath cleanup** — Task 1 step 4 (line 64) explicitly scopes HintPath to DNWQ only: "AND replace the lib subfolder `\lib\netstandard2.0\` with `\lib\net48\` [for DNWQ packages]". Transport variants (`DotNetWorkQueue.Transport.*`) also scoped.
3. **README** — Not in `files_touched`. Phase 6 scope.
4. **CI workflow** — Not in `files_touched`. Phase 5 scope.
5. **LiteDb namespace typo** — Not mentioned. Phase 4 scope (PLAN states "LiteDb namespace typo (Phase 4)").

**Verdict:** ✓ PASS — Plan respects phase boundaries and does not pre-empt downstream work.

---

## Python Transform Correctness Assumptions

**Specification:** Plan must specify:
- `ET.register_namespace('', '...')` to preserve default namespace
- Match pattern for `<Reference>` deletion based on `Include` attribute prefix
- Version bump scoped to DNWQ only
- HintPath subfolder migration only for DNWQ HintPaths

**Plan Evidence:**

1. **Namespace registration** (Task 1, lines 57–58):
   ```
   Use `ET.register_namespace('', 'http://schemas.microsoft.com/developer/msbuild/2003')`
   BEFORE parsing any csproj.
   ```
   ✓ Explicit and correct.

2. **`<Reference>` deletion by prefix** (Task 1, csproj step 1, lines 60–61):
   ```
   Delete every `<Reference>` element whose `Include` attribute starts with
   `App.Metrics.` (match prefix before first comma)
   ```
   ✓ Prefix-based, not regex. Correctly targets Include attribute.

3. **Version bump scoped to DNWQ** (Task 1, csproj step 3, lines 63–64):
   ```
   For remaining `<Reference>` elements whose `Include` starts with
   `DotNetWorkQueue` or `DotNetWorkQueue.Transport.`, replace the substring
   `Version=0.8.0.0` with `Version=0.9.18.0` in the `Include` attribute value.
   ```
   ✓ Scoped. Will not touch unrelated packages' versions.

4. **HintPath subfolder migration scoped to DNWQ** (Task 1, csproj step 4, lines 64–65):
   ```
   For each such DNWQ `<Reference>`, locate its child `<HintPath>` and replace
   the package-folder segment `DotNetWorkQueue.0.8.0` with `DotNetWorkQueue.0.9.18`
   (and the same pattern for every transport variant — match any segment matching
   `DotNetWorkQueue(\.Transport\.[^\\]+)?\.0\.8\.0`), AND replace the lib subfolder
   `\lib\netstandard2.0\` with `\lib\net48\`.
   ```
   ✓ HintPath mutations only apply to DNWQ references (guarded by "For each such DNWQ"). Regex pattern for transport variants is precise.

**Verdict:** ✓ PASS — All four assumptions are explicitly stated and correct.

---

## Overall Assessment

| Criterion | Status | Notes |
|-----------|--------|-------|
| Coverage (R6–R11) | PASS | All 5 requirements fully addressed |
| Task count (≤3) | PASS | Exactly 3 tasks |
| Atomic commit invariant | PASS | Tasks 1–2 stage only; Task 3 commits after build success |
| Pre-commit verification gate | PASS | Restore exit, Build exit, warning count ≤0, explicit rollback all present |
| Absorption budget clarity | PASS | Budget quoted with mechanical/structural taxonomy and file-count guardrail |
| Testable acceptance criteria | PASS | All `<done>` blocks and verify block contain runnable shell checks |
| Scope discipline | PASS | Respects phase boundaries; no Phase 4/5/6 scope creep |
| Python transform correctness | PASS | Namespace registration, prefix matching, version scoping, HintPath scoping all explicit |

---

## Recommendations

1. **No remediation required.** Plan is specification-compliant and ready for builder execution.
2. **Builder attention points:**
   - RESEARCH.md notes restore will be slower than Phase 2 (~30+ new packages). Do not impose wall-clock timeout.
   - Polly v7 fast-check (Task 3, Step 2) will trigger a warning if example code directly imports Polly v7 — pay close attention to the warning and the subsequent build outcome.
   - If absorption is needed, the 30-minute timer and 3-rebuild limit are hard stops; do not skip the `SUMMARY-3.1.md` "Decisions Made" recording.

---

**Status:** Ready for execution.
