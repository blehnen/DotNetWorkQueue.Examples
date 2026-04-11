# Phase 7 Context — Verification & Freeze Commit

Phase 7 is the final phase. It verifies the frozen state is actually runnable, then lands the freeze commit.

## Decisions

### Smoke tests run manually by the user

R19 (mandatory SQLite/LiteDb round-trip) and R20 (best-effort SQL Server + Postgres + Redis) are manual. The user launches the WinForms console-shell executables in a Windows terminal (NOT WSL), types the shell commands per a prepared script, and confirms the round-trip works. The user then tells the main planning context what happened, and those results get recorded in SUMMARY-7.1.md.

**Rationale:** the WinForms shell cannot be automated from WSL without meaningful new scaffolding (a separate console harness), and CONTEXT-3.md's freeze-scope discipline says no new code beyond the specific PROJECT.md requirements. A user-driven manual test is honest, simple, and sufficient for a one-way-door freeze.

### R19 is blocking; R20 is not

- **R19** must pass before the freeze commit lands. If the SQLite (or LiteDb) round-trip fails, Phase 7 stops and escalates — something is broken in the post-0.9.18 code paths that the compile did not catch.
- **R20** can be skipped entirely or attempted per-transport at the user's discretion. Each transport either passes, fails (documented with a one-line note), or is explicitly "not attempted". The freeze commit ships even if 0 of the 3 remote transports are tested, as long as R19 passes.

### Final Release build is part of Phase 7

Before the manual smoke tests, Phase 7 runs one more clean Release build via MSBuild from WSL to prove determinism and catch any last-minute drift between Phase 6's text edits and the build state. This is a fast re-run (package cache is already populated from Phase 3).

### The freeze commit is an empty-tree marker

Phases 1–6 already landed all the source changes. There is no Phase 7 source edit. The "freeze commit" is a single commit that:

1. Updates `.shipyard/ROADMAP.md` marking Phase 7 complete.
2. Writes SUMMARY-7.1.md with R18 result + R19 result + R20 results.
3. Writes VERIFICATION.md documenting the full PROJECT.md success criteria checklist (all 10).
4. Optionally writes a short `docs/FREEZE.md` or equivalent at repo root — actually no, that duplicates README.md's banner. Skip.
5. Tags `freeze` (no phase number — this is THE freeze tag that external consumers will reference). Plus the usual `post-build-phase-7` tag for consistency.

### No CI trigger in Phase 7

The Phase 5 CI workflow will fire automatically on the `push` of the freeze commit once the user pushes to GitHub. That confirmation is asynchronous and outside this plan's scope. Phase 7 only verifies the LOCAL build is clean.

## What Phase 7 does NOT do

- No source edits.
- No package bumps, no TFM changes.
- No new test harness code.
- No GitHub archive (explicitly out of scope per PROJECT.md).
- No push to origin (user decides when and where to push).
- No `/shipyard:ship` orchestration (that's a separate command the user runs afterward).
