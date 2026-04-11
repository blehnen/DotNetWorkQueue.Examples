# Plan 5.1: GitHub Actions CI workflow

## Context

Add `.github/workflows/ci.yml` for build verification on every push and pull
request to `main`. The workflow is the permanent build signal for the frozen
repo: the first run on the freeze commit becomes the tombstone proof that the
solution compiles clean, and every future fork or drive-by PR gets the same
signal.

Toolchain is classic `msbuild` + `nuget restore` (NOT `dotnet build`). See
CONTEXT-5.md and RESEARCH.md for the rationale.

## Dependencies

Phase 4 complete. No intra-phase dependencies.

## Tasks

### Task 1: Create `.github/workflows/ci.yml`

**Files:** `.github/workflows/ci.yml` (new file; `.github/workflows/`
directory must be created if absent)

**Action:** create

**Description:**

Write a single YAML file with this exact shape:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: windows-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup MSBuild
        uses: microsoft/setup-msbuild@v2

      - name: Setup NuGet
        uses: NuGet/setup-nuget@v2

      - name: NuGet restore
        run: nuget restore Source\Examples\Examples.sln

      - name: MSBuild (Release)
        run: msbuild Source\Examples\Examples.sln /t:Build /p:Configuration=Release /v:minimal /nologo
```

Notes on choices:
- Single job `build`, single runner `windows-latest`.
- Steps use backslashes (`Source\Examples\...`) because the runner is Windows.
- `nuget restore` before `msbuild` — packages.config requires explicit NuGet restore; `msbuild /t:Restore` is a no-op for packages.config (Phase 3 lesson).
- `/v:minimal /nologo` for minimal output noise; the warning/error summary still prints at the tail of the log, which is all CI needs.
- No test step.
- No artifact upload.
- No matrix strategy.

**Acceptance criteria:**
- `.github/workflows/ci.yml` exists at repo root path.
- File contents match the shape above (exact YAML structure).
- `yq` or basic YAML parse does not error (can be verified with `python3 -c "import yaml; yaml.safe_load(open('.github/workflows/ci.yml'))"` on WSL since PyYAML is typically available; if not, a visual sanity check of the indentation is sufficient).
- The workflow file is the ONLY new file in the commit (no source edits).

### Task 2: Commit + wait for first CI run

**Files:** commit only

**Action:** commit

**Description:**
- Stage the new workflow file: `git add .github/workflows/ci.yml`
- Commit with message `shipyard(phase-5): add GitHub Actions CI workflow`
- Note that the first run will fire on the next `push` to `main`. If the
  repo is already being pushed to GitHub at the freeze, the run should start
  automatically. If not (local-only repo), the run will fire when the user
  first pushes.
- This plan does NOT wait for or verify the CI run result — that's a
  deliverable the user confirms manually after pushing.

**Acceptance criteria:**
- Single new commit on `main` past `post-plan-phase-5` with the exact subject
  above.
- `git status --short` is clean after commit (only `CLAUDE.md` untracked).
- `.github/workflows/ci.yml` is tracked in the commit and lives at the
  expected path.

### Task 3: Write SUMMARY-5.1.md

**Files:** `.shipyard/phases/5/results/SUMMARY-5.1.md`

**Action:** create

**Description:** standard Shipyard summary format. Status `complete`. Note
that the CI workflow's first green run against the freeze commit (when
pushed to GitHub) satisfies R-CI-3 from PROJECT.md, but that confirmation is
asynchronous and the user will verify it outside this plan's scope.

Force-add: `git add -f .shipyard/phases/5/results/SUMMARY-5.1.md` because
`.shipyard/.gitignore` blocks everything under `.shipyard/` by default.

Commit subject: `shipyard(phase-5): plan 5.1 build summary`

**Acceptance criteria:**
- SUMMARY-5.1.md exists with all required sections.
- File is tracked in a commit.

## Verification

```bash
test -f .github/workflows/ci.yml && echo "workflow exists"
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/ci.yml'))" && echo "yaml valid"
grep -E '^(name|runs-on|uses|run):' .github/workflows/ci.yml | head -10
git log -2 --format='%s' HEAD~1..HEAD
git status --short  # expect only CLAUDE.md
```

## Rollback

If the workflow is somehow broken and cannot be fixed in-place:

```bash
git revert HEAD~1 HEAD  # revert the two phase-5 commits
# or, if nothing has been pushed:
git reset --hard post-plan-phase-5
```
