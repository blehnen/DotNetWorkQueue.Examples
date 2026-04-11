# CONTEXT — Phase 4 (Code & Repo Hygiene)

Captured 2026-04-11 before researcher/architect dispatch. These are the user-locked decisions and pre-discovered facts that frame Phase 4 planning, so downstream agents work from shared ground truth instead of re-deriving everything from ROADMAP.md.

## Phase scope (from ROADMAP lines 69–84)

Delivers R10, R12, R13, R14, R15. Effort S (1–3h), Risk LOW. Mechanical edits with no build-graph interaction. Maps to PROJECT.md Success Criteria #7, #8, #9.

## Pre-discovered facts (verified 2026-04-11)

1. **R10 bug confirmed:** `Source/Examples/LiteDb/Consumer/LiteDbConsumer/Commands/ConsumeMessage.cs:30` declares `namespace SQLiteConsumer.Commands`. Needs to become `LiteDbConsumer.Commands`. Research must check whether any `using SQLiteConsumer.Commands;` references exist (unlikely — each shell discovers commands via reflection over `GetExecutingAssembly()`, not cross-project imports).

2. **R12 baseline confirmed:** `SharedAssemblyInfo.cs` at repo root declares `AssemblyVersion`, `AssemblyFileVersion`, and `AssemblyInformationalVersion` all at `"0.7.1"`. All three must bump to `"0.9.18"`.

