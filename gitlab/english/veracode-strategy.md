# Veracode Security Pipeline for Bitbucket

Automated security strategy that integrates multiple Veracode products into the SDLC, balancing feedback speed with analysis depth depending on the development context.

**Supported Technologies**: Java (Maven/Gradle/Ant), .NET (Core/Framework), Node.js, Python, Go, PHP, Ruby, Scala, and more.

---

## Scanning Strategy

| Context | Veracode Product | Time | Gate | Purpose |
|---------|------------------|------|------|---------|
| `feature/*` branches | Pipeline Scan | 3-10 min | No | Fast feedback for developers |
| Pull Requests | Pipeline Scan | 3-10 min | Yes (Very High / High) | Prevent vulnerabilities before merge |
| `release/*` branches | Sandbox Scan | 30-90 min | No | Isolated pre-production validation |
| `main` branch | Policy Scan | 25-60 min | Optional | Production certification |

**All contexts**: Agent-Based SCA (dependency analysis) runs in parallel.

---

## Workflow Structure

```text
+------------------------------------------------------------------+
|     Trigger: push to branch  /  pull_request                     |
+------------------------------------------------------------------+
                                |
                                v
+------------------------------------------------------------------+
|  PACKAGE STEP (always runs)                                      |
|   - Resolve APP_NAME (VERACODE_APP_NAME or BITBUCKET_REPO_FULL)  |
|   - Install Veracode CLI                                         |
|   - Run autopackager                                             |
|   - Build artifact_list.txt                                      |
|   - Validate artifacts exist                                     |
|   - Publish: veracode CLI, verascan/, artifact_list.txt,         |
|              app_name.txt as Bitbucket artifacts                 |
+------------------------------------------------------------------+
                                |
                  +-------------+-------------+
                  |                           |
                  v                           v
       +------------------+        +-----------------------+
       |   SCA STEP       |        |   SCAN STEP           |
       |   (parallel)     |        |   (branch dependent)  |
       |                  |        |                       |
       |   Agent-based    |        |   feature/* -> Pipe   |
       |   dependency     |        |   PR        -> PipeG  |
       |   scan           |        |   release/* -> Sand   |
       |                  |        |   main      -> Policy |
       +------------------+        +-----------------------+
```

---

## Required Repository Variables

Configure under **Repository settings > Pipelines > Repository variables**.

| Variable | Required | Secured | Description |
|----------|----------|---------|-------------|
| `VERACODE_API_ID` | Yes | No | Veracode API ID for authentication |
| `VERACODE_API_KEY` | Yes | Yes | Veracode API key for authentication |
| `SRCCLR_API_TOKEN` | No | Yes | SCA agent token (only if SCA is used) |
| `VERACODE_APP_NAME` | No | No | Override the default application profile name |

### How to Obtain Credentials

