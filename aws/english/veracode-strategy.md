# Veracode Security Pipeline for AWS CodeBuild

Buildspec: [`buildspec.yml`](./buildspec.yml). Place it in your repository root (default CodeBuild buildspec name).

**Supported technologies**: anything the [Veracode CLI autopackager](https://docs.veracode.com/r/About_auto_packaging) supports. Use an `aws/codebuild/standard` (Ubuntu) or `amazonlinux` image that supports `java: corretto17`, and add the toolchains your project needs.

---

## Scanning Strategy

| Trigger | Scan mode | Veracode Product | Gate |
|---------|-----------|------------------|------|
| Push to `feature/*` | `feature` | Pipeline Scan | Fails on any flaw |
| Pull request to default branch | `pr` | Pipeline Scan | `Veracode Recommended Very High` policy |
| Push to default branch | `policy` | Policy Scan | Platform policy |
| Anything else | `skip` | none | Build passes without scanning |
| Any scanning mode | | Agent-Based SCA | Non-blocking |

Pipeline Scan exit codes ([docs](https://docs.veracode.com/r/Pipeline_Scan_Status_Codes)): `0` no flaws, `1-200` number of flaws that matched the criteria, `253-255` timeout or error. Any non-zero code fails the build.

---

## How the Scan Mode Is Resolved

The first `build` command sets `SCAN_MODE` from [CodeBuild webhook variables](https://docs.aws.amazon.com/codebuild/latest/userguide/build-env-ref-env-vars.html):

| Source | Logic |
|--------|-------|
| `SCAN_MODE` already set | Used as-is (manual override) |
| `CODEBUILD_WEBHOOK_TRIGGER=pr/<n>` | `pr` if `CODEBUILD_WEBHOOK_BASE_REF` is `refs/heads/$DEFAULT_BRANCH`, else `skip` |
| `CODEBUILD_WEBHOOK_TRIGGER=branch/<name>` | `policy` for the default branch, `feature` for `feature/*`, else `skip` |
| No webhook variables (CodePipeline, manual start) | Same branch logic using `BRANCH_NAME` |

Mode resolution runs in the same phase as the scans because buildspec `0.2` shares one shell between commands.

---

## Setup

### 1. Credentials in Secrets Manager

Create a secret named `veracode/credentials` with key/value pairs:

```json
{
  "VERACODE_API_ID": "...",
  "VERACODE_API_KEY": "...",
  "SRCCLR_API_TOKEN": "..."
}
```

Grant the CodeBuild service role `secretsmanager:GetSecretValue` on the secret. The buildspec reads it with `env.secrets-manager` using the `secret-id:json-key` form ([buildspec reference](https://docs.aws.amazon.com/codebuild/latest/userguide/build-spec-ref.html)). Uncomment the `SRCCLR_API_TOKEN` line once the key exists; a missing key fails the build at startup.

API credentials: [Generate API credentials](https://docs.veracode.com/r/t_create_api_creds). SCA token: [Create an SCA agent](https://docs.veracode.com/r/t_sc_cli_agent).

### 2. Project environment variables (optional)

| Variable | Default | Description |
|----------|---------|-------------|
| `VERACODE_APP_NAME` | CodeBuild project name | Application profile name |
| `DEFAULT_BRANCH` | `main` | Default branch |
| `BRANCH_NAME` | none | Branch when CodePipeline starts the build. Pass `#{SourceVariables.BranchName}` from the source action. |
| `SCAN_MODE` | resolved | Force `policy`, `feature`, `pr`, or `skip` |

### 3. Webhook filter groups (GitHub, GitHub Enterprise Server, Bitbucket sources)

Configure **Primary source webhook events** with two filter groups:

| Group | `EVENT` | Additional filter |
|-------|---------|-------------------|
| 1 | `PUSH` | `HEAD_REF` = `^refs/heads/(main\|feature/.*)$` |
| 2 | `PULL_REQUEST_CREATED, PULL_REQUEST_UPDATED, PULL_REQUEST_REOPENED` | `BASE_REF` = `^refs/heads/main$` |

The buildspec still validates the trigger, so a looser filter only costs build minutes, it never runs the wrong scan.

**CodePipeline**: webhook variables are not available when CodePipeline starts CodeBuild. Set `BRANCH_NAME` on the CodeBuild action. CodePipeline has no native PR event, so use `SCAN_MODE=pr` in a dedicated pipeline if you need PR gating there.

---

## Build Steps

1. **Resolve mode** and validate credentials (credentials are only required when a scan will run).
2. **Package**: installs the Veracode CLI, runs `veracode package --source . --output verascan --trust`, writes `artifact_list.txt`, fails if empty.
3. **SCA**: `sca-downloads.veracode.com/ci.sh scan --recursive --update-advisor --appname "$APP_NAME"`, errors swallowed with `|| echo`. The `--appname` value is the same application profile the policy scan uploads to, so agent-based findings land against the same profile. `APP_NAME` is resolved once in the mode-resolution step and reused by the policy scan.
4. **Pipeline Scan** (`feature`/`pr`): each artifact scanned separately, failures aggregated, build fails at the end. `pr` adds `--policy_name "$PR_GATE_POLICY"`. Results go to `scan_results/`.
5. **Policy Scan** (`policy`): latest [Java API Wrapper](https://docs.veracode.com/r/c_about_wrappers) `UploadAndScan` on `verascan/`, version `<default branch>-<CODEBUILD_BUILD_NUMBER>`.

To keep Pipeline Scan results, add an `artifacts` section for `scan_results/**/*` and configure project artifacts (S3).

---

## Customization

**Change the PR gate policy**: edit `PR_GATE_POLICY` under `env.variables`.

**Relax the feature gate**: set `GATE_ARGS=(--fail_on_severity "Very High, High")` in the `else` branch.

**Scan every non-default branch**: change `feature/*)` to `*)` in the branch `case` and widen the `HEAD_REF` filter.

**Parameter Store instead of Secrets Manager**: replace the block with `env.parameter-store` entries pointing to SecureString parameters.

---

## Troubleshooting

| Symptom | Check |
|---------|-------|
| Build fails at startup with a Secrets Manager error | Secret name or JSON key mismatch, or missing `GetSecretValue` permission. |
| Always `Scan mode: skip` from CodePipeline | `BRANCH_NAME` not passed to the CodeBuild action. |
| PR build shows `skip` | PR target is not `DEFAULT_BRANCH`. |
| `No packaged artifacts found` | Build image lacks your toolchain. |
| Pipeline Scan exits `255` | Invalid credentials, network block to `api.veracode.com`, or unsupported artifact. |
| Pipeline Scan exits `253` or `254` | Scan timed out. Add `--timeout <minutes>` (max 60). |
| `UploadAndScan` rejected | A previous scan for the profile may still be running. |

---

## Resources

- [CodeBuild buildspec reference](https://docs.aws.amazon.com/codebuild/latest/userguide/build-spec-ref.html)
- [CodeBuild environment variables](https://docs.aws.amazon.com/codebuild/latest/userguide/build-env-ref-env-vars.html)
- [Pipeline Scan parameters](https://docs.veracode.com/r/r_pipeline_scan_commands)
- [Java API Wrapper](https://docs.veracode.com/r/c_about_wrappers)
- [SCA CI script](https://docs.veracode.com/r/c_sc_ci_script)
