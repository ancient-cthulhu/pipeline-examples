# Veracode Security Pipeline for Jenkins

Pipeline files:

| File | Agent | Shell |
|------|-------|-------|
| [`Jenkinsfile-linux`](./Jenkinsfile-linux) | label `linux` | POSIX `sh` (works with dash) |
| [`Jenkinsfile-win`](./Jenkinsfile-win) | label `windows` | Windows PowerShell 5.1+ |

Rename the one you use to `Jenkinsfile` in your repository root and create a **Multibranch Pipeline** job. Both files are tested against [verademo](https://github.com/veracode/verademo) (Maven, `app/pom.xml`).

---

## Scanning Strategy

| Trigger | Stage | Veracode Product | Gate |
|---------|-------|------------------|------|
| Push to `feature/*` | `Pipeline Scan` | Pipeline Scan | Fails on any flaw |
| Change request (PR) to default branch | `Pipeline Scan` | Pipeline Scan | `Veracode Recommended Very High` policy |
| Push to default branch | `Policy Scan` | Policy Scan | Platform policy |
| All of the above | `Agent-Based SCA` | Agent-Based SCA | Non-blocking |

Pipeline Scan exit codes ([docs](https://docs.veracode.com/r/Pipeline_Scan_Status_Codes)): `0` no flaws, `1-200` number of flaws that matched the criteria, `253-255` timeout or error. Any non-zero code fails the stage.

---

## Pipeline Structure

```text
Build (Maven) -> Package Artifacts (stash "verascan")
                         |
         +---------------+----------------+
         |               |                |
  Agent-Based SCA   Pipeline Scan     Policy Scan
  (always)          feature/* push    push to DEFAULT_BRANCH
                    or CR to default
```

| Stage | `when` condition |
|-------|------------------|
| `Pipeline Scan` | `!CHANGE_ID && BRANCH_NAME.startsWith('feature/')`, or `CHANGE_ID && CHANGE_TARGET == DEFAULT_BRANCH` |
| `Policy Scan` | `!CHANGE_ID && BRANCH_NAME == DEFAULT_BRANCH` |

`BRANCH_NAME`, `CHANGE_ID`, and `CHANGE_TARGET` are set by Multibranch Pipeline jobs. In a plain Pipeline job they are empty, so only SCA runs.

Scan stages `unstash` into their own subdirectory (`pipeline-scan/`, `policy-scan/`) so parallel stages on the same agent do not overwrite each other.

---

## Setup

### Credentials

**Manage Jenkins > Credentials**, kind **Secret text**:

| ID | Required | Description |
|----|----------|-------------|
| `veracode-api-id` | Yes | Veracode API ID |
| `veracode-api-key` | Yes | Veracode API Key |
| `srcclr-api-token` | No | Agent-based SCA token. If missing, SCA is skipped |

API credentials: [Generate API credentials](https://docs.veracode.com/r/t_create_api_creds). SCA token: [Create an SCA agent](https://docs.veracode.com/r/t_sc_cli_agent).

### Optional environment variables (job or folder)

| Variable | Default | Description |
|----------|---------|-------------|
| `VERACODE_APP_NAME` | Job path without the branch segment | Application profile name |
| `MAVEN_POM_PATH` | `app/pom.xml` | Maven pom. Use `pom.xml` for repo-root projects |
| `VERACODE_SOURCE_DIR` | `app` | Autopackager source directory. Use `.` for repo-root projects |

`DEFAULT_BRANCH` and `PR_GATE_POLICY` are defined in the `environment` block of the Jenkinsfile.

### Plugins

Pipeline (Declarative), Pipeline: Multibranch, Credentials Binding, Workspace Cleanup (`cleanWs`), Timestamper (`timestamps()`), plus the branch source plugin for your SCM (GitHub, Bitbucket, GitLab).

### Agent tools

| Agent | Required on PATH |
|-------|------------------|
| Linux | `java` 8+, `mvn`, `curl`, `unzip`, GNU `grep` (for `-P`) |
| Windows | `java` 8+, `mvn`, Windows PowerShell 5.1+ |

---

## Stage Details

### Build (Maven)

`mvn -B -f $MAVEN_POM_PATH clean package -DskipTests`. Remove this stage if the autopackager alone handles your project.

### Package Artifacts

Resolves `APP_NAME`, installs the Veracode CLI, runs `veracode package --trust`, writes relative artifact paths to `artifact_list.txt`, fails if none exist, and stashes `verascan/**` plus the list.

Multibranch `JOB_NAME` is `<folder>/<repo>/<branch>`. The last segment is removed so every branch maps to the same application profile.

### Agent-Based SCA

| Agent | Command |
|-------|---------|
| Linux | `curl -sSL https://sca-downloads.veracode.com/ci.sh \| sh -s -- scan --recursive --update-advisor --appname "$APP_NAME"` |
| Windows | Downloads `https://sca-downloads.veracode.com/ci.ps1` and runs it with `-ArgumentList scan, --recursive, --update-advisor, --appname, $env:APP_NAME` ([docs](https://docs.veracode.com/r/t_sc_agent_proxy)) |

Non-blocking on both: Linux swallows errors with `|| echo`, Windows uses `powershell(returnStatus: true)`.

`APP_NAME` is the value resolved in the Package stage, so agent-based findings land against the same application profile the policy scan uploads to.

`--appname` makes the agent call the Veracode Platform, which needs HMAC credentials on top of `SRCCLR_API_TOKEN`. The agent reads them only from `VERACODE_API_KEY_ID` and `VERACODE_API_KEY_SECRET`, so the step maps your existing API ID and key onto those two names. Without them the scan fails with `HMAC authentication failed for license API` ([HMAC credentials](https://docs.veracode.com/r/HMAC_credentials)). The profile must also have at least one completed static scan before the agent can link to it ([SCA agent commands](https://docs.veracode.com/r/SCA_agent_commands)).

### Pipeline Scan

Each artifact is scanned separately, the loop continues after a failure, and the stage fails at the end if any artifact failed. Change requests add `--policy_name "$PR_GATE_POLICY"`; feature pushes add no gate arguments. `scan_results/` is archived even on failure.

### Policy Scan

Downloads the latest [Veracode Java API Wrapper](https://docs.veracode.com/r/c_about_wrappers) and runs `UploadAndScan` on `verascan/` with version `<branch>-<BUILD_NUMBER>`.

---

## Behavior Notes

- **Duplicate builds**: with a branch source that discovers both branches and PRs, a push to `feature/x` with an open PR builds both `feature/x` (any-flaw gate) and `PR-n` (policy gate). Use the PR build as the required status check.
- **`disableConcurrentBuilds()`** applies per branch job, so a policy scan on `main` never overlaps with another `main` build.

---

## Customization

**Change the PR gate policy**: edit `PR_GATE_POLICY`. For a custom policy, download it with `--request_policy` and use `--policy_file` ([parameters](https://docs.veracode.com/r/r_pipeline_scan_commands)).

**Relax the feature gate**: add `--fail_on_severity "Very High, High"` to the no-policy branch of the scan command.

**Scan every non-default branch**: change the feature expression to `!env.CHANGE_ID && env.BRANCH_NAME != env.DEFAULT_BRANCH`.

---

## Troubleshooting

| Symptom | Check |
|---------|-------|
| Scan stages always skipped | Job is not Multibranch, so `BRANCH_NAME` is empty. |
| `No packaged artifacts found` | `VERACODE_SOURCE_DIR` or `MAVEN_POM_PATH` does not match your layout. |
| `grep: invalid option -- 'P'` (Linux) | Agent uses BusyBox or BSD grep. Install GNU grep. |
| Pipeline Scan exits `255` | Invalid credentials, network block to `api.veracode.com`, or unsupported artifact. |
| Pipeline Scan exits `253` or `254` | Scan timed out. Add `--timeout <minutes>` (max 60). |
| `UploadAndScan` rejected | A previous scan for the profile may still be running. |

---

## Resources

- [Jenkins Pipeline syntax: when](https://www.jenkins.io/doc/book/pipeline/syntax/#when)
- [Pipeline Scan parameters](https://docs.veracode.com/r/r_pipeline_scan_commands)
- [Java API Wrapper](https://docs.veracode.com/r/c_about_wrappers)
- [SCA CI script](https://docs.veracode.com/r/c_sc_ci_script)
