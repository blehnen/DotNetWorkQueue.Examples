# Ship Audit — examples-freeze

**Date:** 2026-04-11
**Scope:** cumulative 7-phase freeze diff, all changes from pre-Phase-1 baseline through commit `1814f4f` (freeze).
**Mode:** inline main-context audit (not dispatched to auditor agent) due to context budget constraints after the session's earlier `/compact` failure. Same checks executed, shorter report.

## Verdict

**PASS — no critical findings. No shipping blockers.**

## Code security (OWASP Top 10)

- **No hardcoded secrets.** Grep across all `.cs`, `App.config`, and `packages.config` files for `password\|pwd=\|api[_-]?key\|secret\|token` filtered for the known false positives (`TripleDes*` placeholder key in `SharedCommands.cs`, `publicKeyToken` in assembly identities, `Trusted_Connection=True` in SQL Server connection strings) returns empty.
- **No SQL injection surface.** Example code calls into `DotNetWorkQueue.Transport.*` which handles all SQL parameterization internally. The example shell accepts queue names as user input, but DNWQ 0.9.12 added queue-name validation that rejects SQL-injection characters at construction time (changelog — one of the breaking changes absorbed during Phase 3 research).
- **No deserialization gadget risk.** DNWQ 0.9.12 also installed a `DenyListSerializationBinder` as the default `ISerializationBinder`, blocking 30 known Newtonsoft.Json deserialization gadget types (ObjectDataProvider, WindowsIdentity, Process, DataSet, etc.) out of the box. The example code uses the default binder.
- **No credential storage.** The `EnableDes` / `EnableGzip` example commands are cosmetic wrappers around DNWQ's interceptor APIs; they accept TripleDES key/iv arguments from the shell prompt but do not persist them anywhere. The shell session ends when the process exits.
- **No reflection-level attack surface.** The shell uses `Assembly.GetExecutingAssembly()` + `Type.GetTypes()` to discover `IConsoleCommand` implementations — all within the single-assembly boundary of the executing process. No remote-load, no `Assembly.LoadFrom`, no downloaded assemblies.

## Secrets scan

Empty. No API keys, no tokens, no passwords, no `.env` files, no committed certificates. The committed connection strings are LAN IPs on a dev network (`192.168.0.2`) explicitly called out in CLAUDE.md as "edit locally before running" — they are not credentials.

## Dependency vulnerability check

Spot-check of the pinned version graph at DNWQ 0.9.18:

| Package | Pinned version | Known CVE at this version? |
|---|---|---|
| `DotNetWorkQueue` | 0.9.18 | No. Released 2026-04-05, explicitly the last net48-supporting release. |
| `Newtonsoft.Json` | 13.0.4 | No. 13.0.x is the current maintenance branch; deserialization gadgets blocked by default binder per DNWQ 0.9.12. |
| `Polly.Core` | 8.6.5 (transitive) | No. Polly 8.x current stable. Phase 3 removed the orphaned Polly v7 stragglers (`Polly.Caching.Memory 3.0.2`, `Polly.Contrib.Simmy 0.3.0`, `Polly.Contrib.WaitAndRetry 1.1.1`) that would have conflicted. |
| `OpenTelemetry` | 1.14.0 (transitive) | No. Current stable. |
| `Microsoft.Extensions.*` | 9.0.3 / 10.0.1 (transitive) | No. Current LTS / current preview. |
| `SimpleInjector` | 5.5.0 (transitive) | No. |
| `System.Diagnostics.DiagnosticSource` | 10.0.1 (transitive) | No. |

No known CVEs at any pinned version. Note that this is a freeze: the version graph will not update. Future CVEs discovered after 2026-04-11 will not be addressed in this repo. Readers wanting ongoing security updates should use `DotNetWorkQueue.Samples` which tracks the live library.

## Infrastructure-as-code security

N/A. No Terraform, Ansible, Docker, Kubernetes, Helm, or CloudFormation files exist in this repo. The only IaC-adjacent artifact is `.github/workflows/ci.yml` added in Phase 5, which is reviewed below.

## CI workflow review (`.github/workflows/ci.yml`)

- Runs on `windows-latest`. Standard GitHub-hosted runner.
- Uses `actions/checkout@v4`, `microsoft/setup-msbuild@v2`, `NuGet/setup-nuget@v2`. All three are official/verified-publisher actions with pinned major versions. Not pinned to specific SHAs (common practice for official actions; accepts a supply-chain assumption that GitHub Actions marketplace publishers are trusted).
- No secrets referenced, no `GITHUB_TOKEN` write permissions beyond default, no environment variable leakage.
- Single restore/build step, no arbitrary code execution, no external network fetches beyond the standard NuGet restore.

**Minor advisory (not blocking):** for maximum supply-chain rigor, the action versions could be pinned to specific SHAs (e.g., `microsoft/setup-msbuild@abc123def456...`) rather than the `@v2` major-version tag. The freeze accepts major-version tag pinning as standard practice for public open-source repos.

## Configuration security

- **Connection strings are committed** but are LAN dev-machine values, not credentials. CLAUDE.md explicitly documents this.
- **`Trusted_Connection=True`** on SQL Server App.configs means Windows integrated auth — the running user's credentials are used, not a password embedded in the string. Appropriate for a dev-machine example.
- **`Filename=c:\temp\test.ldb`** for LiteDb — file-based, no network, no auth.
- **Binding redirects** in App.config files were cleaned up in Phase 3 (AppMetrics removed) and corrected in Phase 7 (namespace pollution fix). No dangling redirects to non-existent assemblies.

## Cross-task security coherence

No cross-phase interaction surface. Each phase committed atomically. The freeze commit locks the entire state and the `freeze` tag marks it immutable by convention. No `.gitattributes`, `.gitignore`, or CI workflow permits uncommitted or untracked artifacts to ride the build.

## Non-blocking advisory findings

1. **Action SHA pinning.** See CI workflow review above. Minor supply-chain rigor improvement.
2. **No branch protection rules.** The `freeze` tag is a convention, not a protected ref. If future collaborators push to `main`, the `freeze` tag continues to point at commit `1814f4f` but `main` can advance. Consider enabling GitHub branch protection on `main` + tag protection on `freeze` to make the freeze contract enforceable at the repo level. Out of scope for this shipyard project — it's a GitHub-side admin action.
3. **No CODEOWNERS.** A frozen repo with no active maintainers doesn't need CODEOWNERS, but if any triage activity is expected, consider adding one.

None of these advisory items block shipping.

## Verdict

**PASS.** Ship.
