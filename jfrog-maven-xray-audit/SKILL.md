---
name: jfrog-maven-xray-audit
description: >-
  Explains how to run JFrog CLI `jf audit` for Maven projects against Xray:
  command entrypoints, flags, server URL rules, violation context, sub-scanners
  (SCA/IaC/SAST/secrets), exclusions, working dirs, outputs, exit behavior, and
  optional env vars. Use for Maven dependency or source audits, Xray SCA, CI
  hardening, or when the user mentions jfrog-cli-security audit behavior.
---

# JFrog CLI Maven audit (Xray / `jfrog-cli-security`)

## Canonical command

```bash
jf audit --mvn
```

Aliases: `jf aud --mvn`.

**Preferred** over the legacy hidden commands `jf audit-mvn` / `jf am`, which call the same pipeline but log a deprecation warning (`AuditSpecificCmd` → `logNonGenericAuditCommandDeprecation` in `jfrog-cli-security/cli/scancommands.go`).

## What `jf audit` does (from `jfrog-cli-security`)

- **Description** (`cli/docs/scan/audit/help.go`): On-demand source scan for CVEs, license issues, misconfigurations, secrets, and other risks; results in the terminal and the JFrog Platform.
- **Execution** (`commands/audit/audit.go`):
  - Resolves **working directories** and whether the scan is **recursive** (`getRelatedWorkingDirs`).
  - Builds a dependency / SBOM path via **BOM generator** + **SCA scan strategy** (default: build-info BOM + Xray scan-graph strategy; optional **static SCA** / Xray lib path when `--static-sca` is used — hidden flag in `cli/docs/flags.go`).
  - Sends analytics via **XSC** (`xsc.SendNewScanEvent`, `SendScanEndedWithResults`).
  - Runs parallel audit scanners (`runParallelAuditScans`), then **policy / fail-build** handling (`OutputResultsAndCmdError` + `policy.CheckPolicyFailBuildError` when `--fail` applies).
- **JAS / Advanced Security**: `CreateAuditCmd` sets `SetUseJas(true)`. If the server is not entitled for JAS, the CLI surfaces an informational message about Contextual Analysis, Secrets, IaC, and SAST (`audit.go` results writer).

Maven is identified internally as technology **`maven`** (`utils/techutils/techutils.go`), integrated with **build-info** / **Xray** graph flows (`getScanLogicOptions`, `xrayplugin.WithSpecificTechnologies`).

## Explicit Maven selection

In `AuditCmd` (`jfrog-cli-security/cli/scancommands.go`), the only technology that uses the **`--mvn`** flag (not the raw `Technology.String()` name) is Maven:

```go
// jfrog-cli-security/cli/scancommands.go — AuditCmd
if tech == techutils.Maven {
	techExists = c.GetBoolFlagValue(flags.Mvn)
} else {
	techExists = c.GetBoolFlagValue(tech.String())
}
```

Pass **`--mvn=true`** to **restrict** the audit to Maven among supported tech flags. If you omit all tech flags, **`techutils.DetectTechnologiesDescriptors`** still discovers descriptors (e.g. `pom.xml`) under the chosen working dirs — but for a **standard Maven-only** workflow, set `--mvn` explicitly.

## Server configuration (Xray URL is mandatory)

`CreateServerDetailsFromFlags` (`cli/utils.go`):

- At least one of **`--url`** (platform) or **`--xray-url`** must be set; otherwise the command errors.
- If only **`--url`** is set, **`XrayUrl`** is derived as `<url>/xray/`.
- If only **`--xray-url`** is set, **`Url`** is derived by stripping the `xray/` suffix.
- **Artifactory** and **Catalog** URLs are also derived from the platform URL when missing.

`validateConnectionInputs` requires **`XrayUrl`** to be non-empty before the scan runs.

Typical setup:

```bash
jf config add myserver --url https://acme.jfrog.io --access-token "$JFROG_ACCESS_TOKEN" --interactive=false
jf config use myserver
cd /path/to/maven-root
jf audit --mvn
```

Override per run: `--server-id`, or inline `--url` / `--xray-url`, `--access-token`, `--user`, `--password`, `--insecure-tls`.

## Violation context: `--project`, `--watches`, `--repo-path`

`validateConnectionAndViolationContextInputs` (`cli/utils.go`):

- **At most one** of `--watches`, `--project`, `--repo-path` may be set.
- **`--project`** may also come from the **`JFROG_PROJECT`** environment variable (`getProject` reads `coreutils.Project`).

**CycloneDX (`--format cyclonedx`)**:

- With **`--vuln`**: violations from context flags are not represented in CDX; the CLI warns and emits vulnerability-oriented CDX only.
- **Without `--vuln`**: CycloneDX output **cannot** carry violations — the command **fails** if any of `--watches`, `--project`, `--repo-path` is set.

## Output format (`--format`)

From `cli/docs/flags.go` / live `jf audit --help`:

| Value        | Notes |
|-------------|--------|
| `table`     | Default. |
| `json`      | Does **not** include some Advanced Security scan details (per flag help text). |
| `simple-json` | Compact JSON variant. |
| `sarif`     | SARIF output. |
| `cyclonedx` | SBOM-oriented; violation / context constraints above. |

**`--extended-table`**: Extra columns (e.g. CVSS, Xray issue id); **only** when `format` is `table`.

**`--sbom`**: Show all SBOM components in table / cyclonedx-style output, not only affected (ignored for other formats per help).

**`--licenses`**: Include license list in output.

**`--min-severity`**: `Low`, `Medium`, `High`, or `Critical`.

**`--fixable-only`**: Only issues with a fix version.

**`--vuln`**: List **all** vulnerabilities regardless of Xray policy (affects how violation context behaves with SARIF / CDX per help and validation).

