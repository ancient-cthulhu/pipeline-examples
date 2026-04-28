# Veracode Security Pipeline for GitLab CI/CD

Automated security strategy that integrates multiple Veracode products into the SDLC, balancing feedback speed with analysis depth depending on the development context.

**Supported Technologies**: Java (Maven/Gradle/Ant), .NET (Core/Framework), Node.js, Python, Go, PHP, Ruby, Scala, and more.

---

## Scanning Strategy

| Context | Veracode Product | Time | Gate | Purpose |
|---------|------------------|------|------|---------|
| Feature branches (`feature/*`) | Pipeline Scan | 3-10 min | No | Fast feedback for developers |
| Merge Requests | Pipeline Scan | 3-10 min | Yes (Very High/High) | Prevent vulnerabilities before merge |
| Release branches (`release/*`) | Sandbox Scan | 30-90 min | No | Isolated pre-production validation |
| `main` branch | Policy Scan | 25-60 min | Optional | Production certification |

**All contexts**: Agent-Based SCA (dependency analysis), runs in parallel.

---

## Workflow Structure

```text
┌─────────────────────────────────────────────────────────────────┐
│           on: push (branches) / merge_request_event             │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│  STAGE: package                                                 │
│  package job (always runs for matched branches/MRs)             │
│  - Checkout code                                                │
│  - Configure APP_NAME from CI_PROJECT_PATH or override          │
│  - Install Veracode CLI                                         │
│  - Run autopackager                                             │
│  - List and validate artifacts -> artifact_list.txt             │
│  - Publish verascan/ as job artifacts                           │
│  - Outputs: APP_NAME, ARTIFACT_COUNT (via dotenv report)        │
└─────────────────────────────────────────────────────────────────┘
                                │
              ┌─────────────────┼─────────────────┐
              │                 │                 │
              ▼                 ▼                 ▼
┌──────────────────┐ ┌──────────────────┐ ┌─────────────────────┐
│  STAGE: sca      │ │  STAGE: scan     │ │  STAGE: scan        │
│  sca job         │ │  Pipeline Scans  │ │  Platform Scans     │
│  (all triggers,  │ │                  │ │                     │
│  parallel)       │ │  feature/*:      │ │  release/*:         │
│                  │ │    No gate       │ │    Sandbox Scan     │
│  Dependency      │ │                  │ │                     │
│  analysis        │ │  MR:             │ │  main:              │
│  agent-based     │ │    Policy gate   │ │    Policy Scan      │
└──────────────────┘ └──────────────────┘ └─────────────────────┘
```

The `workflow:rules` block prevents duplicate pipelines: when an MR is open for a branch, only the MR pipeline runs.

---

## Required CI/CD Variables

Configure in: **Settings > CI/CD > Variables**

For all secrets, enable **Masked** and **Protected** (when only protected branches like `main` and `release/*` should access them).

| Variable | Required | Description |
|----------|----------|-------------|
| `VERACODE_API_ID` | Yes | Veracode API ID for authentication |
| `VERACODE_API_KEY` | Yes | Veracode API key for authentication |
| `SRCCLR_API_TOKEN` | No | SCA agent token (only if you use SCA) |
| `VERACODE_APP_NAME` | No | Override the default app profile name |

### How to Obtain Credentials

