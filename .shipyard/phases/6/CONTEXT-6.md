# Phase 6 Context — Documentation

Phase 6 updates user-facing documentation to reflect the frozen state.

## Decisions

1. **README.md banner shape:** insert a bolded "FROZEN" block at the very top of the file, above the existing `DotNetWorkQueue.Examples` title. Content: project is frozen at DotNetWorkQueue `0.9.18` on .NET Framework `4.8`, reason (upstream dropped net48 in 0.9.19), link to `https://github.com/blehnen/DotNetWorkQueue.Samples` for current usage.
2. **Preserve the existing README body** — the transport bullet list, the 3rd-party library acknowledgments, and the License section all stay as-is. Only the top gets a banner prepended.
3. **CLAUDE.md handling (loose end from /handoff):** the `CLAUDE.md` file at repo root has been untracked since the session's `/init` run. Phase 6 is the natural home to commit it — it IS the documentation file that Phase 6's R17 requires updating. Updates inside the same commit:
   - TFM reference `4.7.2` → `4.8`
   - Add a one-line note: "Pinned at DotNetWorkQueue 0.9.18 (last net48-supporting release)."
   - Optionally update the intro paragraph to reflect the freeze (mention it's the final state).
   - Everything else in CLAUDE.md (architecture summary, conventions) stays — it's still accurate for the frozen state.
4. **No link check** — the Samples repo URL is public and stable. No runtime verification.
5. **No atomic commit invariant needed** — README.md and CLAUDE.md edits are independent of each other and independent of any build. They can land as one commit or two; one is simpler, landed.

## What Phase 6 does NOT do

- No source edits.
- No build verification (pure text changes).
- No CI workflow changes.
- No package or config changes.
- Does not archive the GitHub repo — that's a GitHub-side action outside the freeze's scope (explicitly called out as a non-goal in PROJECT.md).
