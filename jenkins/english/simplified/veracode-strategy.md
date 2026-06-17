# Veracode Security Pipeline for Jenkins
 
## Overview
 
The goal is to run the Veracode scans directly inside your existing build pipeline so security runs automatically on every build, with no separate process to maintain. Agent-Based SCA covers your third-party dependencies, the Container/IaC/Secrets scan covers infrastructure-as-code misconfigurations and hardcoded secrets in your source, and the Policy Scan certifies your own code against your Veracode policy.
 
This Jenkinsfile is a working example of how to integrate those scans. Treat it as a starting point, drop it into your repo, wire up the credentials, and adapt the build and branch logic to match your project.
 
Three scan types:
 
- **Agent-Based SCA** on every build (third-party dependency analysis)
- **Container/IaC/Secrets** on every build (IaC misconfigurations and secrets in source)
- **Policy Scan** on top-level branches only (production certification)
  
**Supported Technologies**: Java (Maven/Gradle/Ant), .NET, Node.js, Python, Go, PHP, Ruby, Scala, and more.
 
**Pipeline Type**: Declarative, designed for **Multibranch Pipeline** jobs (or equivalent: GitHub Organization, Bitbucket Team, GitLab Group). Top-level branch detection requires a multibranch context.
 
**Agent prerequisites** on `PATH`: `bash`, `curl`, `unzip`, `java` (JRE 8+), plus your build toolchain (Maven, Gradle, npm, dotnet, etc.). The build toolchain is required because the autopackager runs your build.
 
---
 
## Scanning Strategy
 
| Context | Veracode Product | Gate | Purpose |
|---------|------------------|------|---------|
| Every build | Agent-Based SCA | No | Dependency vulnerability analysis |
| Every build | Container/IaC/Secrets | No | IaC misconfigurations and secrets in source |
| `main` / `master` / `develop` | Policy Scan | Optional | Production certification |
 
Top-level branches are configurable via `TOP_LEVEL_BRANCHES` (regex, default `main|master|develop`). Change requests never trigger a Policy Scan.
 
---
 
## Pipeline Structure
 
```text
Trigger: SCM push / change request
        |
        v
Checkout  (always)  -> resolves IS_TOP_LEVEL + app profile name
        |
        v
Package Artifacts    when IS_TOP_LEVEL  -> autopackager -> stash verascan bundle
        |
        v
Veracode Security Scans (parallel)
   |                       |                        |
   v                       v                        v
Agent-Based SCA      Container/IaC/Secrets      Policy Scan
(every build,        (every build,              (top-level
 token-gated)         directory scan)            branch only)
```
 
Package runs only when a Policy Scan will follow, so feature branches and change requests run SCA and the Container/IaC/Secrets scan alone.
 
---
 
## Required Jenkins Credentials
 
Configure under **Manage Jenkins > Credentials > System > Global credentials**.
 
| Credential ID | Type | Required | Description |
|---------------|------|----------|-------------|
| `veracode-api-id` | Secret text | Yes | Veracode API ID |
| `veracode-api-key` | Secret text | Yes | Veracode API Key |
| `srcclr-api-token` | Secret text | No | SCA Agent token. If absent, the SCA stage skips gracefully |
 
The Container/IaC/Secrets scan authenticates with `veracode-api-id` and `veracode-api-key` (the Veracode CLI reads them), so it needs no extra credential.
 
### Optional Environment Variables
 
| Variable | Default | Description |
|----------|---------|-------------|
| `VERACODE_APP_NAME` | `org/repo` | Override the Veracode application profile name |
| `VERACODE_SOURCE_DIR` | repo root | Directory the autopackager and the directory scan use |
| `TOP_LEVEL_BRANCHES` | `main\|master\|develop` | Regex of branches that get a Policy Scan |
 
 

### How to Obtain Credentials

**To obtain a Service Account provide your Veracode contact with an e-mail of who will be in charge of this service account and we can create it for you.**
You will then receive a confirmation and next-steps for that account setup.