## Fail build / exit behavior

- **`--fail`**: Default **`true`**. When **`--watches`**, **`--project`**, or **`--repo-path`** is used and a **Fail build** rule matches, the command returns **exit code 3** (see flag help in `flags.go`).
- After results are printed, **`OutputResultsAndCmdError`** calls **`policy.CheckPolicyFailBuildError`** when `failBuild` is true.

## Working directories and discovery

- **`--working-dirs`**: Comma-separated **relative** paths; determines audit roots (`CreateAuditCmd` → `SetWorkingDirs(splitByCommaAndTrim(...))`).
- If **omitted** and SCA / SBOM is involved, **`getRelatedWorkingDirs`** sets **`isRecursiveScan`** so the CLI **recursively** discovers targets from the repo root (`audit.go`).
- **`--exclusions`**: Semicolon-separated patterns; `*` and `?` wildcards. **Default** string is built from **`utils.DefaultScaExcludePatterns`**: `*.git*`, `*node_modules*`, `*target*`, `*venv*`, `*test*`, `dist` (`utils/utils.go`). Maven `target/` is excluded by default from **sub-project** audit traversal; adjust if you intentionally need to scan under those paths.

## Maven wrapper

- **`--use-wrapper`**: Default **`true`**. Description: use **Gradle or Maven wrapper** when present (`flags.go`). Set `--use-wrapper=false` to force non-wrapper Maven.

## Gradle-only flag (do not use for Maven)

- **`--exclude-test-deps`**: Help text marks it **`[Gradle]`** only — not for Maven.
- **`--use-included-builds`**: **`[Gradle]`** composite builds only.

## Selective sub-scanners (JAS)

Flags (`cli/docs/flags.go` → `flagsMap`):

| Flag | Role |
|------|------|
| `--sca` | SCA sub-scan; with `--sca` alone, **contextual analysis** runs too unless disabled. |
| `--without-contextual-analysis` | Disables contextual analysis **only** when combined with `--sca`. Cannot use without `--sca` (`getSubScansToPreform`). |
| `--secrets` | Secrets detection. |
| `--validate-secrets` | Token validation on secrets; **requires** `--secrets`. |
| `--iac` | Infrastructure-as-code scan. |
| `--sast` | SAST scan. |
| `--add-sast-rules` | Extra SAST rules JSON file; **requires** `--sast` to be in the requested sub-scans (`validateSnippetDetection` / `AuditCmd`). |

**`--snippet`**: Snippet-level detection; only allowed when SCA is requested or **`--sbom`** is set (`validateSnippetDetection`).

Sub-scan types in code: `sca`, `contextual_analysis`, `iac`, `sast`, `secrets`, `secrets_token_validation`, `malicious_code` (`utils/utils.go`).

## Threads and tooling paths

- **`--threads`**: Parallelism for scanning (default **3** in `jf audit --help`; flag definition uses `cliutils.Threads` from core).
- Hidden / advanced: **`--analyzer-manager-path`**, **`--xray-lib-plugin-path`**, **`--output-dir`** (partial results), **`--skip-auto-install`**, **`--allow-partial-results`** (see `commandFlags` for `Audit` in `flags.go`).

Environment (**JFrog CLI** docs / core): **`JFROG_CLI_ANALYZER_MANAGER_VERSION`** pins Analyzer Manager version for security commands.

## Optional: releases repo for Maven-related CLI behavior

For **`jf mvn`** and **audit on Maven/Gradle**, JFrog CLI documents **`JFROG_CLI_RELEASES_REPO`** as `<serverId>/<repo>` pointing at a repository that proxies **`https://releases.jfrog.io`** (see `jfrog-cli/docs/common/env.go`). Use when internal tooling or resolution fails during Maven-audit flows.

## Copy-paste examples

**Minimal Maven SCA from repo root**

```bash
jf audit --mvn
```

**CI-friendly JSON + explicit server**

```bash
jf audit --mvn --server-id ci --format json --fail=true
```

**Monorepo: only certain Maven roots**

```bash
jf audit --mvn --working-dirs services/order-api,libs/core
```

**Policy via Xray watch**

```bash
jf audit --mvn --watches "my-watch-name" --fail=true
```

**Maven wrapper disabled (system `mvn`)**

```bash
jf audit --mvn --use-wrapper=false
```

**SCA only (no contextual analysis)**

```bash
jf audit --mvn --sca --without-contextual-analysis
```

## Implementation map (`jfrog-cli-security`)

| Area | Location |
|------|-----------|
| Command registration, aliases, `AuditCmd`, `CreateAuditCmd`, deprecated `audit-mvn` | `cli/scancommands.go` |
| Server / Xray URL validation, violation context, sub-scan flag validation | `cli/utils.go` |
| Audit flag keys and help text | `cli/docs/flags.go` |
| Audit run loop, BOM + scan graph, results | `commands/audit/audit.go` |
| Parameters / graph / technologies | `commands/audit/auditparams.go`, `auditbasicparams.go` |
| Default exclusion patterns | `utils/utils.go` (`DefaultScaExcludePatterns`) |
| Technology constants (Maven → `maven`) | `utils/techutils/techutils.go` |

## Distinction from `jf scan`

- **`jf audit`**: **Source** / project context; Maven graph via build-info + Xray **scan graph** (and optional static-SCA path). Implemented in **`jfrog-cli-security`** as above.
- **`jf scan` (`xr scan`)**: **Binary / file**-oriented Xray scanning with file specs — different command in the same security plugin (`cli/scancommands.go`).

For **Maven dependency and source risk** in a standard `pom.xml` project, **`jf audit --mvn`** is the primary CLI entrypoint.
