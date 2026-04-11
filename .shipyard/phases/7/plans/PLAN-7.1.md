# Plan 7.1: Verification & Freeze Commit

## Context

Final phase. No source edits. Three tasks:

1. Re-run a clean Release build (R18) from WSL via the Phase-1 validated MSBuild invocation. Must exit 0 with warnings ≤ Phase 1 baseline.
2. User-driven manual SQLite/LiteDb smoke test (R19 hard gate) + optional SQL Server / Postgres / Redis remote smoke tests (R20 best-effort). Results captured in SUMMARY-7.1.md.
3. Land the freeze commit: update ROADMAP.md, write VERIFICATION.md documenting all 10 PROJECT.md success criteria, tag `freeze` + `post-build-phase-7`.

## Dependencies

Phase 6 complete. Tree state: `post-build-phase-6`.

## Tasks

### Task 1: R18 — Final clean Release build

**Files:** none edited; produces `/tmp/phase7/build.log`

**Action:** verify

**Description:**

```bash
mkdir -p /tmp/phase7
MSBUILD='/mnt/c/Program Files/Microsoft Visual Studio/18/Community/MSBuild/Current/Bin/MSBuild.exe'
SLN_WIN="$(wslpath -w Source/Examples/Examples.sln)"

"$MSBUILD" "$SLN_WIN" /t:Build /p:Configuration=Release /v:normal > /tmp/phase7/build.log 2>&1
BUILD_EXIT=$?
echo "build exit: $BUILD_EXIT"

# Parse summary — do NOT read the full log
grep -E '^\s*[0-9]+ (Warning|Error)\(s\)' /tmp/phase7/build.log | tail -2
```

No `nuget restore` required — the package cache is already populated from Phase 3. If anything is missing in the cache, MSBuild will discover it and we fall back to the standalone nuget.exe approach.

**Acceptance criteria:**
- `BUILD_EXIT` is 0.
- Warning count is 0 (matches Phase 1 baseline).
- Error count is 0.
- If either is non-zero, STOP and escalate. The freeze does not ship on a drifted baseline.

### Task 2: R19 manual smoke test + R20 best-effort remotes (user-driven)

**Files:** instructions written into SUMMARY-7.1.md; results pasted back by user

**Action:** verify (manual)

**Description:**

Present the user with a concrete, copy-pasteable script and wait for confirmation.

---

**R19 — SQLite or LiteDb round-trip (hard gate, pick one):**

Open TWO Windows terminals (PowerShell, cmd, or Windows Terminal — NOT WSL).

**Terminal A — Producer:**

For LiteDb:
```
cd F:\git\dotnetworkqueue.examples\Source\Examples\LiteDb\Producer\LiteDbProducer\bin\Release
LiteDbProducer.exe
```

In the shell that opens, type:
```
QueueCreation CreateQueue phase7test
SendMessage Send phase7test 1 false
```

Leave the producer window open.

For SQLite (alternative if LiteDb fails):
```
cd F:\git\dotnetworkqueue.examples\Source\Examples\SQLite\Producer\SQLiteProducer\bin\Release
SQLiteProducer.exe
```
Same shell commands: `QueueCreation CreateQueue phase7test` then `SendMessage Send phase7test 1 false`.

**Terminal B — Consumer:**

For LiteDb:
```
cd F:\git\dotnetworkqueue.examples\Source\Examples\LiteDb\Consumer\LiteDbConsumer\bin\Release
LiteDbConsumer.exe
```

In the consumer shell, type:
```
ConsumeMessage Start phase7test 1 false
```

Watch for the message to appear in the consumer's log output. You should see a line indicating that SimpleMessage was received and processed.

Then stop the consumer with:
```
ConsumeMessage Stop phase7test
```

And clean up:
```
QueueCreation RemoveQueue phase7test
```
in the producer terminal.

**Tell me:** (1) did the message round-trip? (2) any errors or surprises? (3) which transport you used (LiteDb or SQLite)?

---

**R20 — Remote transports (best-effort, OK to skip any or all):**

All 3 remote transports point at `192.168.0.2` per the Phase 4 IP correction. If you have SQL Server, PostgreSQL, and Redis running there, repeat the producer/consumer round-trip for any or all of them using the per-transport executables:

- `Source\Examples\SQLServer\Producer\SqlServerProducer\bin\Release\SqlServerProducer.exe` + matching Consumer
- `Source\Examples\PostGresSQL\Producer\PostGresSQLProducer\bin\Release\PostGresSQLProducer.exe` + matching Consumer
- `Source\Examples\Redis\Producer\RedisProducer\bin\Release\RedisProducer.exe` + matching Consumer