1. Sign in to the [Veracode Platform](https://analysiscenter.veracode.com)
2. Click your profile (top right) > **API Credentials**
3. Generate or copy your API ID and Key
4. For SCA: **Workspace > Agents > Generate Token**

---

## Step Descriptions

### 1. Package Step

**Runs on**: All triggers (every branch and every pull request)

**Purpose**: Prepare artifacts for scanning using the Veracode CLI autopackager.

| Sub-step | Description |
|----------|-------------|
| Resolve App Name | Use `VERACODE_APP_NAME` if set, otherwise `$BITBUCKET_REPO_FULL_NAME` |
| Install Veracode CLI | Download and install the CLI from `tools.veracode.com` |
| Run Autopackager | Run `veracode package --source . --output verascan --trust` |
| List Artifacts | Find all `.war`, `.jar`, `.zip` files into `artifact_list.txt` |
| Validate | Fail if no artifacts are produced |
| Publish | Pass `verascan/`, `artifact_list.txt`, `app_name.txt`, and the CLI binary forward as Bitbucket artifacts |

---

### 2. SCA Step

**Runs on**: All triggers (in parallel with the scan step)

**Purpose**: Software Composition Analysis for vulnerabilities in third-party dependencies.

**Why always run SCA**: A large portion of vulnerabilities come from dependencies. SCA complements SAST by analyzing third-party code, transitive dependencies, and license risk.

---

### 3. Feature Pipeline Scan

**Runs on**: Pushes to `feature/*` branches

**Purpose**: Fast, non-blocking feedback during active development.

| Sub-step | Description |
|----------|-------------|
| Download artifacts | Receive `verascan/` and `artifact_list.txt` from package step |
| Download Scanner | Get `pipeline-scan.jar` from `downloads.veracode.com` |
| Scan each artifact | Loop through `artifact_list.txt`, scan each individually |
| Persist results | Save each scan as `scan_results/${ARTIFACT_NAME}_results.json` |

**Gate**: None (`--fail_on_severity ""`).

**Why no gate**: Developers need fast, non-blocking feedback during active development.

---

### 4. PR Pipeline Scan (Gate)

**Runs on**: Pull requests targeting any branch

**Purpose**: Block merges with Very High and High severity findings.

| Sub-step | Description |
|----------|-------------|
| Download artifacts | Receive `verascan/` and `artifact_list.txt` from package step |
| Download Scanner | Get `pipeline-scan.jar` |
| Scan with Gate | Use `--policy_name "Veracode Recommended Very High"` and `--fail_on_severity "Very High, High"` |
| Track failures | Collect every artifact that fails the gate, report at the end |
| Exit code | Non-zero if any artifact fails the gate |

**Why gate PRs**: Prevents vulnerabilities from entering protected branches and integrates security into code review.

---

### 5. Release Sandbox Scan

**Runs on**: Pushes to `release/*` branches

**Purpose**: Full SAST analysis in an isolated sandbox environment.

| Sub-step | Description |
|----------|-------------|
| Download artifacts | Receive `verascan/` and `app_name.txt` |
| Download API Wrapper | Get the latest `vosp-api-wrappers-java` from Maven Central |
| Upload | `-filepath "verascan"` (directory upload, no zip required) |

**Sandbox Name**: `bitbucket-release` (configurable in the YAML).

**Why a sandbox for releases**: Full validation without affecting production policy metrics. Allows safe experimentation with new features before promoting to main.

---

### 6. Main Policy Scan

**Runs on**: Pushes to `main`

**Purpose**: Production certification for compliance.

| Sub-step | Description |
|----------|-------------|
| Download artifacts | Receive `verascan/` and `app_name.txt` |
| Download API Wrapper | Get the latest version dynamically |
| Upload | `-filepath "verascan"` for policy evaluation |

**Why a policy scan on main**: Official security certification of code in production. Provides traceability for SOC2, PCI-DSS, ISO 27001, and similar regulations.

---

## Application Profile Names

The pipeline automatically builds the Veracode application profile name.

**Default format**: `$BITBUCKET_REPO_FULL_NAME` (which is `{workspace}/{repo}`).

| Bitbucket Repository | Veracode App Name |
|----------------------|-------------------|
| `acme-corp/api-service` | `acme-corp/api-service` |
| `myorg/frontend` | `myorg/frontend` |
| `company/backend-api` | `company/backend-api` |

**Override**: Set the `VERACODE_APP_NAME` repository variable to use a custom name.

---

## Multi-Artifact Handling

The pipeline handles repositories with multiple deployable artifacts.

### Pipeline Scans (feature / PR)

Each artifact is scanned individually:

```text
verascan/
  backend-api.jar      -> scanned separately
  frontend.zip         -> scanned separately
  common-lib.jar       -> scanned separately

scan_results/
  backend-api.jar_results.json
  frontend.zip_results.json
  common-lib.jar_results.json
```

### Platform Scans (release / main)

All artifacts are uploaded together via directory:

```text
-filepath "verascan"    # API Wrapper handles multi-file upload natively
```

**Note**: No zip bundle is created. The Java API Wrapper accepts a directory path directly, avoiding zip bomb rejection issues.

---

## Scan Results

### Pipeline Scan Results

Saved per artifact in `scan_results/` and exposed as Bitbucket step artifacts.

**Download**: Pipeline run > step > **Artifacts** tab.

**JSON Structure**:
```json
{
  "findings": [...],
  "pipeline_scan": {...},
  "scan_status": "SUCCESS"
}
```

### Platform Scan Results (Sandbox / Policy)

Check in the Veracode Platform:
1. Sign in to [analysiscenter.veracode.com](https://analysiscenter.veracode.com)
2. Navigate to your application profile
3. Review findings, compliance status, and trends

---

## Customization

### Change the PR Gate Policy

In the `&pipeline-scan-gate` step:

```yaml
--policy_name "Your Custom Policy"
```

### Change the Severity Gate

```yaml
--fail_on_severity "Very High"             # Fail only on Very High
--fail_on_severity "Very High, High"       # Fail on Very High and High (default)
--fail_on_severity ""                      # No gate (informational only)
```

### Custom Sandbox Name

In the `&sandbox-scan` step:

```yaml
-sandboxname "your-sandbox-name"
```

### Add a Build Step

If the autopackager does not work for your project, add a build step before the package step:

```yaml
- step:
    name: Build
    script:
      # Maven
      - mvn -B clean package -DskipTests
      # Gradle
      # - ./gradlew build -x test
      # npm
      # - npm ci && npm run build
    artifacts:
      - target/**
      - build/**
      - dist/**
```

### Change Trigger Branches

Modify the `pipelines` section:

```yaml
pipelines:
  branches:
    main:
      - step: *package
    develop:
      - step: *package
    'release/*':
      - step: *package
    'feature/*':
      - step: *package
```

---

## Pipeline Scan Parameters

| Parameter | Description | Used In |
|-----------|-------------|---------|
| `-f` | File to scan | All pipeline scans |
| `-vid` | Veracode API ID | All scans |
| `-vkey` | Veracode API Key | All scans |
| `--fail_on_severity` | Severities that fail the build | Feature (empty), PR (`Very High, High`) |
| `--policy_name` | Veracode policy to evaluate against | PR scans |
| `--issue_details` | Include detailed findings information | All scans |
| `-jo` | JSON output only | All scans |

> Note: `--policy_name` is the correct parameter (the deprecated alias `--policy` should be avoided).

---

## API Wrapper Parameters

| Parameter | Description | Used In |
|-----------|-------------|---------|
| `-action` | `UploadAndScan` | All platform scans |
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

**Symptoms**: Package step fails with "No packaged artifacts found".

**Solutions**:
1. Make sure the project builds successfully before packaging
2. Verify compiled artifacts exist in expected locations
3. Review the Veracode CLI autopackager output for errors
4. Add an explicit build step before the autopackager

### Pipeline Scan Produces No Results

**Symptoms**: `results.json` is not created for an artifact.

**Causes**:
- Artifact is not scannable (test JARs, resource bundles)
- Artifact does not contain application code
- Unsupported module type

**Solutions**:
1. Review Pipeline Scanner output for warnings
2. Verify the artifact contains application code
3. This is normal for some artifacts; the pipeline continues

### Policy Gate Fails Unexpectedly

**Symptoms**: PR pipeline scan fails with no obvious issues.

**Solutions**:
1. Verify the policy name matches exactly (case-sensitive)
2. Confirm the policy exists in your Veracode account
3. Review `results.json` for specific findings
4. Verify the policy thresholds

### Sandbox or Policy Upload Fails

**Symptoms**: API Wrapper fails during upload.

**Solutions**:
1. Verify `VERACODE_API_ID` and `VERACODE_API_KEY` are set as repository variables
2. Verify the credentials have upload permissions
3. Ensure the application profile name is valid (no special characters)
4. Review API Wrapper output for errors

---

## Files

| File | Description |
|------|-------------|
| `veracode-pipeline.yml` | Bitbucket Pipelines configuration |
| `veracode-strategy.md` | Documentation |

---

## Quick Start

1. Copy `veracode-pipeline.yml` to the root of your repository
2. Add repository variables under **Repository settings > Pipelines > Repository variables**:
   - `VERACODE_API_ID`
   - `VERACODE_API_KEY` (mark as secured)
   - `SRCCLR_API_TOKEN` (optional, mark as secured)
3. Enable Pipelines under **Repository settings > Pipelines > Settings**
4. Push to a `feature/*` branch to trigger the first scan
5. Review results in the pipeline run artifacts and the Veracode Platform

---

## Best Practices

**Shift-Left**:
- Run Pipeline Scan on every commit
- Enable gates on PRs to block High and Very High findings
- Educate the team on common findings

**Compliance**:
- Require Policy Scan before production
- Maintain scan history for audit
- Document exceptions and mitigations

**Optimization**:
- Use Pipeline Scan for fast iteration
- Reserve Policy Scan for official releases
- Cache Maven, Gradle, npm, and pip dependencies to speed up builds

**SCA**:
- Continuously monitor new CVEs
- Update dependencies regularly
- Review licenses before adopting new libraries

---

## Resources

- [Veracode Documentation](https://docs.veracode.com)
- [Veracode CLI Installation](https://docs.veracode.com/r/Install_the_Veracode_CLI)
- [Pipeline Scan Commands](https://docs.veracode.com/r/r_pipeline_scan_commands)
- [Pipeline Scan Examples](https://docs.veracode.com/r/r_pipeline_scan_examples)
- [Java API Wrapper](https://docs.veracode.com/r/c_about_wrappers)
- [SCA Agent-Based Scans](https://docs.veracode.com/r/Agent_Based_Scans)
- [Bitbucket Pipelines Documentation](https://support.atlassian.com/bitbucket-cloud/docs/get-started-with-veracode-pipeline/)