3. **R13 scope narrowed:** `find` locates 56 `bin`/`obj` directories on disk across `Source/Examples/**`. **Only the 8 Rpc/** bin/obj directories are "orphaned"**:
   - `Source/Examples/PostGresSQL/Rpc/PostGreSQLRpcConsumer/{bin,obj}`
   - `Source/Examples/PostGresSQL/Rpc/PostGreSQLRpcProducer/{bin,obj}`
   - `Source/Examples/Redis/Rpc/RedisRpcConsumerAsync/{bin,obj}`
   - `Source/Examples/Redis/Rpc/RedisRpcProducer/{bin,obj}`
   - plus 4 more under `Source/Examples/SQLite/Rpc/**` and `Source/Examples/SQLServer/Rpc/**` (confirm during research).

   The other 48 `bin/obj` dirs are **active build output from Phase 3's `msbuild /p:Configuration=Release`** and **must not be deleted** — they belong to projects that are in `Examples.sln` and will be regenerated on the next build anyway. ROADMAP is explicit: "orphaned RPC build artifacts" only.

   The Rpc project source folders themselves (e.g. `Source/Examples/Redis/Rpc/RedisRpcProducer/*.cs`, `.csproj`, `App.config`) **stay put** — they are out of scope for Phase 4. ROADMAP scopes R13 to `bin/obj` subdirectories only.

   **Git state of these Rpc bin/obj dirs:** all untracked (0 tracked bin/obj repo-wide). This means R13 is purely a working-tree `rm -rf` — no `git rm`, no commit content from R13.

4. **R14 is a no-op (verified):** `git ls-files | grep -E '/(bin|obj)/'` returns 0 results. No bin/obj content is currently tracked. `.gitignore` already has `[Bb]in/` and `[Oo]bj/` per CONCERNS.md. The plan task for R14 becomes "verify zero tracked bin/obj and zero gaps in .gitignore coverage" — a read-only check, not an edit.

5. **R15 scope confirmed:** exactly 6 `App.config` files contain `192.168.0.58`:
   - `Source/Examples/PostGresSQL/Consumer/PostGreSQLConsumerAsync/App.config`
   - `Source/Examples/PostGresSQL/Consumer/PostGresSQLConsumer/App.config`
   - `Source/Examples/PostGresSQL/Producer/PostGresSQLProducer/App.config`
   - `Source/Examples/SQLServer/Consumer/SqlServerConsumer/App.config`
   - `Source/Examples/SQLServer/Consumer/SqlServerConsumerAsync/App.config`
   - `Source/Examples/SQLServer/Producer/SqlServerProducer/App.config`

   All 6 must change `192.168.0.58` → `192.168.0.2`. SQLite/LiteDb are file-based and have no IP. Redis `App.config` files already use `192.168.0.2` (leave alone — ROADMAP says "Leave the existing Redis 192.168.0.2 as-is").

## User decisions locked in

- **Commit strategy:** single atomic commit for Phase 4, consistent with Phase 1–3. Message: `shipyard(phase-4): code & repo hygiene`. Working tree must be clean on both sides of the commit except the untracked `CLAUDE.md` (keep it out of the commit as in every prior phase).
- **Build gate:** after the edits land, `nuget restore` + `msbuild /p:Configuration=Release` must emit **0 warnings / 0 errors** — the Phase 1 baseline ceiling holds. If a warning appears, revert and investigate.
- **R13 implementation:** delete the 8 (or however many the researcher finds) Rpc bin/obj directories via shell `rm -rf`. No `git` verb needed since they're untracked. Verify `git status` is unchanged after deletion to prove nothing tracked was touched.
- **R14 implementation:** one-shot verification step, not an edit. Record command + output (`git ls-files | grep -E '/(bin|obj)/'` returns empty) in the plan's verification section.
- **R10 implementation:** single `Edit` operation on one file + follow-up grep for `SQLiteConsumer.Commands` across the repo to confirm no dangling references. Do not rename the class or any other identifier.
- **R12 implementation:** `Edit` on `SharedAssemblyInfo.cs` changing three `0.7.1` → `0.9.18` occurrences (replace_all safe — only three attribute lines match).
- **R15 implementation:** six targeted `Edit` calls, one per App.config. Change only the `192.168.0.58` literal in each; leave surrounding XML, queue name, credential, and whitespace untouched.

## Plan shape the architect should produce

Single wave, **one plan file** (`PLAN-4.1.md`). Phase 4 is mechanical, has no internal ordering dependencies among R10/R12/R13/R14/R15 (each requirement touches an orthogonal set of files), and the whole thing fits inside an atomic commit. Splitting into multiple plans would just create wave ordering overhead for no benefit.

The plan should contain **at most 3 tasks** (Shipyard plan structure rule):

- **Task 1 — Source edits (R10 + R12 + R15):** edit the LiteDb namespace, bump SharedAssemblyInfo to 0.9.18, correct 6 App.config IP strings. Single coherent "source tree hygiene" step.
- **Task 2 — Working-tree cleanup (R13 + R14):** delete Rpc bin/obj orphans, verify git state, verify `.gitignore` coverage. Single "working tree hygiene" step.
- **Task 3 — Build gate + atomic commit:** `nuget restore` + `msbuild Release`, confirm 0/0, `git add` the modified files (not the untracked `CLAUDE.md`), commit with the Phase 4 message. Single "verify and land" step.

Each task has concrete acceptance criteria derived directly from the ROADMAP success criterion ("Solution still builds clean in Release. No bin/ or obj/ directories tracked in git. LiteDb namespace fix verified. SharedAssemblyInfo at 0.9.18.") plus the R15 IP correction.

## Non-goals for Phase 4

- Do not touch any `.csproj` reference, HintPath, or package reference. Phase 3 already landed those edits.
- Do not rename the `LiteDbConsumer` class, project, or any other identifier — only the declared namespace line.
- Do not touch the Rpc source code, csproj files, or packages.config files — only their bin/obj output dirs.
- Do not delete any `bin/obj` dir outside the Rpc folders.
- Do not stage, commit, or delete the untracked `CLAUDE.md`. It stays out of the Phase 4 commit as it did in Phases 1–3.
- Do not change `192.168.0.2` in any existing file (Redis). Only correct `192.168.0.58`.
- Do not add or remove any `.gitignore` entry unless R14 verification reveals a gap (none expected — CONCERNS.md already confirmed coverage).