1. Sign in to the [Veracode Platform](https://analysiscenter.veracode.com)
2. Click your profile (top right corner) > **API Credentials**
3. Generate or copy your API ID and Key
4. For SCA: **Workspace > Agents > Generate Token**

### Variable Protection Notes

- `VERACODE_API_KEY` and `SRCCLR_API_TOKEN` should be **masked**.
- If marked **protected**, they only inject into pipelines for protected branches/tags. Make sure `main` and `release/*` are configured as protected refs (Settings > Repository > Protected branches), or unmark **protected** so feature branches and MRs can also use them.
- For shared infrastructure, prefer **group-level** variables to avoid duplication across projects.

---

## Job Descriptions

### 1. `package` (stage: `package`)

**Runs on**: All matched triggers (push to `main`, `release/*`, `feature/*`, and merge request events)

**Purpose**: Prepare artifacts for scanning using the Veracode CLI autopackager

| Step | Description |
|------|-------------|
| Set App Name | Build profile name from `CI_PROJECT_PATH` or use the `VERACODE_APP_NAME` variable |
| Install Veracode CLI | Download and install the CLI from `tools.veracode.com` |
| Run Autopackager | Run `veracode package --source . --output verascan --trust` |
| List Artifacts | Find all `.war`, `.jar`, `.zip` files and create `artifact_list.txt` |
| Validate | Fail if no artifacts are found |
| Publish | Make `verascan/` available as a job artifact for downstream stages |

**Outputs (via `reports:dotenv`)**:
- `APP_NAME`: Veracode application profile name (consumed by sandbox/policy jobs)
- `ARTIFACT_COUNT`: Number of artifacts found

---

### 2. `sca` (stage: `sca`)

**Runs on**: All matched triggers, in parallel with package (uses `needs: []`)

**Purpose**: Software Composition Analysis for vulnerabilities in third-party dependencies

| Step | Description |
|------|-------------|
| Run SCA | Run an agent-based scan with `--recursive --update-advisor` against `sca-downloads.veracode.com/ci.sh` |

**Why always run SCA**: A large portion of application vulnerabilities come from dependencies. SCA complements SAST by analyzing third-party code.

**Behavior if `SRCCLR_API_TOKEN` is unset**: the job logs a warning and exits 0. The job is also marked `allow_failure: true` so SCA never blocks the build.

---

### 3. `feature-pipeline-scan` (stage: `scan`)

**Runs on**: Pushes to `feature/*` branches (only when no MR is open for that branch)

**Purpose**: Fast feedback during active development (no gate)

| Step | Description |
|------|-------------|
| Download Artifacts | Get `verascan/` via `needs:artifacts: true` |
| Download Scanner | Get `pipeline-scan.jar` from `downloads.veracode.com` |
| Scan Each Artifact | Iterate through `artifact_list.txt` and scan each artifact individually |
| Save Results | `scan_results/${ARTIFACT_NAME}_results.json` |
| Publish | Job artifacts retained for 1 month |

**Gate**: None (`--fail_on_severity ""`)

---

### 4. `mr-pipeline-scan` (stage: `scan`)

**Runs on**: Merge request events where the source project equals the target project (excludes forks)

**Purpose**: Security gate for merge requests

| Step | Description |
|------|-------------|
| Download Artifacts | Get `verascan/` from the package job |
| Download Scanner | Get `pipeline-scan.jar` |
| Scan with Gate | Use `--policy_name "Veracode Recommended Very High"` |
| Track Failures | Collect failed artifacts and report them at the end |
| Save Results | `scan_results/${ARTIFACT_NAME}_results.json` |
| Exit Code | Non-zero if any artifact fails the policy |

**Gate**: Fails on Very High and High severity findings.

**Fork handling**: The rule `$CI_MERGE_REQUEST_SOURCE_PROJECT_ID == $CI_MERGE_REQUEST_PROJECT_ID` skips this job for fork-originated MRs. Forks do not have access to the parent project's masked variables. To run gated scans for fork MRs, a maintainer can trigger the pipeline in the parent project (Pipelines tab > Run pipeline). See [Run pipelines in fork merge requests](https://docs.gitlab.com/ci/pipelines/merge_request_pipelines/#run-pipelines-in-fork-merge-requests).

---

### 5. `release-sandbox-scan` (stage: `scan`)

**Runs on**: Pushes to `release/*` branches

**Purpose**: Full SAST analysis in an isolated sandbox environment

| Step | Description |
|------|-------------|
| Download Artifacts | Get `verascan/` from the package job |
| List Contents | Show the artifacts being uploaded |
| Download API Wrapper | Get the latest `vosp-api-wrappers-java` from Maven Central |
| Upload to Sandbox | `-filepath "verascan"` (directory upload, no zip required) |

**Sandbox Name**: `gitlab-release` (configurable in the job script).

**Version label**: `Release ${CI_PIPELINE_IID}` (project-scoped, monotonically increasing).

---

### 6. `main-policy-scan` (stage: `scan`)

**Runs on**: Pushes to the `main` branch

**Purpose**: Production certification for compliance

| Step | Description |
|------|-------------|
| Download Artifacts | Get `verascan/` from the package job |
| List Contents | Show the artifacts being uploaded |
| Download API Wrapper | Get the latest version dynamically |
| Upload to Platform | `-filepath "verascan"` for policy evaluation |

**Version label**: `main ${CI_PIPELINE_IID}`.

---

## Application Profile Names

The pipeline automatically builds the Veracode application profile name from GitLab predefined variables.

**Default format**: `${CI_PROJECT_PATH}` which is `{namespace}/{project}`.

| GitLab project | Veracode App Name |
|----------------|-------------------|
| `acme-corp/api-service` | `acme-corp/api-service` |
| `myorg/frontend` | `myorg/frontend` |
| `company/group/backend-api` | `company/group/backend-api` |

**Override**: Configure the `VERACODE_APP_NAME` CI/CD variable to use a custom name.

---

## GitHub Actions to GitLab CI Equivalents

For reference when comparing the two pipelines:

| GitHub Actions | GitLab CI/CD |
|----------------|--------------|
| `secrets.X` | `$X` (CI/CD variable) |
| `${{ github.repository }}` | `$CI_PROJECT_PATH` |
| `${{ github.run_number }}` | `$CI_PIPELINE_IID` |
| `${{ github.ref_name }}` | `$CI_COMMIT_BRANCH` |
| `github.event_name == 'pull_request'` | `$CI_PIPELINE_SOURCE == "merge_request_event"` |
| `pull_request.head.repo.fork == false` | `$CI_MERGE_REQUEST_SOURCE_PROJECT_ID == $CI_MERGE_REQUEST_PROJECT_ID` |
| `actions/upload-artifact` | `artifacts:` keyword |
| `actions/download-artifact` | `needs:artifacts: true` |
| job `outputs` | `artifacts:reports:dotenv` |
| `if:` on a job | `rules:` |

---

## Multi-Artifact Handling

The pipeline handles repositories with multiple deployable artifacts.

### Pipeline Scans (feature/MR)

Each artifact is scanned individually:

```text
verascan/
  ├── backend-api.jar      -> scanned separately
  ├── frontend.zip         -> scanned separately
  └── common-lib.jar       -> scanned separately

scan_results/
  ├── backend-api.jar_results.json
  ├── frontend.zip_results.json
  └── common-lib.jar_results.json
```

### Platform Scans (release/main)

All artifacts are uploaded together via directory:

```text
-filepath "verascan"    # API Wrapper handles multi-file upload natively
```

No zip bundle is created. The Java API Wrapper accepts a directory path directly, avoiding zip bomb rejections.

---

## Scan Results

### Pipeline Scan Results

Results are saved per artifact in `scan_results/`. Download from the pipeline job page:

**Pipeline > Job > Job artifacts > Browse / Download**

Artifact names:
- `pipeline-scan-results-<sha>` for feature scans
- `pipeline-scan-gate-results-<sha>` for MR scans

JSON structure:
```json
{
  "findings": [...],
  "pipeline_scan": {...},
  "scan_status": "SUCCESS"
}
```

### Platform Scan Results (Sandbox/Policy)

Check in the Veracode Platform:
1. Sign in to [analysiscenter.veracode.com](https://analysiscenter.veracode.com)
2. Navigate to your application profile
3. View findings, compliance status, and trends

---

## Customization

### Change MR Gate Policy

In the `mr-pipeline-scan` job:

```yaml
--policy_name "Your Custom Policy"
```

### Change Severity Gate

```yaml
--fail_on_severity "Very High"           # Fail only on Very High
--fail_on_severity "Very High, High"     # Fail on Very High and High
--fail_on_severity ""                    # No gate (informational only)
```

### Custom Sandbox Name

In the `release-sandbox-scan` job:

```yaml
-sandboxname "your-sandbox-name"
```

### Add a Build Step

If the autopackager does not work for your project, add an explicit build before packaging. For example, change the `package` job's `script` to include:

```yaml
script:
  # Maven
  - mvn -B clean package -DskipTests
  # or Gradle
  - ./gradlew build -x test
  # or npm
  - npm ci && npm run build
  # then continue with the existing autopackager steps
```

You may also need to switch the `image:` to one with the appropriate toolchain (e.g. `maven:3.9-eclipse-temurin-17`, `node:20`).

### Change Trigger Branches

Modify both the top-level `workflow:` and the per-job `rules:`. For example, to also include `develop`:

```yaml
workflow:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH && $CI_OPEN_MERGE_REQUESTS
      when: never
    - if: $CI_COMMIT_BRANCH == "main"
    - if: $CI_COMMIT_BRANCH == "develop"
    - if: $CI_COMMIT_BRANCH =~ /^release\/.*/
    - if: $CI_COMMIT_BRANCH =~ /^feature\/.*/
```

---

## Pipeline Scan Parameters

| Parameter | Description | Used In |
|-----------|-------------|---------|
| `-f` | File to scan | All pipeline scans |
| `-vid` | Veracode API ID | All scans |
| `-vkey` | Veracode API key | All scans |
| `--fail_on_severity` | Severities that fail the build | Feature (empty), MR (via policy) |
| `--policy_name` | Policy to evaluate against | MR scans |
| `--issue_details` | Include detailed findings info | All scans |
| `-jo` | JSON output only | All scans |

---

## API Wrapper Parameters

| Parameter | Description | Used In |
|-----------|-------------|---------|
| `-action` | UploadAndScan | All platform scans |
| `-appname` | Application profile name | All platform scans |
| `-createprofile` | Create app if it does not exist | All platform scans |
| `-autoscan` | Start scan automatically | All platform scans |
| `-sandboxname` | Sandbox name | Release scans |
| `-createsandbox` | Create sandbox if it does not exist | Release scans |
| `-filepath` | Path to artifacts (file or directory) | All platform scans |
| `-version` | Scan version label | All platform scans |

---

## Troubleshooting

### No Artifacts Found

**Symptoms**: `package` job fails with "No packaged artifacts found"

**Solutions**:
1. Make sure the project builds successfully before packaging
2. Verify compiled artifacts exist in expected locations
3. Review Veracode CLI autopackager output for errors
4. Add an explicit build step before the autopackager (see Customization)

### Pipeline Scan Produces No Results

**Symptoms**: `results.json` is not created for an artifact

**Causes**:
- Artifact is not scannable (test JARs, resource bundles)
- Artifact does not contain application code
- Unsupported module type

**Solutions**:
1. Review Pipeline Scanner output for warnings
2. Verify the artifact contains application code
3. This is normal for some artifacts; the pipeline continues

### Policy Gate Fails Unexpectedly

**Symptoms**: MR pipeline scan fails with no obvious issues

**Solutions**:
1. Verify the policy name matches exactly (case-sensitive)
2. Confirm the policy exists in your Veracode account
3. Review `results.json` in the job artifacts for specific findings
4. Verify the policy thresholds

### Duplicate Pipelines on MRs

**Symptoms**: Two pipelines run for the same push when an MR is open

**Cause**: `workflow:rules` not preventing both branch and MR pipelines

**Solution**: The provided `workflow:rules` already includes `$CI_COMMIT_BRANCH && $CI_OPEN_MERGE_REQUESTS when: never`. If you customize the workflow, keep that rule near the top.

### Fork MRs Are Skipped

**Symptoms**: MRs from forks do not run the gated scan

**Cause**: Forks do not get masked variables; the rule excludes them by design

**Solutions**:
1. The pipeline intentionally skips fork MRs to protect secrets
2. A maintainer can trigger the pipeline in the parent project from the MR's Pipelines tab
3. For open source, consider a separate, secret-free job for forks

### Sandbox/Policy Upload Fails

**Symptoms**: API Wrapper fails during upload

**Solutions**:
1. Verify that `VERACODE_API_ID` and `VERACODE_API_KEY` are configured and not restricted to protected branches when needed
2. Verify the credentials have upload permissions in the Veracode Platform
3. Ensure the application profile name is valid (no special characters)
4. Review API Wrapper output for errors

### Variables Not Available on Feature Branches

**Symptoms**: Feature/MR jobs fail with "VERACODE_API_ID not set"

**Cause**: Variables marked **Protected** only inject into pipelines on protected refs

**Solutions**:
1. Either unmark **Protected** for the credential variables
2. Or protect `feature/*` and `main` and `release/*` (Settings > Repository > Protected branches)

---

## Files

| File | Description |
|------|-------------|
| `.gitlab-ci.yml` | GitLab CI/CD pipeline |
| `veracode-strategy.md` | This documentation |

---

## Quick Start

1. Copy `.gitlab-ci.yml` to the repo root.
2. Add CI/CD variables (Settings > CI/CD > Variables):
   - `VERACODE_API_ID` (masked)
   - `VERACODE_API_KEY` (masked)
   - `SRCCLR_API_TOKEN` (masked, optional)
3. Configure protected branches as needed (`main`, `release/*`).
4. Push to a `feature/*` branch to trigger the first scan.
5. Review results under the pipeline's job artifacts and the Veracode Platform.

---

## Best Practices

**Shift-Left**:
- Run Pipeline Scan on every commit to feature branches
- Enable gates on MRs to block High/Very High findings
- Educate the team on common findings

**Compliance**:
- Require Policy Scan before production deploy
- Maintain scan history in the Veracode Platform
- Document exceptions and mitigations

**Optimization**:
- Use Pipeline Scan for fast iteration
- Reserve Policy Scan for official releases
- Cache build dependencies (`cache:` keyword) to speed up builds

**SCA**:
- Continuously monitor new CVEs
- Update dependencies regularly
- Review licenses before adopting libraries

---

## Resources

- [Veracode Documentation](https://docs.veracode.com)
- [Veracode CLI Installation](https://docs.veracode.com/r/Install_the_Veracode_CLI)
- [Pipeline Scan](https://docs.veracode.com/r/Pipeline_Scan)
- [GitLab Pipeline Scan Examples (official)](https://docs.veracode.com/r/Gitlab_Pipeline_Scan_Examples)
- [Veracode Pipeline Scan CI/CD Component](https://gitlab.com/veracode/veracode-components/veracode-pipeline-scan)
- [Java API Wrapper](https://docs.veracode.com/r/c_about_wrappers)
- [SCA Agent-Based Scans](https://docs.veracode.com/r/Agent_Based_Scans)
- [GitLab CI/CD YAML Reference](https://docs.gitlab.com/ci/yaml/)
- [GitLab Predefined CI/CD Variables](https://docs.gitlab.com/ci/variables/predefined_variables/)
