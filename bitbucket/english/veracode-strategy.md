# Veracode Security Pipeline for Bitbucket Pipelines

Pipeline file: [`veracode-scans.yml`](./veracode-scans.yml). Rename it to `bitbucket-pipelines.yml` in your repository root.

**Supported technologies**: anything the [Veracode CLI autopackager](https://docs.veracode.com/r/About_auto_packaging) supports. The default image is `maven:3.9-eclipse-temurin-17`; change it if your project needs a different toolchain.

---

## Scanning Strategy

| Trigger | Custom pipeline | Veracode Product | Gate |
|---------|-----------------|------------------|------|
| Push to `main` | `veracode-policy` | Policy Scan | Platform policy |
| Push to `feature/**` | `veracode-feature` | Pipeline Scan | Fails on any flaw |
| PR with destination `main` | `veracode-pr` | Pipeline Scan | `Veracode Recommended Very High` policy |
| All of the above | (same pipeline) | Agent-Based SCA | Non-blocking |

Pipeline Scan exit codes ([docs](https://docs.veracode.com/r/Pipeline_Scan_Status_Codes)): `0` no flaws, `1-200` number of flaws that matched the criteria, `253-255` timeout or error. Any non-zero code fails the step.

---

## Pipeline Structure

The file uses Bitbucket [start conditions](https://support.atlassian.com/bitbucket-cloud/docs/pipeline-start-conditions/) (`triggers:`). Classic `pull-requests:` selectors only match the PR **source** branch; `pullrequest-push` with `BITBUCKET_PR_DESTINATION_BRANCH` filters on the destination natively, so PRs to other branches start nothing.

```text
triggers:
  repository-push   BITBUCKET_BRANCH == "main"               -> veracode-policy
  repository-push   glob(BITBUCKET_BRANCH, "feature/**")     -> veracode-feature
  pullrequest-push  BITBUCKET_PR_DESTINATION_BRANCH == "main" -> veracode-pr

each pipeline:  package  ->  parallel( sca, <scan step> )
```

Triggers can only start pipelines defined under `pipelines.custom`. Those pipelines also appear under **Run pipeline** for manual runs.

---

## Repository Variables

**Repository settings > Pipelines > Repository variables**

| Variable | Secured | Required | Description |
|----------|---------|----------|-------------|
| `VERACODE_API_ID` | Yes | Yes | Veracode API ID |
| `VERACODE_API_KEY` | Yes | Yes | Veracode API Key |
| `SRCCLR_API_TOKEN` | Yes | For SCA | Agent-based SCA token |
| `VERACODE_APP_NAME` | No | No | Application profile name. Defaults to `$BITBUCKET_REPO_FULL_NAME` (`workspace/repo`) |

API credentials: [Generate API credentials](https://docs.veracode.com/r/t_create_api_creds). SCA token: [Create an SCA agent](https://docs.veracode.com/r/t_sc_cli_agent).

---

## Step Details

### Package Artifacts

Installs the Veracode CLI, runs `veracode package --source . --output verascan --trust`, writes every `.war`, `.jar`, `.zip` to `artifact_list.txt`, and fails if none exist. `verascan/**` and `artifact_list.txt` are passed to later steps as artifacts.

### Agent-Based SCA

Runs `sca-downloads.veracode.com/ci.sh scan --recursive --update-advisor`. Errors are swallowed with `|| echo` so SCA never blocks. Remove that suffix to enforce SCA policy.

### Pipeline Scan (feature / PR gate)

Both steps share one script through a YAML anchor (`&pipeline_scan_loop`). Each step sets `GATE_POLICY` first:

| Step | `GATE_POLICY` | Result |
|------|---------------|--------|
| `Pipeline Scan (feature)` | empty | No fail criteria, any flaw fails the step |
| `Pipeline Scan (PR gate)` | `Veracode Recommended Very High` | Adds `--policy_name`, only policy-violating flaws fail |

Each artifact is scanned separately, the loop continues after a failure, and the step fails at the end if any artifact failed. Results are saved to `scan_results/<artifact>_results.json`.

### Policy Scan

Downloads the latest [Veracode Java API Wrapper](https://docs.veracode.com/r/c_about_wrappers) and runs `UploadAndScan` on the whole `verascan/` folder.

| Parameter | Value |
|-----------|-------|
| `-appname` | `VERACODE_APP_NAME` or `workspace/repo` |
| `-createprofile` / `-autoscan` | `true` |
| `-filepath` | `verascan` |
| `-version` | `<branch>-<build number>-<timestamp>` (unique on step reruns) |

---

## Behavior Notes

- **Default branch**: Bitbucket has no predefined default branch variable. If yours is not `main`, change both `main` conditions in `triggers:`.
- **Duplicate runs**: a push to `feature/x` with an open PR to `main` starts both `veracode-feature` and `veracode-pr`. This is expected. Use the `veracode-pr` result as the merge check.
- **Fork PRs**: secured variables are not available to forks, so scans will fail authentication there.
- **Script lines share a shell**: variables set in one `script:` item (for example `GATE_POLICY`) are available to later items in the same step.

---

## Customization

**Change the PR gate policy**: edit `GATE_POLICY` in the `Pipeline Scan (PR gate)` step. For a custom policy, download it with `--request_policy` and use `--policy_file` ([parameters](https://docs.veracode.com/r/r_pipeline_scan_commands)).

**Relax the feature gate**: add `--fail_on_severity "Very High, High"` to the `else` branch of the scan loop.

**Scan every non-default branch**: change the feature condition to:

```yaml
- condition: glob(BITBUCKET_BRANCH, "**") && BITBUCKET_BRANCH != "main"
```

**Use classic selectors instead**: move the three custom pipelines under `branches:` / `pull-requests:` and remove `triggers:`. You then need an in-script `BITBUCKET_PR_DESTINATION_BRANCH` check, as described in the [Atlassian KB](https://support.atlassian.com/bitbucket-cloud/kb/trigger-pipelines-on-pull-request-to-destination-branch/).

---

## Troubleshooting

| Symptom | Check |
|---------|-------|
| No pipeline starts | Branch name does not match a trigger condition, or conditions still say `main` while your default branch differs. |
| `No packaged artifacts found` | The image lacks your build toolchain. Change `image:` or add a build step before the autopackager. |
| Pipeline Scan exits `255` | Invalid credentials, network block to `api.veracode.com`, or unsupported artifact. |
| Pipeline Scan exits `253` or `254` | Scan timed out. Add `--timeout <minutes>` (max 60). |
| `UploadAndScan` rejected | A previous scan for the profile may still be running. Wait for it or cancel it in the Platform. |

---

## Resources

- [Bitbucket pipeline start conditions](https://support.atlassian.com/bitbucket-cloud/docs/pipeline-start-conditions/)
- [Pipeline Scan parameters](https://docs.veracode.com/r/r_pipeline_scan_commands)
- [Pipeline Scan status codes](https://docs.veracode.com/r/Pipeline_Scan_Status_Codes)
- [Java API Wrapper](https://docs.veracode.com/r/c_about_wrappers)
- [SCA CI script](https://docs.veracode.com/r/c_sc_ci_script)
