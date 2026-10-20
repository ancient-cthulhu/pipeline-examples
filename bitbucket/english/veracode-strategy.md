# Veracode Security Pipeline for Bitbucket Pipelines

Pipeline file: [`veracode-scans.yml`](./veracode-scans.yml). Rename it to `bitbucket-pipelines.yml` in your repository root.

**Supported technologies**: anything the [Veracode CLI autopackager](https://docs.veracode.com/r/About_auto_packaging) supports. The default image is `maven:3.9-eclipse-temurin-17`; change it if your project needs a different toolchain.

---

## Scanning Strategy

| Trigger | Pipeline | Veracode Product | Gate |
|---------|----------|------------------|------|
| Push to `main` | `branches: main` | Policy Scan | Platform policy |
| Push to any non-default branch | `default` | Pipeline Scan | Reports any flaw, non-blocking (step shows Failed, Ignored) |
| PR with destination `main` | `veracode-pr` | Pipeline Scan | `Veracode Recommended Very High` policy |
| All of the above | (same pipeline) | IaC scan (Veracode CLI) | Non-blocking |
| All of the above | (same pipeline) | Agent-Based SCA | Non-blocking |

Pipeline Scan exit codes ([docs](https://docs.veracode.com/r/Pipeline_Scan_Status_Codes)): `0` no flaws, `1-200` number of flaws that matched the criteria, `253-255` timeout or error. Any non-zero code fails the step; on the branch step that failure is ignored (see below).

---

## Pipeline Structure

Pushes use the classic selectors, which Bitbucket resolves most-specific-first and which never match tag pushes. Pull requests use a [start condition](https://support.atlassian.com/bitbucket-cloud/docs/pipeline-start-conditions/) (`triggers:`), because classic `pull-requests:` selectors match only the PR **source** branch, while `pullrequest-push` with `BITBUCKET_PR_DESTINATION_BRANCH` filters on the destination.

```text
pipelines:
  branches: main   -> package -> parallel( sca, Policy Scan )
  default          -> package -> parallel( sca, Pipeline Scan )   every other branch, no tags
  custom:  veracode-pr -> package -> parallel( sca, Pipeline Scan, policy gate )

triggers:
  pullrequest-push  BITBUCKET_PR_DESTINATION_BRANCH == "main" -> veracode-pr
```

An exact branch name beats `default`, so `main` gets the policy scan and every other branch gets the pipeline scan. `default` [excludes tag pushes](https://support.atlassian.com/bitbucket-cloud/docs/pipeline-start-conditions/). Triggers can only start pipelines defined under `pipelines.custom`, which is why the PR pipeline lives there; it also appears under **Run pipeline** for manual runs.

---

## Repository Variables

**Repository settings > Pipelines > Repository variables**

| Variable | Secured | Required | Description |
|----------|---------|----------|-------------|
| `VERACODE_API_ID` | Yes | Yes | Veracode API ID |
| `VERACODE_API_KEY` | Yes | Yes | Veracode API Key |
| `SRCCLR_API_TOKEN` | Yes | For SCA | Agent-based SCA token |
| `VERACODE_APP_NAME` | No | No | Application profile name. Defaults to `$BITBUCKET_REPO_FULL_NAME` (`workspace/repo`) |
| `VERACODE_BUSINESS_CRITICALITY` | No | No | `VeryHigh`, `High`, `Medium`, `Low`, `VeryLow`. Used only when the profile is created. Defaults to `VeryHigh` |

API credentials: [Generate API credentials](https://docs.veracode.com/r/t_create_api_creds). SCA token: [Create an SCA agent](https://docs.veracode.com/r/t_sc_cli_agent).

---

## Step Details

### Package Artifacts

Installs the Veracode CLI into `/opt/veracode-cli` (outside the cloned repo, so the CLI binary is not part of the scanned source), runs `veracode package --source . --output verascan --trust`, writes every `.war`, `.jar`, `.ear`, `.zip`, `.apk`, `.ipa` (the [autopackager output types](https://docs.veracode.com/r/About_auto_packaging)) to `artifact_list.txt`, and fails if none exist. `verascan/**` and `artifact_list.txt` are passed to later steps as artifacts.

### Agent-Based SCA

Runs `sca-downloads.veracode.com/ci.sh scan --recursive --update-advisor`. Errors are swallowed with `|| echo` so SCA never blocks. Remove that suffix to enforce SCA policy.

### IaC scan

Runs inside the `Package Artifacts` step, right after the CLI is installed, so it scans the checked-out files rather than a build output:

```bash
veracode scan --source . --type directory --format table
```

This is the Veracode CLI [container, IaC, and secrets scan](https://docs.veracode.com/r/veracode_scan) pointed at the working directory. It authenticates with `VERACODE_API_KEY_ID` and `VERACODE_API_KEY_SECRET`, derived from the existing repository variables. Errors are swallowed with `|| echo`, so it never blocks.

### Pipeline Scan (branch / PR gate)

Both steps share one script through a YAML anchor (`&pipeline_scan_loop`). Each step sets `GATE_POLICY` first:

| Step | `GATE_POLICY` | Result |
|------|---------------|--------|
| `Pipeline Scan (branch)` | empty | No fail criteria, any flaw fails the step. `on-fail: strategy: ignore` keeps the build green |
| `Pipeline Scan (PR gate)` | `Veracode Recommended Very High` | Adds `--policy_name`, only policy-violating flaws fail |

Each artifact is scanned separately, the loop continues after a failure, and the step fails at the end if any artifact failed. Results are saved to `scan_results/<artifact>_results.json` and uploaded as a scoped artifact with `capture-on: always`, so they are kept when the scan fails (the default only keeps artifacts of successful steps).

### Policy Scan

Downloads the latest [Veracode Java API Wrapper](https://docs.veracode.com/r/c_about_wrappers) and runs `UploadAndScan` on the whole `verascan/` folder.

| Parameter | Value |
|-----------|-------|
| `-appname` | `VERACODE_APP_NAME` or `workspace/repo` |
| `-createprofile` | `true` |
| `-criticality` | `VERACODE_BUSINESS_CRITICALITY` or `VeryHigh`. [Required with `-createprofile`](https://docs.veracode.com/r/r_uploadandscan) |
| `-filepath` | `verascan` |
| `-version` | `<branch>-<build number>-<timestamp>` (unique on step reruns) |

---

## Behavior Notes

- **Default branch**: Bitbucket has no predefined default branch variable. If yours is not `main`, rename the `main:` selector under `branches:` and update the condition in `triggers:`.
- **Duplicate runs**: a push to a branch with an open PR to `main` starts both the `default` pipeline and `veracode-pr`, because triggers are cumulative with classic selectors. Both report a build status on the same commit, and Bitbucket's merge check ("Minimum number of successful builds for the last commit with no failed builds") counts every build on that commit. That is why the branch Pipeline Scan is non-blocking: otherwise any flaw would block the PR even when the `veracode-pr` policy gate passes. To enforce the gate, enable that merge check plus **Prevent a merge with unresolved merge checks** (Premium) under **Settings > Workflow > Branch restrictions**.
- **Rate limit**: the duplicate runs scan each artifact twice. One Veracode account can start at most six Pipeline Scans per minute (HTTP 429 above that).
- **Fork PRs**: Bitbucket does not run the parent repository's pipelines for pull requests from forks ([BCLOUD-13162](https://jira.atlassian.com/browse/BCLOUD-13162)), so those PRs get no Veracode scan.
- **Script lines share a shell**: variables set in one `script:` item (for example `GATE_POLICY`) are available to later items in the same step.

---

## Customization

**Change the PR gate policy**: edit `GATE_POLICY` in the `Pipeline Scan (PR gate)` step. For a custom policy, download it with `--request_policy` and use `--policy_file` ([parameters](https://docs.veracode.com/r/r_pipeline_scan_commands)).

**Relax the branch gate**: add `--fail_on_severity "Very High, High"` to the `else` branch of the scan loop.

**Make the branch scan blocking**: remove `on-fail` from `Pipeline Scan (branch)`. Branch flaws then fail the build, which also blocks any open PR from that branch.

**Limit which branches are scanned**: replace `default:` with explicit selectors under `branches:`, for example `'feature/**'`. Branches that match nothing then run no pipeline.

**Scan tag pushes too**: add a `tags:` selector with the steps you want. `default` never runs for tags.

**Avoid the trigger entirely**: move the PR pipeline under `pull-requests:` and delete `triggers:`. That selector matches the PR source branch only, so you then need an in-script `BITBUCKET_PR_DESTINATION_BRANCH` check, as described in the [Atlassian KB](https://support.atlassian.com/bitbucket-cloud/kb/trigger-pipelines-on-pull-request-to-destination-branch/).

---

## Troubleshooting

| Symptom | Check |
|---------|-------|
| No pipeline starts | Your default branch is not `main`, so the `main:` selector and the PR condition still point at the wrong branch. |
| `No packaged artifacts found` | The image lacks your build toolchain. Change `image:` or add a build step before the autopackager. |
| Pipeline Scan exits `255` | Invalid credentials, network block to `api.veracode.com`, or unsupported artifact. |
| Pipeline Scan exits `254` | Scan exceeded the default 60-minute limit (the maximum). Split the artifact or exclude modules. |
| Pipeline Scan exits `253` | Scan exceeded your `--timeout` value. Raise it (max 60). |
| `UploadAndScan` rejected | A previous scan for the profile may still be running. Wait for it or cancel it in the Platform. |

---

## Resources

- [Bitbucket pipeline start conditions](https://support.atlassian.com/bitbucket-cloud/docs/pipeline-start-conditions/)
- [Pipeline Scan parameters](https://docs.veracode.com/r/r_pipeline_scan_commands)
- [Pipeline Scan status codes](https://docs.veracode.com/r/Pipeline_Scan_Status_Codes)
- [Java API Wrapper](https://docs.veracode.com/r/c_about_wrappers)
- [SCA CI script](https://docs.veracode.com/r/c_sc_ci_script)