On the Service Account:
1. Sign in to the [Veracode Platform](https://analysiscenter.veracode.com)
2. Profile menu (top right) > **API Credentials** for the API ID and Key

On your human user account (Okta):
1. For SCA: **Agent-Based Scan > Workspace > Agents > Actions > Create > Jenkins > Create Agent & Generate Token**


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
 
Checks out source and resolves `IS_TOP_LEVEL` from `BRANCH_NAME`, `CHANGE_ID`, and `TOP_LEVEL_BRANCHES`. This single flag gates Package and Policy Scan.
 
### 2. Package Artifacts (top-level only)
 
Installs the Veracode CLI and runs the autopackager (`veracode package --source <dir> --output verascan --trust`), which builds and packages supported projects in a Veracode-ready format with no language-specific wiring. It validates that artifacts exist, then stashes `verascan/`, `artifact_list.txt`, and `app_name.txt` for the Policy Scan.
 
There is no separate build stage. The autopackager runs your build, so the build toolchain for your language must be installed on the agent, and `--trust` lets it run the detected build commands.
 
**Use your own build instead**: if you already build deployable artifacts in CI, run that build in this stage, then either point `VERACODE_SOURCE_DIR` at the build output or place the packaged artifact(s) directly into `verascan/` so the Policy Scan picks them up. Either way the artifact must meet Veracode packaging requirements. Generate exact, language-specific guidance with the [Veracode Packaging Cheat Sheet](https://docs.veracode.com/cheatsheet/) and [About autopackaging](https://docs.veracode.com/r/About_auto_packaging).
 
### 3. Agent-Based SCA (every build)
 
Binds `srcclr-api-token`, installs the SCA agent from `sca-downloads.veracode.com/ci.ps1`, and runs `scan --recursive --update-advisor` against the checked-out source. SCA scans source directly, so it does not need the packaged artifacts. If the token credential is missing, the stage logs a message and continues.
 
About 80% of vulnerabilities come from dependencies, so SCA runs on every build regardless of branch.
 
### 4. Container/IaC/Secrets Scan (every build)
 
Installs the Veracode CLI and runs `veracode scan --type directory --source <dir>` against the checked-out source. A directory (or repo) target runs the IaC scan for infrastructure-as-code misconfigurations plus secrets detection. It scans source directly, so no packaged artifact is needed. Results are written to `container_iac_secrets.json` and archived as a build artifact.
 
This stage is non-gating by default: the build does not fail on findings. To gate, remove the exit-code handling so a nonzero exit propagates (the Linux variant drops the trailing `|| echo ...`; the Windows variant replaces the `Write-Host` with `if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }`).
 
- **Secrets rules**: define detection rules under `container_scan:` `secret-rules:` in your `.veracode/veracode.yml` file; the scan applies them during the run. Supported keys include common providers such as AWS, Adobe, Alibaba, Asana, Atlassian, and asymmetric private keys.
- **Policy**: add `--policy policy-container-scan.rego` for pass/fail policy results. The Rego policy must be created for Container and IaC scans and downloaded first; a policy mixing SCA and Container/IaC rules cannot be downloaded.
- **Output formats**: `json` (Syft schema), `cyclonedx` (SBOM), or `table` (console summary) via `--format`.
- **No build needed**: unlike the Policy Scan, this does not consume the packaged artifact, which is why it runs on every build in parallel with SCA.
 
**Alternative placement**: to avoid a second CLI install, you can move the `veracode scan` call into the Package Artifacts stage right after the CLI install. That limits the scan to top-level branches, so the every-build parallel stage is preferred for earliest IaC/secrets feedback.
 
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
 
**Container/IaC/Secrets**: the JSON report is archived on the Jenkins build. To review in the Veracode Platform, set `analysis_on_platform: true` in `veracode.yml` before scanning.
 
**Policy Scan**: review findings, compliance status, and trends in the application profile at [analysiscenter.veracode.com](https://analysiscenter.veracode.com).
 
---
 
## Customization
 
### Change which branches get a Policy Scan
 
Set `TOP_LEVEL_BRANCHES`, for example `main|release/.*`.
 
### SCA scan options
 
Adjust the agent arguments in the SCA stage, for example add `--no-upload` for local-only runs, or set `$env:SRCCLR_REGION` for European or US Federal regions.
 
### SCA fail-on-policy gate (optional)
 
The SCA stage is non-gating by default. To fail the build on SCA policy violations, set the relevant `SRCCLR_*` threshold environment variables or use `srcclr --pass-fail` semantics, then remove the surrounding `try/catch` so failures propagate.
 
### Container/IaC/Secrets options
 
Adjust the `veracode scan` arguments in that stage: change `--format` (`json`, `cyclonedx`, `table`), add `--policy policy-container-scan.rego` for policy pass/fail, add `--verbose` for detailed output, or `--no-cache` to skip the cached vulnerability database. Configure secret detection rules in `.veracode/veracode.yml`.
 
### Use your own build instead of the autopackager
 
Run your existing build in the Package stage, then point `VERACODE_SOURCE_DIR` at the build output or drop the packaged artifact(s) into `verascan/`. Confirm the artifact meets packaging requirements with the [Veracode Packaging Cheat Sheet](https://docs.veracode.com/cheatsheet/).
 
---
 
## Troubleshooting
 
### Policy Scan stage does not run
 
`IS_TOP_LEVEL` resolved to `false`. Confirm `BRANCH_NAME` matches `TOP_LEVEL_BRANCHES` and that the build is not a change request (`CHANGE_ID` unset).
 
### `unstash` fails in Policy Scan
 
The Package stage failed before `stash` ran. Confirm the autopackager produced at least one `.war` / `.jar` / `.zip`, or add an explicit build step.
 
### Agent-Based SCA skipped
 
The `srcclr-api-token` credential is not configured, or the scan threw. Add the credential, or review the stage log for the captured error message.
 
### Container/IaC/Secrets scan finds nothing
 
The scan only reports IaC misconfigurations and secrets. If the repo has no IaC files (Terraform, CloudFormation, Kubernetes manifests, Dockerfiles, etc.) and no detectable secrets, an empty result is expected. Use `--verbose` to confirm files were inspected, and check `VERACODE_SOURCE_DIR` points at the right directory.
 
### Sandbox / Policy upload fails
 
Verify `veracode-api-id` and `veracode-api-key` are configured with upload permissions and that the application profile name has no invalid characters.
 
---
 
## Quick Start
 
1. Copy `Jenkinsfile` to the root of your repository
2. Create a **Multibranch Pipeline** job pointing at your repository
3. Add credentials: `veracode-api-id`, `veracode-api-key`, and optionally `srcclr-api-token`
4. (Optional) Set `VERACODE_APP_NAME`, `VERACODE_SOURCE_DIR`, `TOP_LEVEL_BRANCHES`
5. Push to a top-level branch to trigger Package and Policy Scan; push elsewhere for SCA and Container/IaC/Secrets only
6. Review results in the Veracode Platform and the archived `container_iac_secrets.json` artifact
---
 
## Resources
 
- [Veracode Documentation](https://docs.veracode.com)
- [Veracode CLI Installation](https://docs.veracode.com/r/Install_the_Veracode_CLI)
- [Packaging Cheat Sheet](https://docs.veracode.com/cheatsheet/)
- [About autopackaging](https://docs.veracode.com/r/About_auto_packaging)
- [Java API Wrapper](https://docs.veracode.com/r/c_about_wrappers)
- [Install the SCA agent with PowerShell](https://docs.veracode.com/r/Install_the_Veracode_SCA_Agent_with_PowerShell)
- [SCA Agent-Based Scans](https://docs.veracode.com/r/Agent_Based_Scans)
- [Veracode Container Security](https://docs.veracode.com/r/Veracode_Container_Security)
- [veracode scan command reference](https://docs.veracode.com/r/veracode_scan)
- [Run Container Security scans](https://docs.veracode.com/r/Run_Container_Security_scans)
- [Detect secrets in containers and IaC files](https://docs.veracode.com/r/Detect_custom_keywords_and_secrets_in_containers_and_IaC_files)
- [Jenkins Declarative Pipeline Syntax](https://www.jenkins.io/doc/book/pipeline/syntax/)
