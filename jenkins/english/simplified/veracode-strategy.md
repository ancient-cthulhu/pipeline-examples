# Veracode Security Pipeline for Jenkins (Simplified)

Two scan types only:

- **Agent-Based SCA** on every build (third-party dependency analysis)
- **Policy Scan** on top-level branches only (production certification)

**Supported Technologies**: Java (Maven/Gradle/Ant), .NET, Node.js, Python, Go, PHP, Ruby, Scala, and more.

**Pipeline Type**: Declarative, designed for **Multibranch Pipeline** jobs (or equivalent: GitHub Organization, Bitbucket Team, GitLab Group). Top-level branch detection requires a multibranch context.

**Host**: Windows-native (PowerShell). Agents need `java`, `maven`, `powershell`, and `curl` on `PATH`.

---

## Scanning Strategy

| Context | Veracode Product | Gate | Purpose |
|---------|------------------|------|---------|
| Every build | Agent-Based SCA | No | Dependency vulnerability analysis |
| `main` / `master` / `develop` | Policy Scan | Optional | Production certification |

Top-level branches are configurable via `TOP_LEVEL_BRANCHES` (regex, default `main|master|develop`). Change requests never trigger a Policy Scan.

---

## Pipeline Structure

```text
Trigger: SCM push / change request
        |
        v
Checkout  (always)  -> resolves IS_TOP_LEVEL flag
        |
        v
Build (Maven)        when IS_TOP_LEVEL
Package Artifacts    when IS_TOP_LEVEL  -> stash verascan bundle
        |
        v
Veracode Security Scans (parallel)
   |                         |
   v                         v
Agent-Based SCA          Policy Scan
(every build,            (top-level
 token-gated)             branch only)
```

Build and Package run only when a Policy Scan will follow, so feature branches and change requests run SCA alone.

---

## Required Jenkins Credentials

Configure under **Manage Jenkins > Credentials > System > Global credentials**.

| Credential ID | Type | Required | Description |
|---------------|------|----------|-------------|
| `veracode-api-id` | Secret text | Yes | Veracode API ID |
| `veracode-api-key` | Secret text | Yes | Veracode API Key |
| `srcclr-api-token` | Secret text | No | SCA Agent token. If absent, the SCA stage skips gracefully |

### Optional Environment Variables

Set at folder, job, or pipeline level.

| Variable | Default | Description |
|----------|---------|-------------|
| `VERACODE_APP_NAME` | `org/repo` | Override the Veracode application profile name |
| `MAVEN_POM_PATH` | `app/pom.xml` | Path to the Maven pom |
| `VERACODE_SOURCE_DIR` | `app` | Directory the autopackager scans |
| `TOP_LEVEL_BRANCHES` | `main\|master\|develop` | Regex of branches that get a Policy Scan |

### How to Obtain Credentials

