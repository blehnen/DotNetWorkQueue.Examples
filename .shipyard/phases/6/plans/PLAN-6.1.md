# Plan 6.1: Documentation — README.md + CLAUDE.md

## Context

Phase 6 finalizes user-facing docs for the freeze. Two files get edits:

- `README.md` gets a prepended "FROZEN" banner naming DNWQ 0.9.18 + .NET
  Framework 4.8 and linking to `DotNetWorkQueue.Samples`. Body preserved.
- `CLAUDE.md` (currently untracked from the initial `/init` run) gets its
  TFM reference updated from `4.7.2` to `4.8`, a note about the 0.9.18
  pin, and then committed for the first time.

No source code is touched. No build runs. Pragmatic single-pass plan.

## Dependencies

Phase 5 complete. No intra-phase dependencies.

## Tasks

### Task 1: Prepend FROZEN banner to README.md

**Files:** `README.md`

**Action:** modify (prepend)

**Description:**

Read the current README.md. Locate the first line (should be the
`DotNetWorkQueue.Examples` title) and insert the following banner above
it, leaving the rest of the file untouched:

```markdown
> **⚠ This repository is frozen.**
>
> This example set is pinned at **DotNetWorkQueue 0.9.18** targeting
> **.NET Framework 4.8** — the last upstream release that supports
> net48. DotNetWorkQueue 0.9.19 dropped .NET Framework in favor of
> .NET 10 / .NET 8 only, so this repo cannot follow forward without
> losing the classic-csproj identity that differentiates it from the
> newer samples repository.
>
> For current usage, modern APIs, and the up-to-date metrics story,
> see [DotNetWorkQueue.Samples](https://github.com/blehnen/DotNetWorkQueue.Samples).
>
> This repo now receives no further updates. The examples still build
> cleanly against the pinned package set on .NET Framework 4.8.

---

```

**Acceptance criteria:**
- `README.md` first non-blank block is the FROZEN banner shown above.
- The existing transport bullet list, 3rd-party library notes, and
  License section are unchanged. `git diff README.md` shows only
  insertions at the top of the file (no deletions, no modifications to
  the existing content).
- The banner contains a working link to
  `https://github.com/blehnen/DotNetWorkQueue.Samples`.
- No emojis beyond the single warning symbol (⚠) — matching the
  existing README's plain-text style.

### Task 2: Update and commit CLAUDE.md

**Files:** `CLAUDE.md`

**Action:** modify (currently untracked, will become tracked)

**Description:**

The `CLAUDE.md` file at repo root was created during the session's
`/init` run and has been untracked ever since. Phase 6 is the right
home to finalize and commit it.

Read the current file and make these edits:

1. Update the `## Build / run` section:
   - Change **".NET Framework 4.7.2"** → **".NET Framework 4.8"**.
   - Add a new bullet or sentence noting: "Pinned at DotNetWorkQueue
     0.9.18 (the last upstream release that supports net48; 0.9.19
     dropped the framework)."

2. Update the `## Repository purpose` section: add a sentence at the
   end: "The repo is now frozen as of 2026-04 and receives no further
   updates — see `DotNetWorkQueue.Samples` for modern usage."

3. Everything else stays. The architecture description, conventions,
   and shared-base hierarchy notes are still accurate for the frozen
   state.

**Acceptance criteria:**
- `CLAUDE.md` is staged and committed (no longer untracked).
- TFM reference is `4.8`, not `4.7.2`, in the Build / run section.
- The 0.9.18 pin is explicitly called out.
- The "frozen" note is in the Repository purpose section.
- The file is valid Markdown (no broken headings, no truncated bullets).

### Task 3: Commit and write SUMMARY-6.1.md

**Files:** commit + `.shipyard/phases/6/results/SUMMARY-6.1.md`

**Action:** commit + create

**Description:**

Stage both README.md and CLAUDE.md and commit atomically:

```bash
git add README.md CLAUDE.md
git commit -m "shipyard(phase-6): freeze banner in README, update CLAUDE.md for 0.9.18 / net48"
```

Then write `.shipyard/phases/6/results/SUMMARY-6.1.md` documenting the
edits, force-add it past `.shipyard/.gitignore`, and commit:

```bash
git add -f .shipyard/phases/6/results/SUMMARY-6.1.md
git commit -m "shipyard(phase-6): plan 6.1 build summary"
```

**Acceptance criteria:**
- Two new commits on `main` past `post-plan-phase-6` with the expected
  subjects.
- `git status --short` is empty after both commits (no untracked files
  — `CLAUDE.md` is now tracked).
- SUMMARY-6.1.md exists with standard Shipyard structure.

## Verification

```bash
head -20 README.md                    # banner visible at top
grep -c 'DotNetWorkQueue.Samples' README.md  # expect >= 1
grep -c '4\.7\.2' CLAUDE.md           # expect 0 (no stale TFM)
grep -c '4\.8' CLAUDE.md              # expect >= 1
grep -c '0\.9\.18' CLAUDE.md          # expect >= 1
git ls-files CLAUDE.md                # expect non-empty (tracked)
git status --short                    # expect empty
git log -2 --format='%s' HEAD~1..HEAD # expect two phase-6 subjects
```
