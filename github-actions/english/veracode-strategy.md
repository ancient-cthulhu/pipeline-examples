# Veracode Security Pipeline for GitHub Actions

Workflow file: [`veracode-scans.yml`](./veracode-scans.yml). Copy it to `.github/workflows/` in your repository.

**Supported technologies**: anything the [Veracode CLI autopackager](https://docs.veracode.com/r/About_auto_packaging) supports (Java, .NET, JavaScript/TypeScript, Python, Go, PHP, Ruby, Scala, Kotlin, and more).

---

## Scanning Strategy

| Trigger | Veracode Product | Gate | Purpose |
|---------|------------------|------|---------|
| Push to `feature/**` | Pipeline Scan | Fails on any flaw | Developer feedback on every push |
| Pull request to default branch | Pipeline Scan | `Veracode Recommended Very High` policy | Prove the PR is safe to merge |
| Push to default branch | Policy Scan | Platform policy | Compliance record for the application profile |
| All of the above | Agent-Based SCA | Non-blocking | Third-party dependency analysis |

Pipeline Scan exit codes ([docs](https://docs.veracode.com/r/Pipeline_Scan_Status_Codes)): `0` no flaws, `1-200` number of flaws that matched the criteria, `253-255` timeout or error. The workflow treats any non-zero code as a failure.

---

## Workflow Structure

```text
on: push (main, feature/**) | pull_request (to main)
                    |
        +-----------+-----------+
        |                       |
     package                   sca
  (CLI autopackager)      (non-blocking)
        |
        +---------------------------+
        |                           |
  pipeline-scan                policy-scan
  push feature/**: any flaw    push default branch:
  PR: policy gate              UploadAndScan
```

| Job | Runs when |
|-----|-----------|
| `package` | Always (except fork PRs) |
| `sca` | Always (except fork PRs) |
| `pipeline-scan` | `pull_request`, or `push` to a non-default branch |
| `policy-scan` | `push` to `github.event.repository.default_branch` |

---

## Required Secrets

**Settings > Secrets and variables > Actions**

| Secret | Required | Description |
|--------|----------|-------------|
| `VERACODE_API_ID` | Yes | Veracode API ID |
| `VERACODE_API_KEY` | Yes | Veracode API Key |
| `SRCCLR_API_TOKEN` | For SCA | Agent-based SCA token |
| `VERACODE_APP_NAME` | No | Application profile name. Defaults to `github.repository` (`org/repo`) |

API credentials: [Generate API credentials](https://docs.veracode.com/r/t_create_api_creds). SCA token: [Create an SCA agent](https://docs.veracode.com/r/t_sc_cli_agent).

---

## Job Details

### package

1. Checks out the code.
2. Installs the Veracode CLI (`curl -fsS https://tools.veracode.com/veracode-cli/install | sh`).
3. Runs `veracode package --source . --output verascan --trust`.
4. Writes every `.war`, `.jar`, `.zip` found to `artifact_list.txt` and fails if none exist.
5. Uploads `verascan/` and `artifact_list.txt` as the `verascan` workflow artifact.

### sca

Runs `sca-downloads.veracode.com/ci.sh scan --recursive --update-advisor --appname "$APP_NAME"`. The `--appname` value is the same application profile the policy scan uploads to, so agent-based findings land against the same profile. The job recomputes the name from `VERACODE_APP_NAME` or `github.repository` rather than taking `needs: package`, so SCA still starts in parallel. Errors are swallowed with `|| echo` so SCA never blocks the build. Remove that suffix to enforce SCA policy.

### pipeline-scan

Scans each artifact in `artifact_list.txt` separately, keeps going after a failure, then fails the job if any artifact failed.

| Event | Arguments added | Result |
|-------|-----------------|--------|
| `push` to `feature/**` | none | Every flaw counts, any finding fails the job |
| `pull_request` | `--policy_name "Veracode Recommended Very High"` | Only policy-violating flaws fail the job |

Results are uploaded as `pipeline-scan-results` (`<artifact>_results.json`), even on failure.

### policy-scan

Downloads the latest [Veracode Java API Wrapper](https://docs.veracode.com/r/c_about_wrappers) from Maven Central and runs `UploadAndScan` against the whole `verascan/` folder, so all modules land in one build.

| Parameter | Value |
|-----------|-------|
| `-appname` | `VERACODE_APP_NAME` or `org/repo` |
| `-createprofile` | `true` |
| `-autoscan` | `true` |
| `-filepath` | `verascan` |
| `-version` | `<branch>-<run_number>-<run_attempt>` (unique per re-run) |

The job returns after the upload is accepted. Results appear in the Veracode Platform when the scan completes.

---

## Behavior Notes

- **Default branch**: triggers under `on:` cannot use expressions, so `main` is hardcoded there. Job conditions use `github.event.repository.default_branch`. If your default branch is `master` or `develop`, change both `branches:` lists.
- **Duplicate runs**: a push to `feature/x` that has an open PR triggers both `push` and `pull_request`. This is expected: the push run gives full feedback, the PR run is the merge gate. Mark only the PR check as required in branch protection.
- **Concurrency**: newer runs cancel older runs on the same ref, except on the default branch, where policy uploads are never cancelled.
- **Fork PRs**: skipped, because GitHub does not expose secrets to them.
- **Self-hosted runners**: `actions/checkout@v6`, `upload-artifact@v7`, and `download-artifact@v8` run on Node.js 24 and need Actions Runner 2.327.1 or later. Java is required for the scanner and API wrapper (preinstalled on `ubuntu-latest`).

---

## Customization

**Change the PR gate policy**: edit `PR_GATE_POLICY` in the top-level `env:`. Built-in policies are supported by name. For a custom policy, download it with `--request_policy` and pass `--policy_file` instead ([parameters](https://docs.veracode.com/r/r_pipeline_scan_commands)).

**Relax the feature gate**: add arguments in the `else` branch of the gate block, for example:

```bash
GATE_ARGS=(--fail_on_severity "Very High, High")
```

**Scan all non-default branches**: replace `'feature/**'` under `on.push.branches` with `'**'`. The job conditions already handle it.

**Add a build step**: if autopackaging does not cover your project, build before the autopackager or replace it and write your artifact paths to `artifact_list.txt`:

```yaml
- name: Build
  run: mvn -B clean package -DskipTests
```

---

## Troubleshooting

| Symptom | Check |
|---------|-------|
| `No packaged artifacts found` | Run `veracode package --source . --output verascan --trust` locally. Confirm the project type is supported or add a build step. |
| Pipeline Scan exits `255` | Invalid credentials, network block to `api.veracode.com`, or an unsupported artifact. |
| Pipeline Scan exits `253` or `254` | Scan timed out. Add `--timeout <minutes>` (max 60) or split large artifacts. |
| PR gate fails unexpectedly | Open `pipeline-scan-results` and review `results.json`. Confirm the policy name matches exactly. |
| `UploadAndScan` rejected | A previous scan for the profile may still be running. Wait for it or cancel it in the Platform. Verify the API user has upload permissions. |
| Jobs skipped on default branch | Default branch in GitHub settings does not match the `branches:` filter. |

---

## Resources

- [Pipeline Scan parameters](https://docs.veracode.com/r/r_pipeline_scan_commands)
- [Pipeline Scan status codes](https://docs.veracode.com/r/Pipeline_Scan_Status_Codes)
- [Veracode CLI](https://docs.veracode.com/r/Install_the_Veracode_CLI)
- [Java API Wrapper](https://docs.veracode.com/r/c_about_wrappers)
- [Agent-Based SCA](https://docs.veracode.com/r/Agent_Based_Scans)
- [GitHub Actions: events that trigger workflows](https://docs.github.com/actions/using-workflows/events-that-trigger-workflows)