1. Sign in to the [Veracode Platform](https://analysiscenter.veracode.com)
2. Profile menu (top right) > **API Credentials** for the API ID and Key
3. For SCA: **Agent-Based Scan > Workspace > Agents > Actions > Create > Windows > PowerShell > Create Agent & Generate Token**

---

## Required Jenkins Plugins

| Plugin | Purpose |
|--------|---------|
| Pipeline (Declarative) | Base declarative syntax |
| Pipeline: Multibranch | Branch and change request detection |
| Credentials Binding | `withCredentials` and `credentials()` |
| Workspace Cleanup | `cleanWs` in `post { always }` |
| Timestamper | `timestamps()` option |

---

## Stage Descriptions

### 1. Checkout (always)

Checks out source and resolves `IS_TOP_LEVEL` from `BRANCH_NAME`, `CHANGE_ID`, and `TOP_LEVEL_BRANCHES`. This single flag gates Build, Package, and Policy Scan.

### 2. Build (Maven) (top-level only)

Runs `mvn -B -f <pom> clean package -DskipTests`. Replace with your own build, or remove it if the autopackager can build your project directly.

### 3. Package Artifacts (top-level only)

Installs the Veracode CLI, runs the autopackager (`veracode package --source <dir> --output verascan --trust`), validates that artifacts exist, then stashes `verascan/`, `artifact_list.txt`, and `app_name.txt` for the Policy Scan.

### 4. Agent-Based SCA (every build)

Binds `srcclr-api-token`, installs the SCA agent from `sca-downloads.veracode.com/ci.ps1`, and runs `scan --recursive --update-advisor` against the checked-out source. SCA scans source directly, so it does not need the packaged artifacts. If the token credential is missing, the stage logs a message and continues.

About 80% of vulnerabilities come from dependencies, so SCA runs on every build regardless of branch.

### 5. Policy Scan (top-level only)

Unstashes the bundle, downloads the latest Java API Wrapper from Maven Central, and runs `UploadAndScan` with directory upload (`-filepath verascan`). Provides the official certification and traceability for frameworks such as SOC 2 and PCI DSS.

---

## Application Profile Names

The profile name defaults to `org/repo` and is the same for every branch. In a multibranch job `JOB_NAME` is `org/repo/branch`, so the pipeline drops the final path segment (the branch) to keep all branches certifying against one profile. Set `VERACODE_APP_NAME` to override with a fixed value.

| Jenkins Job (`JOB_NAME`) | Resolved App Name |
|--------------------------|-------------------|
| `acme-corp/api-service/main` | `acme-corp/api-service` |
| `acme-corp/api-service/feature%2Fauth` | `acme-corp/api-service` |

---

## Scan Results

**SCA**: results upload to the workspace tied to the agent token. Review under **Scans & Analysis > Software Composition Analysis** in the Veracode Platform.

**Policy Scan**: review findings, compliance status, and trends in the application profile at [analysiscenter.veracode.com](https://analysiscenter.veracode.com).

---

## Customization

### Change which branches get a Policy Scan

Set `TOP_LEVEL_BRANCHES`, for example `main|release/.*`.

### SCA scan options

Adjust the agent arguments in the SCA stage, for example add `--no-upload` for local-only runs, or set `$env:SRCCLR_REGION` for European or US Federal regions.

### SCA fail-on-policy gate (optional)

The SCA stage is non-gating by default. To fail the build on SCA policy violations, set the relevant `SRCCLR_*` threshold environment variables or use `srcclr --pass-fail` semantics, then remove the surrounding `try/catch` so failures propagate.

### Build step

If the autopackager cannot build your project, keep or adjust the Build stage (Maven shown; swap for Gradle, npm, etc.).

---

## Troubleshooting

### Policy Scan stage does not run

`IS_TOP_LEVEL` resolved to `false`. Confirm `BRANCH_NAME` matches `TOP_LEVEL_BRANCHES` and that the build is not a change request (`CHANGE_ID` unset).

### `unstash` fails in Policy Scan

The Package stage failed before `stash` ran. Confirm the autopackager produced at least one `.war` / `.jar` / `.zip`, or add an explicit build step.

### Agent-Based SCA skipped

The `srcclr-api-token` credential is not configured, or the scan threw. Add the credential, or review the stage log for the captured error message.

### Sandbox / Policy upload fails

Verify `veracode-api-id` and `veracode-api-key` are configured with upload permissions and that the application profile name has no invalid characters.

---

## Quick Start

1. Copy `Jenkinsfile` to the root of your repository
2. Create a **Multibranch Pipeline** job pointing at your repository
3. Add credentials: `veracode-api-id`, `veracode-api-key`, and optionally `srcclr-api-token`
4. (Optional) Set `VERACODE_APP_NAME`, `MAVEN_POM_PATH`, `VERACODE_SOURCE_DIR`, `TOP_LEVEL_BRANCHES`
5. Push to a top-level branch to trigger Build, Package, and Policy Scan; push elsewhere for SCA only
6. Review SCA and Policy results in the Veracode Platform

---

## Resources

- [Veracode Documentation](https://docs.veracode.com)
- [Veracode CLI Installation](https://docs.veracode.com/r/Install_the_Veracode_CLI)
- [Java API Wrapper](https://docs.veracode.com/r/c_about_wrappers)
- [Install the SCA agent with PowerShell](https://docs.veracode.com/r/Install_the_Veracode_SCA_Agent_with_PowerShell)
- [SCA Agent-Based Scans](https://docs.veracode.com/r/Agent_Based_Scans)
- [Jenkins Declarative Pipeline Syntax](https://www.jenkins.io/doc/book/pipeline/syntax/)
