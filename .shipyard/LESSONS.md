# Shipyard Lessons Learned

## [2026-04-11] examples-freeze — 7-phase freeze to DNWQ 0.9.18 / .NET 4.8

### What Went Well

- **Atomic commit + checkpoint discipline held across all 7 phases.** Every phase committed as a tight unit tagged at `post-plan-phase-N`, `pre-build-phase-N`, and `post-build-phase-N`. Rollback was always one `git reset --hard <tag>` away. No phase needed it, but knowing it was available made risky phases (especially Phase 3) safer to execute.
- **Pre-commit build verification gate.** Every source-touching phase ran `msbuild /t:Build /p:Configuration=Release /v:normal` BEFORE committing, with hard rollback on any exit != 0 or warning count > Phase 1 baseline. This caught exactly zero issues — but only because it was always there as the gate. Without it, Phase 3's coupled package bump + code strip could have committed a broken state to `main`.
- **The research-driven plan shape.** Every phase's RESEARCH.md surfaced real facts (exact file counts, current TFM inventory, AppMetrics code shape, DNWQ 0.9.18 nuspec dependency list) before the architect dispatched. That let the plan's acceptance criteria reference concrete integers rather than aspirational language.
- **Phase 1's baseline capture as the immutable ceiling.** Recording "0 warnings, 0 errors" as the locked-in baseline, then enforcing `warnings ≤ baseline` at every subsequent phase gate, meant we never had to argue about what counted as "acceptable drift". Zero drift was the bar, and it held through all 7 phases.
- **CONTEXT-N.md capturing user decisions BEFORE research/plan dispatch.** Decisions like "work on main directly", "absorb ≤10 files of mechanical breaks, structural = rollback", "strip AppMetrics don't port to OTel" were captured explicitly in each phase's context file. Downstream agents and the main context both referenced those decisions without re-litigating. Major win for coherence across 7 phases in a session that spanned multiple compaction-breaking conversations.
- **Pragmatic fallback when agents timed out.** Three consecutive builder-agent dispatches hit internal turn/token budgets (Phase 1 researcher, Phase 2 builder, Phase 3 researcher+builder). Main-context execution of the same plan preserved the contract without rewriting the workflow.

### Surprises / Discoveries

- **`xml.etree.ElementTree.register_namespace` is unreliable.** Despite calling `ET.register_namespace('', MSBUILD_NS)` before parsing csproj files, the serializer wrote `ns0:` prefixes on every element. Separately, on App.config files it promoted the inner `<assemblyBinding>` default namespace onto the root `<configuration>` element — which was syntactically valid XML but semantically invalid for .NET Framework's config parser, producing an opaque "side-by-side configuration is incorrect" runtime error. **Both quirks required sed post-processing to fix.** Lesson: for XML manipulation of .NET Framework config files, use `lxml` instead of ElementTree, OR do text-level sed transforms with precise element-name scoping. Do not rely on ElementTree namespace round-trip fidelity.

- **`msbuild /t:Restore` is a no-op for classic csproj + packages.config.** It's a PackageReference-only target. Phases 1 and 2 got away with using it because the `packages/` cache was already populated from pre-session developer builds — the restore reported "Nothing to do" every time. Only Phase 3's cache purge exposed the trap: the purged directory stayed empty and the build failed to resolve any DLL. **Fix: download standalone `nuget.exe` from `dist.nuget.org` and use `nuget restore Examples.sln` directly.** This also drove the Phase 5 CI workflow to use `NuGet/setup-nuget@v2` (not `actions/setup-dotnet`).

- **MSBuild.exe rejects WSL paths.** Invoking MSBuild from WSL with a `/mnt/f/...` solution path produces `MSBUILD : error MSB1001: Unknown switch` because MSBuild.exe is a Windows binary that treats the string as a switch. **Fix: translate via `wslpath -w` FIRST**: `SLN_WIN="$(wslpath -w Source/Examples/Examples.sln)"; "$MSBUILD" "$SLN_WIN" /t:Build ...`. Cemented as the standard invocation pattern from Phase 1 onward.

- **Non-metrics breaking changes across 10 minor releases were minimal.** I expected DNWQ 0.8.0 → 0.9.18 to surface multiple API breaks in example code. Only ONE break appeared during the Phase 3 build: `AbortWorkerThreadsWhenStopping` was removed from `IWorkerConfiguration` in DNWQ 0.9.13. Every other load-bearing public API surface the examples touch (`QueueContainer<TInit>`, `QueueCreationContainer<TInit>`, `IProducerBaseQueue`, `QueueMessage<T, IAdditionalMessageData>`, `QueueConnection`, `ConsumerQueueTypes`) was stable. The absorption budget (10 files / 30 min / mechanical only) was 90% unused.

