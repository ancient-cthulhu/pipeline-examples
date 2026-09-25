# Veracode Security Pipeline for GitLab CI/CD

Pipeline file: [`.gitlab-ci.yml`](./.gitlab-ci.yml). Copy it to the root of your project.

**Supported technologies**: anything the [Veracode CLI autopackager](https://docs.veracode.com/r/About_auto_packaging) supports (Java, .NET, JavaScript/TypeScript, Python, Go, PHP, Ruby, Scala, Kotlin, and more).

---

## Scanning Strategy

| Trigger | Veracode Product | Gate | Purpose |
|---------|------------------|------|---------|
| Push to `feature/**` | Pipeline Scan | Fails on any flaw | Developer feedback on every push |
| Merge request to default branch | Pipeline Scan | `Veracode Recommended Very High` policy | Prove the MR is safe to merge |
| Push to default branch | Policy Scan | Platform policy | Compliance record for the application profile |
| All of the above | Agent-Based SCA | Non-blocking (`allow_failure: true`) | Third-party dependency analysis |

Pipeline Scan exit codes ([docs](https://docs.veracode.com/r/Pipeline_Scan_Status_Codes)): `0` no flaws, `1-200` number of flaws that matched the criteria, `253-255` timeout or error. Any non-zero code fails the job.

---

## Pipeline Structure

```text
workflow:rules  (MR to default | push default | push feature/**)
                    |
   stage: package   +-- package (CLI autopackager, dotenv APP_NAME)
                    +-- sca     (needs: [], allow_failure)
                    |
   stage: scan      +-- pipeline-scan  push feature/**: any flaw fails
                    |                  MR: policy gate
                    +-- policy-scan    push default branch: UploadAndScan
```

| Job | Runs when |
|-----|-----------|
| `package` | Every pipeline allowed by `workflow:rules` |
| `sca` | Every pipeline allowed by `workflow:rules` |
| `pipeline-scan` | MR event from the same project, or push to a non-default branch |
| `policy-scan` | Push to `$CI_DEFAULT_BRANCH` |

`workflow:rules` in order:

1. MR pipeline whose target is `$CI_DEFAULT_BRANCH`: run.
2. Any other MR pipeline: never.
3. Push to `$CI_DEFAULT_BRANCH`: run. Evaluated before rule 4 so the policy scan still runs if the default branch is the source of an open MR.
4. Branch push with an open MR (`$CI_OPEN_MERGE_REQUESTS`): never, the MR pipeline covers it. Note: this also applies when the open MR targets a non-default branch, so that feature branch gets no pipeline.
5. Push to `feature/*` (regex `^feature\/`, matches nested names): run.

---

## Required CI/CD Variables

**Settings > CI/CD > Variables**

| Variable | Required | Flags | Description |
|----------|----------|-------|-------------|
| `VERACODE_API_ID` | Yes | Masked | Veracode API ID |
| `VERACODE_API_KEY` | Yes | Masked | Veracode API Key |
| `SRCCLR_API_TOKEN` | For SCA | Masked | Agent-based SCA token. SCA is skipped if unset |
| `VERACODE_APP_NAME` | No | | Application profile name. Defaults to `$CI_PROJECT_PATH` |

**Do not mark these variables Protected** unless `feature/*` is a protected branch pattern. Protected variables are only exposed to pipelines on protected branches and tags, so feature and MR pipelines would run with empty credentials.

API credentials: [Generate API credentials](https://docs.veracode.com/r/t_create_api_creds). SCA token: [Create an SCA agent](https://docs.veracode.com/r/t_sc_cli_agent).

---

## Job Details

### package

Image `ubuntu:22.04`. Installs the Veracode CLI, runs `veracode package --source . --output verascan --trust`, writes every `.war`, `.jar`, `.zip` to `artifact_list.txt`, and fails if none exist. Exports `APP_NAME` through a `dotenv` report so downstream jobs receive it.

### sca

Image `eclipse-temurin:17-jdk`, `needs: []` so it starts immediately. Runs `sca-downloads.veracode.com/ci.sh scan --recursive --update-advisor --appname "$APP_NAME"`, reusing the shared `&set_app_name` anchor. The `--appname` value is the same application profile the policy scan uploads to, so agent-based findings land against the same profile. `allow_failure: true` shows SCA failures as warnings without blocking. Remove it to enforce SCA policy.

`--appname` makes the agent call the Veracode Platform, which needs HMAC credentials on top of `SRCCLR_API_TOKEN`. The agent reads them only from `VERACODE_API_KEY_ID` and `VERACODE_API_KEY_SECRET`, so the step maps your existing API ID and key onto those two names. Without them the scan fails with `HMAC authentication failed for license API` ([HMAC credentials](https://docs.veracode.com/r/HMAC_credentials)). The profile must also have at least one completed static scan before the agent can link to it ([SCA agent commands](https://docs.veracode.com/r/SCA_agent_commands)).

### pipeline-scan

Scans each artifact separately, continues after failures, then fails the job if any artifact failed.

| Pipeline source | Arguments added | Result |
|-----------------|-----------------|--------|
| `push` to `feature/**` | none | Every flaw counts, any finding fails the job |
| `merge_request_event` | `--policy_name "$PR_GATE_POLICY"` | Only policy-violating flaws fail the job |

Results are saved as job artifacts (`scan_results/<artifact>_results.json`, `when: always`, 1 month).

### policy-scan

Downloads the latest [Veracode Java API Wrapper](https://docs.veracode.com/r/c_about_wrappers) from Maven Central and runs `UploadAndScan` on the whole `verascan/` folder.

| Parameter | Value |
|-----------|-------|
| `-appname` | `$APP_NAME` from `package` |
| `-createprofile` | `true` |
| `-autoscan` | `true` |
| `-filepath` | `verascan` |
| `-version` | `<branch>-<pipeline_iid>-<job_id>` (unique per retry) |

The job ends when the upload is accepted. Results appear in the Veracode Platform when the scan completes.

---

## Behavior Notes

- **Default branch**: all rules use `$CI_DEFAULT_BRANCH`. No edits needed if your default is `master` or `develop`.
- **MRs to other branches**: rule 2 blocks them. If a feature branch has an open MR to a non-default branch, rule 3 also blocks its push pipelines, so that branch gets no scan. Remove rule 3 if that matters for your flow (feature pushes and MRs will then both run).
- **Fork MRs**: `pipeline-scan` requires `$CI_MERGE_REQUEST_SOURCE_PROJECT_ID == $CI_MERGE_REQUEST_PROJECT_ID`. Fork MR pipelines run in the fork without your variables.
- **Merge request approvals**: to block merges on a failed gate, enable **Settings > Merge requests > Pipelines must succeed**.

---

## Customization

**Change the MR gate policy**: edit `PR_GATE_POLICY` under `variables:`. For custom policies, download with `--request_policy` and pass `--policy_file` ([parameters](https://docs.veracode.com/r/r_pipeline_scan_commands)).

**Relax the feature gate**: add `--fail_on_severity "Very High, High"` to the `else` branch scanner call.

**GitLab Security Dashboard**: add `--gl_vulnerability_generation true` to the scanner call and publish `veracode_gitlab_vulnerabilities.json` under `artifacts:reports:sast` ([Veracode example](https://docs.veracode.com/r/Pipeline_Scan_Example_for_Using_GitLab_and_Gradle_with_Automatic_Vulnerability_Generation_Using_a_Built_in_Policy)). Rename the file per artifact inside the loop.

**Scan all non-default branches**: replace the last `workflow:rules` entry with `- if: $CI_PIPELINE_SOURCE == "push" && $CI_COMMIT_BRANCH`.

**Add a build step**: build in `package` before the autopackager, or replace it and write your artifact paths to `artifact_list.txt`.

---

## Troubleshooting

| Symptom | Check |
|---------|-------|
| Credentials empty on feature/MR pipelines | Variables marked Protected. Unprotect them. |
| No pipeline on push | Branch does not match `workflow:rules`, or an MR is open (the MR pipeline runs instead). |
| Duplicate branch and MR pipelines | Known GitLab issue ([#555601](https://gitlab.com/gitlab-org/gitlab/-/issues/555601)): `$CI_OPEN_MERGE_REQUESTS` is only evaluated in `workflow:rules` when a job references it. Keep the `echo` line in `package`. |
| `No packaged artifacts found` | Run `veracode package --source . --output verascan --trust` locally. Add a build step if needed. |
| Pipeline Scan exits `255` | Invalid credentials, blocked network to `api.veracode.com`, or unsupported artifact. |
| Pipeline Scan exits `253` or `254` | Timeout. Add `--timeout <minutes>` (max 60). |
| `APP_NAME` empty in `policy-scan` | `package` failed to write the dotenv report, or `needs:` was changed to `artifacts: false`. |
| `UploadAndScan` rejected | A previous scan for the profile may still be running. Verify API user upload permissions. |

---

## Resources

- [Pipeline Scan parameters](https://docs.veracode.com/r/r_pipeline_scan_commands)
- [Pipeline Scan status codes](https://docs.veracode.com/r/Pipeline_Scan_Status_Codes)
- [Veracode CLI](https://docs.veracode.com/r/Install_the_Veracode_CLI)
- [Java API Wrapper](https://docs.veracode.com/r/c_about_wrappers)
- [SCA CI script](https://docs.veracode.com/r/c_sc_ci_script)
- [GitLab: workflow rules](https://docs.gitlab.com/ci/yaml/workflow/)
- [GitLab: predefined variables](https://docs.gitlab.com/ci/variables/predefined_variables/)
