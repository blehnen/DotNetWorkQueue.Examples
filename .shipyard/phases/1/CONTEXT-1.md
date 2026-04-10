# Phase 1 Context — Spike & Baseline Capture

Decisions captured during `/shipyard:plan 1` discussion before researcher/architect dispatch.

## Decision: Line-ending noise fix

**Finding:** At session start the working tree showed 160 modified files, but investigation proved 100% of them were phantom CRLF ↔ LF diffs:

- All 159 text-file diffs had equal insertion/deletion counts (`ins == del`), the signature of a pure line-ending flip.
- Spot-check of `SharedAssemblyInfo.cs` confirmed content-identical on both sides — the same `AssemblyVersion("0.7.1")` string appears unchanged.
- No `.gitattributes` file exists. `core.autocrlf` is unset in local config.
- The repo is Windows-authored (index stores CRLF); the WSL working tree was normalized to LF by some interaction (WSL filesystem driver, a git op with different autocrlf, or a file watcher).

**Decision:** Option A — clean now, add `.gitattributes` in Phase 1 spike.

1. **Immediate cleanup (already executed before research dispatch):** `git checkout -- .` to restore working tree to match index. Zero content loss; all 160 "modifications" were phantom. Verified clean: `git status --short` returns only the untracked `CLAUDE.md` from the earlier `/init` session.
2. **Permanent fix (must be planned as a spike sub-task):** add a `.gitattributes` file with Windows-repo line-ending rules plus binary exclusions for DLL/EXE/PDB files under `Source/Examples/packages/`. Commit early in Phase 1 so every subsequent phase diffs cleanly.

**Rationale:** Without the permanent fix, any future checkout in WSL re-creates the 160-file flip, making every Phase 2+ diff unauditable. Baseline warning capture (also a Phase 1 deliverable) is meaningless on a tree with phantom modifications — you cannot tell whether a warning comes from baseline code or from the noise. The `.gitattributes` file is two lines of text for permanent immunity; deferring to Phase 4 is penny-wise/pound-foolish for a freeze where every phase diff must be auditable.

**Constraints this adds to the Phase 1 plan:**

- The `.gitattributes` task must land and commit BEFORE the baseline warning capture, so the baseline is measured against a tree that both matches the index AND has the permanent fix in place.
- The `.gitattributes` content must include `binary` rules for the `Source/Examples/packages/` tree so NuGet DLLs are not subject to any text normalization. A suggested starter set: `* text=auto eol=crlf` plus `*.dll binary`, `*.exe binary`, `*.pdb binary`, `*.snk binary`, `*.png binary`, `*.ico binary`. Architect should confirm the final rule set against actual file types present in the repo.
- After committing `.gitattributes`, run `git add --renormalize .` and verify the result is empty — if not empty, there are real normalization changes to commit as a one-shot "normalize line endings" cleanup commit, which must land separately before any Phase 2 edits touch the tree.

## Decision: scope of Phase 1 deliverables (unchanged from ROADMAP)

No change to ROADMAP. Phase 1 remains:
- R1: confirm DotNetWorkQueue 0.9.18 publishes packages for all 6 transports in use.
- R2: enumerate every file referencing `App.Metrics` / `DotNetWorkQueue.AppMetrics` / `IMetrics` / `IMetricsRoot` / `using App.Metrics`.
- Baseline warning capture: clean Release build on the post-cleanup tree, recorded as an integer (plus the full warning text for later diffing).
- Plus the new gitattributes sub-task described above.

## Decision: no worktree for Phase 1

Phase 1 is a no-source-edit spike. The only writes are:
- A new `.gitattributes` file at repo root.
- Spike notes under `.shipyard/phases/1/`.
- Possibly a one-shot normalization commit if `git add --renormalize .` produces real changes.

Working on `main` directly is appropriate. Worktree isolation matters for Phase 2+ where real source edits happen; that decision will be re-visited at `/shipyard:plan 2`.

## Decision: NuGet availability verification method

Architect's choice — no user constraint. Suggested approaches in priority order:
1. `WebFetch` against `https://api.nuget.org/v3-flatcontainer/dotnetworkqueue/index.json` (returns a JSON list of all published versions — cheapest, no binary needed).
2. Equivalent fetch for each transport package: `dotnetworkqueue.transport.sqlserver`, `.postgresql`, `.sqlite`, `.litedb`, `.redis`.
3. Cross-reference any version gaps — if any transport does not publish 0.9.18, STOP Phase 1 and raise a scope question before continuing (per ROADMAP Phase 1 work item 1).