- **Phase 3's clean build did NOT validate App.config XML semantics.** MSBuild copies App.config files from source to output without parsing them as configurations. The broken App.config (Phase 3's namespace pollution) compiled and built cleanly, passed Phase 3's build gate, passed Phases 4, 5, 6 without any issue, and only surfaced when Phase 7 actually LAUNCHED the .exe and the CLR tried to load the config at process startup. **Lesson: build-passes-therefore-it-works is not the same as it-runs. A runtime smoke test is a real gate, not a formality.**

- **The builder-agent dispatch pattern is fragile for heavy multi-file work.** Three consecutive builders hit turn/token budgets mid-work in this project. The cause was unclear — possibly the agent's internal context budget was too small for the combined (read plan + read research + read multiple source files + write transform + run msbuild + parse log) workflow. Main-context fallback solved it but defeats the point of delegation.

### Pitfalls to Avoid

- **Don't use `xml.etree.ElementTree` to modify XML files with namespaces.** Use `lxml` or text-level transforms with specific element-name scoping. If you must use ElementTree, always post-process the output to verify the root element's namespace didn't drift.

- **Don't assume `msbuild /t:Restore` handles packages.config.** Always test against a purged package cache. Download and chmod+x `nuget.exe` standalone for any phase that needs to regenerate the cache.

- **Don't pass WSL paths to Windows binaries.** `wslpath -w` or direct Windows drive paths only.

- **Don't trust a clean build to prove runtime correctness.** Especially for apps that have startup-time config parsing, manifest validation, binding redirects, or other external resource loading. For .NET Framework repos, always launch at least ONE `.exe` during verification.

- **Don't skip the SUMMARY.md even when executing in main context.** The post-hoc write-up is what captures the non-obvious decisions, plan defects, and narrative for future readers. The machine-readable artifacts (commit hashes, tag names) aren't enough on their own.

- **Don't let post-commit sweep patterns be too broad.** Phase 2's verification grep `v4.7.2|v4.5.2|net472|net452` accidentally matched HintPath path components in csproj files (intentionally deferred to Phase 3). Element-name-scoped greps (`<TargetFrameworkVersion>v4\.7\.2</TargetFrameworkVersion>`) are the right shape. Same lesson applied again in Phase 3's `DotNetWorkQueue.0.8.0` grep which caught orphaned vendored-DLL HintPaths that the transform had missed.

- **Don't guess method signatures from method names.** Phase 7's initial R19 smoke-test script guessed wrong on `Send`, `StartQueue`, and the CreateQueue two-step pattern, causing multiple round-trips with the user. Always read the actual method signature from source before writing shell command instructions.

### Process Improvements

- **Capture decisions in CONTEXT-N.md before any agent dispatches.** One of the key wins this project had: every phase had a CONTEXT file that pre-captured the absorption budget, the commit atomicity invariant, the work-on-main decision, the HintPath deferral, etc. Researcher and architect agents read it as required context and their plans referenced the user's explicit decisions rather than guessing.

- **Run at least one runtime smoke test before declaring a build-focused project complete.** Even if it's manual, even if it's just launching one `.exe` and typing one command. The cost of the test is low; the cost of a broken binary at freeze time is very high. Phase 7's R19 was load-bearing in this project even though the build gate had been green for 7 phases.

- **For subagent work that produces files, verify the file exists AND has content after the agent returns.** Several agent dispatches in this project returned a truncated summary message but HAD actually written the output file before running out of turns. A quick `ls -la` + `wc -l` check before moving on unblocks the flow without needing to re-dispatch.

- **For multi-file mechanical transforms, prefer text-level sed over XML-parsing Python** when the target files have XML namespaces. ElementTree's namespace handling is the single biggest source of defects in this project (both the ns0: prefix leak and the App.config root-namespace promotion). A sed script with precise element-name patterns would have been faster and more reliable for these specific transforms.

- **Tag the freeze with a simple, external-facing name (`freeze`) in addition to the per-phase checkpoint.** External consumers can `git checkout freeze` to reference the permanent state without knowing about Shipyard's phase numbering. The per-phase tags are internal to the project's execution history; the `freeze` tag is the public handle.

- **For small pragmatic phases, skip the formal agent dispatch and execute in main context directly.** Phases 4, 5, 6, and 7 were all simple enough (single-digit file counts, no build complexity) that the plan+build+review+verify agent chain added overhead with no correctness benefit. Writing the plan, executing the work, and committing from main context was faster and had the same outcome. Reserve agent dispatch for phases with genuine parallelism opportunities or heavy investigation needs.
