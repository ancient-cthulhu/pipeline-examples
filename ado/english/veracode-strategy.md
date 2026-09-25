# Veracode Security Pipeline for Azure Pipelines

Pipeline file: [`azure-pipelines.yml`](./azure-pipelines.yml). Place it in your repository root and create a pipeline from it.

**Supported technologies**: anything the [Veracode CLI autopackager](https://docs.veracode.com/r/About_auto_packaging) supports. The pool is `ubuntu-latest` (Java preinstalled).

---

## Scanning Strategy

| Trigger | Stage | Veracode Product | Gate |
|---------|-------|------------------|------|
| Push to `feature/*` | `PipelineScan` | Pipeline Scan | Fails on any flaw |
| Pull request to default branch | `PipelineScan` | Pipeline Scan | `Veracode Recommended Very High` policy |
| Push to default branch | `PolicyScan` | Policy Scan | Platform policy |
| All of the above | `SCA` | Agent-Based SCA | Non-blocking |

Pipeline Scan exit codes ([docs](https://docs.veracode.com/r/Pipeline_Scan_Status_Codes)): `0` no flaws, `1-200` number of flaws that matched the criteria, `253-255` timeout or error. Any non-zero code fails the stage.

---

## Pipeline Structure

```text
trigger: main, feature/*        pr: main (GitHub/Bitbucket only)
                 |
     +-----------+------------+
     |                        |
  Package                    SCA (dependsOn: [], parallel)
     |
     +-----------------------------+
     |                             |
  PipelineScan                 PolicyScan
  feature/* push: any flaw     push to DEFAULT_BRANCH:
  PR to DEFAULT_BRANCH: policy UploadAndScan
```

| Stage | Condition |
|-------|-----------|
| `Package`, `SCA` | Not a fork PR (`System.PullRequest.IsFork`) |
| `PipelineScan` | Non-PR build on `refs/heads/feature/*`, or PR whose `System.PullRequest.TargetBranch` is `DEFAULT_BRANCH` |
| `PolicyScan` | Non-PR build on `refs/heads/<DEFAULT_BRANCH>` |

The PR target check accepts both `main` and `refs/heads/main`, so it works regardless of the repository provider format.

---

## Setup

### 1. Variable group

**Pipelines > Library > + Variable group**, name `veracode-credentials`, then authorize it for the pipeline.

| Variable | Secret | Required | Description |
|----------|--------|----------|-------------|
| `VERACODE_API_ID` | Yes | Yes | Veracode API ID |
| `VERACODE_API_KEY` | Yes | Yes | Veracode API Key |
| `SRCCLR_API_TOKEN` | Yes | For SCA | Agent-based SCA token |
| `VERACODE_APP_NAME` | **No** | No | Application profile name. Defaults to `{org}/{project}/{repo}` |

Secret variables are not exposed to scripts automatically, so the pipeline maps them with `env:`. `VERACODE_APP_NAME` must stay non-secret because it is read from the environment.

API credentials: [Generate API credentials](https://docs.veracode.com/r/t_create_api_creds). SCA token: [Create an SCA agent](https://docs.veracode.com/r/t_sc_cli_agent).

### 2. Pull request trigger

| Repository | How PR runs are triggered |
|------------|---------------------------|
| Azure Repos Git | YAML `pr:` is ignored. Add a **Build Validation** branch policy on the default branch (**Project settings > Repositories > Branches > main > Branch policies**) and mark it Required. |
| GitHub, Bitbucket Cloud | The `pr:` block in the YAML applies. |

See [Troubleshoot pipeline triggers](https://learn.microsoft.com/en-us/azure/devops/pipelines/troubleshooting/troubleshoot-triggers).

### 3. Default branch

If your default branch is not `main`, change `DEFAULT_BRANCH` and both `branches.include` lists.

---

## Stage Details

### Package

1. Installs the Veracode CLI into `$(Agent.TempDirectory)` so the binary is not published.
2. Runs `veracode package --source $(Build.SourcesDirectory) --output verascan --trust` into `$(Build.ArtifactStagingDirectory)/veracode`.
3. Writes every `.war`, `.jar`, `.zip` to `artifact_list.txt` next to (not inside) `verascan/`, and fails if none exist.
4. Publishes the folder as the `veracode` pipeline artifact.

Keeping `artifact_list.txt` outside `verascan/` prevents it from being uploaded to the Veracode Platform during the policy scan.

### SCA

Runs `sca-downloads.veracode.com/ci.sh scan --recursive --update-advisor --appname "$APP_NAME"` in parallel with Package. The `--appname` value is the same application profile the policy scan uploads to, so agent-based findings land against the same profile. The stage recomputes it from `SYSTEM_COLLECTIONURI`, `SYSTEM_TEAMPROJECT` and `BUILD_REPOSITORY_NAME` so it stays `dependsOn: []`.

`--appname` makes the agent call the Veracode Platform, which needs HMAC credentials on top of `SRCCLR_API_TOKEN`. The agent reads them only from `VERACODE_API_KEY_ID` and `VERACODE_API_KEY_SECRET`, so the step maps your existing API ID and key onto those two names. Without them the scan fails with `HMAC authentication failed for license API` ([HMAC credentials](https://docs.veracode.com/r/HMAC_credentials)). The profile must also have at least one completed static scan before the agent can link to it ([SCA agent commands](https://docs.veracode.com/r/SCA_agent_commands)). Errors are swallowed with `|| echo`. Remove that suffix to enforce SCA policy.

### PipelineScan

- Validates that the credentials are real values, not unexpanded `$(VAR)` macros.
- Scans each artifact separately, continues after a failure, and fails the stage at the end if any artifact failed.
- `Build.Reason == PullRequest` adds `--policy_name "$(PR_GATE_POLICY)"`; feature pushes add no gate arguments.
- Publishes `scan_results/` as `veracode-pipeline-scan-results-<attempt>` even on failure.

### PolicyScan

Downloads the latest [Veracode Java API Wrapper](https://docs.veracode.com/r/c_about_wrappers) and runs `UploadAndScan` on `verascan/`.

| Parameter | Value |
|-----------|-------|
| `-appname` | `VERACODE_APP_NAME` or `{org}/{project}/{repo}` |
| `-createprofile` / `-autoscan` | `true` |
| `-filepath` | `verascan` |
| `-version` | `$(Build.BuildNumber)-$(System.JobAttempt)` (unique on reruns) |

---

## Behavior Notes

- **Duplicate runs (GitHub/Bitbucket repos)**: a push to `feature/x` with an open PR can start both a CI run and a PR run. Use the PR run as the required check.
- **Fork PRs**: skipped, because secrets are not exposed to fork builds by default.
- **Manual runs**: a manual run on `feature/*` behaves like a push (Pipeline Scan, any flaw fails). On the default branch it runs the policy scan.

---

## Customization

**Change the PR gate policy**: edit `PR_GATE_POLICY`. For a custom policy, download it with `--request_policy` and use `--policy_file` ([parameters](https://docs.veracode.com/r/r_pipeline_scan_commands)).

**Relax the feature gate**: set `GATE_ARGS=(--fail_on_severity "Very High, High")` in the `else` branch.

**Scan every non-default branch**: set `trigger.branches.include` to `'*'` and change the feature condition to:

```yaml
ne(variables['Build.SourceBranch'], format('refs/heads/{0}', variables['DEFAULT_BRANCH']))
```

---

## Troubleshooting

| Symptom | Check |
|---------|-------|
| `VERACODE_API_ID is not set` | Variable group not linked or not authorized, or variable name mismatch. |
| PR builds never run (Azure Repos) | No Build Validation branch policy. `pr:` does not apply to Azure Repos. |
| `PipelineScan` skipped on PR | PR target is not `DEFAULT_BRANCH`. |
| Custom app name ignored | `VERACODE_APP_NAME` is marked secret, so it is not in the script environment. Make it non-secret. |
| Pipeline Scan exits `255` | Invalid credentials, network block to `api.veracode.com`, or unsupported artifact. |
| Pipeline Scan exits `253` or `254` | Scan timed out. Add `--timeout <minutes>` (max 60). |
| `UploadAndScan` rejected | A previous scan for the profile may still be running. |

---

## Resources

- [Azure Pipelines triggers](https://learn.microsoft.com/en-us/azure/devops/pipelines/build/triggers)
- [Build Azure Repos Git repositories](https://learn.microsoft.com/en-us/azure/devops/pipelines/repos/azure-repos-git)
- [Pipeline Scan parameters](https://docs.veracode.com/r/r_pipeline_scan_commands)
- [Java API Wrapper](https://docs.veracode.com/r/c_about_wrappers)
- [SCA CI script](https://docs.veracode.com/r/c_sc_ci_script)