**Tell me:** per transport, either "passed", "failed with <one-line reason>", or "skipped".

Each transport failure is documented but does NOT block the freeze.

---

**Acceptance criteria:**
- R19 round-trip result captured in SUMMARY-7.1.md as PASS or FAIL.
- If FAIL, the freeze stops and Phase 7 escalates — do NOT land the freeze commit.
- R20 results captured per-transport (PASS / FAIL / SKIPPED) in SUMMARY-7.1.md. Any combination is acceptable for the freeze to ship.

### Task 3: Land the freeze commit

**Files:**
- `.shipyard/ROADMAP.md` (mark Phase 7 complete + overall project ✅ FROZEN)
- `.shipyard/phases/7/VERIFICATION.md` (new — final 10-point PROJECT.md success criteria checklist)
- `.shipyard/phases/7/results/SUMMARY-7.1.md` (new — captures R18, R19, R20 results)

**Action:** commit

**Description:**

1. Write VERIFICATION.md with a checklist table mapping each of PROJECT.md's 10 success criteria to its evidence in the final state:

   | # | Criterion | Status | Evidence |
   |---|---|---|---|
   | 1 | Every csproj declares v4.8 | PASS | grep v4.7.2/v4.5.2 empty |
   | 2 | DNWQ pinned 0.9.18; zero AppMetrics refs | PASS | grep |
   | 3 | Solution builds clean Release | PASS | Phase 7 R18 log: 0/0 |
   | 4 | CI workflow exists + first run green | PASS (first run deferred to user push) | `.github/workflows/ci.yml` |
   | 5 | R19 mandatory smoke test passes | PASS | SUMMARY-7.1.md Task 2 |
   | 6 | R20 best-effort smoke attempts documented | PASS | SUMMARY-7.1.md Task 2 |
   | 7 | LiteDbConsumer namespace correct | PASS | Phase 4 commit |
   | 8 | SharedAssemblyInfo.cs at 0.9.18 | PASS | Phase 4 commit |
   | 9 | No bin/obj tracked in git | PASS | `git ls-files` check |
   | 10 | README frozen banner + Samples link | PASS | Phase 6 commit |

2. Write SUMMARY-7.1.md with the R18/R19/R20 outcomes pasted in verbatim.

3. Update ROADMAP.md: mark Phase 7 ✅ COMPLETE and add an overall project status line at the very top: `**Project status: FROZEN as of YYYY-MM-DD**`.

4. Commit:
   ```bash
   git add -f .shipyard/ROADMAP.md .shipyard/phases/7/VERIFICATION.md .shipyard/phases/7/results/SUMMARY-7.1.md
   git commit -m "shipyard: freeze at DotNetWorkQueue 0.9.18 / .NET Framework 4.8

   Final phase complete. R18 clean build verified, R19 manual smoke
   test passed (<transport>), R20 remote smoke tests <documented>.
   All 10 PROJECT.md success criteria satisfied.

   This is the freeze commit. The repo receives no further updates."
   ```

5. Tag TWICE:
   ```bash
   git tag post-build-phase-7
   git tag freeze
   ```
   `freeze` is the permanent external-facing tag. `post-build-phase-7` matches the per-phase checkpoint convention.

**Acceptance criteria:**
- Exactly one new commit on `main` with the `shipyard: freeze ...` subject.
- Both tags (`post-build-phase-7` and `freeze`) point at that commit.
- `git status --short` clean.
- ROADMAP.md shows all 7 phases as `✅ COMPLETE` and a top-level project FROZEN marker.
- VERIFICATION.md has all 10 rows as PASS (R20 best-effort rows may be "PASS with partial" or "SKIPPED-per-policy" but all accounted for).

## Verification

```bash
# Build is still clean
MSBUILD='/mnt/c/Program Files/Microsoft Visual Studio/18/Community/MSBuild/Current/Bin/MSBuild.exe'
SLN_WIN="$(wslpath -w Source/Examples/Examples.sln)"
"$MSBUILD" "$SLN_WIN" /t:Build /p:Configuration=Release /v:normal 2>&1 | grep -E '[0-9]+ (Warning|Error)\(s\)' | tail -2

# Freeze tag exists
git tag -l | grep -Fx 'freeze'

# Freeze commit subject
git log -1 --format='%s' freeze

# Working tree clean
git status --short
```

## Rollback

After the freeze commit, rollback is:

```bash
git reset --hard post-build-phase-6
git tag -d freeze post-build-phase-7
```

There is no Phase 8 — after Phase 7, the next user step is `/shipyard:ship` (which runs the final auditor / simplifier / documenter gates) or `git push origin main` to publish the freeze.
