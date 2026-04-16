# Veracode Security Pipeline for GitHub Actions

Automated security strategy that integrates multiple Veracode products into the SDLC, balancing feedback speed with analysis depth depending on the development context.

**Supported Technologies**: Java (Maven/Gradle/Ant), .NET (Core/Framework), Node.js, Python, Go, PHP, Ruby, Scala, and more.

---

## Scanning Strategy

| Context | Veracode Product | Time | Gate | Purpose |
|----------|-------------------|--------|------|-----------|
| Feature branches | Pipeline Scan | 3-10 min | No | Fast feedback for developers |
| Pull Requests | Pipeline Scan | 3-10 min | Yes (Very High/High) | Prevent vulnerabilities before merge |
| Release branches | Sandbox Scan | 30-90 min | No | Isolated pre-production validation |
| Main branch | Policy Scan | 25-60 min | Optional | Production certification |

**All contexts**: Agent-Based SCA (dependency analysis)

---

## Workflow Structure

```text
┌─────────────────────────────────────────────────────────────────┐
│                    on: push / pull_request                     │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│  PACKAGE JOB (always runs)                                     │
│  ├─ Check out code                                             │
│  ├─ Configure APP_NAME from github.repository                  │
│  ├─ Install Veracode CLI                                       │
│  ├─ Run autopackager                                           │
│  ├─ List and validate artifacts → artifact_list.txt            │
│  └─ Upload verascan/ as a workflow artifact                    │
│                                                                 │
│  Outputs: artifact_count, app_name                             │
└─────────────────────────────────────────────────────────────────┘
                                │
              ┌─────────────────┼─────────────────┐
              │                 │                 │
              ▼                 ▼                 ▼
┌───────────────────┐ ┌─────────────────┐ ┌─────────────────────┐
│  SCA JOB          │ │  PIPELINE JOBS  │ │  PLATFORM JOBS      │
│  (all branches)   │ │  SCAN           │ │                     │
│                   │ │                 │ │                     │
│  Dependency       │ │  feature-*:     │ │  release/*:         │
│  analysis         │ │   No gate       │ │   Sandbox Scan      │
│  based on agent   │ │                 │ │                     │
│                   │ │  PR:            │ │  main:              │
│                   │ │   Policy gate   │ │   Policy Scan       │
└───────────────────┘ └─────────────────┘ └─────────────────────┘
```

---

## Required Secrets

Configure in: **Settings > Secrets and variables > Actions**

| Secret | Required | Description |
|--------|-----------|-------------|
| `VERACODE_API_ID` | Yes | Veracode API ID for authentication |
| `VERACODE_API_KEY` | Yes | Veracode API key for authentication |
| `SRCCLR_API_TOKEN` | No | SCA agent token (only if you use SCA) |
| `VERACODE_APP_NAME` | No | Override the default app profile name |

### How to Obtain Credentials

1. Sign in to the [Veracode Platform](https://analysiscenter.veracode.com)
2. Click your profile (top right corner) > **API Credentials**
3. Generate or copy your API ID and Key
4. For SCA: **Workspace > Agents > Generate Token**

---

## Job Descriptions

### 1. Package Job

**Runs on**: All triggers (push and pull_request)

**Purpose**: Prepare artifacts for scanning using the Veracode CLI autopackager

| Step | Description |
|------|-------------|
| Checkout | Clone repository code |
| Configure App Name | Build profile name from `github.repository` or use the `VERACODE_APP_NAME` secret |
| Install Veracode CLI | Download and install the CLI from tools.veracode.com |
| Run Autopackager | Run `veracode package --source . --output verascan --trust` |
| List Artifacts | Find all `.war`, `.jar`, `.zip` files and create `artifact_list.txt` |
| Validate | Fail if no artifacts are found |
| Upload | Make `verascan/` available for later jobs |

**Outputs**:
- `artifact_count`: Number of artifacts found
- `app_name`: Veracode application profile name

---

### 2. SCA Job

**Runs on**: All triggers (in parallel with other jobs)

**Purpose**: Software Composition Analysis for vulnerabilities in third-party dependencies

| Step | Description |
|------|-------------|
| Checkout | Clone repository code |
| Run SCA | Run an agent-based scan with `--recursive --update-advisor` |

**Why always run SCA**: 80% of vulnerabilities come from dependencies. SCA complements SAST by analyzing third-party code.

---

### 3. Feature Pipeline Scan

**Runs on**: Pushes to `feature/*` branches

**Purpose**: Fast feedback during active development (no gate)

| Step | Description |
|------|-------------|
| Download Artifacts | Get `verascan/` from the package job |
| Download Scanner | Get `pipeline-scan.jar` from downloads.veracode.com |
| Scan Each Artifact | Iterate through `artifact_list.txt` and scan each artifact individually |
| Save Results | `scan_results/${ARTIFACT_NAME}_results.json` |
| Upload Results | Publish as a workflow artifact |

**Gate**: None (`--fail_on_severity ""`)

**Why no gate**: Developers need fast, non-blocking feedback during active development. This allows rapid iteration on secure code.

---

### 4. PR Pipeline Scan

**Runs on**: Pull requests (excluding forks)

**Purpose**: Security gate for pull requests

| Step | Description |
|------|-------------|
| Download Artifacts | Get `verascan/` from the package job |
| Download Scanner | Get `pipeline-scan.jar` |
| Scan with Gate | Use `--policy_name "Veracode Recommended Very High"` |
| Track Failures | Collect failed artifacts and report them at the end |
| Save Results | `scan_results/${ARTIFACT_NAME}_results.json` |
| Exit Code | Non-zero if any artifact fails the policy |

**Gate**: Fails on Very High and High severity findings

**Why gate PRs**: Prevents vulnerabilities from entering the main codebase. Integrates security into code review.

---

### 5. Release Sandbox Scan

**Runs on**: Pushes to `release/*` branches

**Purpose**: Full SAST analysis in an isolated sandbox environment

| Step | Description |
|------|-------------|
| Download Artifacts | Get `verascan/` from the package job |
| List Contents | Show the artifacts being uploaded |
| Download API Wrapper | Get the latest `vosp-api-wrappers-java` from Maven Central |
| Upload to Sandbox | `-filepath "verascan"` (directory upload, no zip required) |

**Sandbox Name**: `github-release` (configurable)

**Why sandbox for releases**: Full validation without affecting production metrics. Allows safe experimentation with new features.

---

### 6. Main Policy Scan

**Runs on**: Pushes to the `main` branch

**Purpose**: Production certification for compliance

| Step | Description |
|------|-------------|
| Download Artifacts | Get `verascan/` from the package job |
| List Contents | Show the artifacts being uploaded |
| Download API Wrapper | Get the latest version dynamically |
| Upload to Platform | `-filepath "verascan"` for policy evaluation |

**Why policy scan on main**: Official security certification for code in production. Provides traceability for regulations such as SOC2, PCI-DSS, etc.

---

## Application Profile Names

The workflow automatically builds the Veracode application profile name:

**Default format**: `{organization}/{repository}`

| GitHub Repository | Veracode App Name |
|--------------------|------------------------|
| `acme-corp/api-service` | `acme-corp/api-service` |
| `myorg/frontend` | `myorg/frontend` |
| `company/backend-api` | `company/backend-api` |

**Override**: Configure the `VERACODE_APP_NAME` secret to use a custom name.

---

## Multi-Artifact Handling

The workflow handles repositories with multiple deployable artifacts:

### Pipeline Scans (feature/PR)

Each artifact is scanned individually:

```text
verascan/
  ├── backend-api.jar      → scanned separately
  ├── frontend.zip         → scanned separately
  └── common-lib.jar       → scanned separately

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

**Note**: No zip bundle is created. The Java API Wrapper accepts a directory path directly, avoiding zip bomb rejections.

---

## Scan Results

### Pipeline Scan Results

Results are saved per artifact in `scan_results/`:

```text
scan_results/
  backend-api.jar_results.json
  frontend.zip_results.json
  common-lib.jar_results.json
```

**Download**: Workflow run > Artifacts section > `pipeline-scan-results` or `pipeline-scan-gate-results`

**JSON Structure**:
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

### Change PR Gate Policy

In the `pr-pipeline-scan` job:

```yaml
--policy_name "Your Custom Policy"
```

### Change Severity Gate

```yaml
--fail_on_severity "Very High"           # Fail only on Very High
--fail_on_severity "Very High, High"     # Fail on Very High and High (default)
--fail_on_severity ""                    # No gate (informational only)
```

### Custom Sandbox Name

In the `release-sandbox-scan` job:

```yaml
-sandboxname "your-sandbox-name"
```

### Add a Build Step

If the autopackager does not work for your project, add a build step:

```yaml
- name: Build Application
  run: |
    # Maven
    mvn -B clean package -DskipTests

    # Gradle
    ./gradlew build -x test

    # npm
    npm ci && npm run build
```

### Change Trigger Branches

Modify the `on` section:

```yaml
on:
  push:
    branches:
      - main
      - develop
      - 'release/**'
      - 'feature/**'
```

---

## Pipeline Scan Parameters

| Parameter | Description | Used In |
|-----------|-------------|----------|
| `-f` | File to scan | All pipeline scans |
| `-vid` | Veracode API ID | All scans |
| `-vkey` | Veracode API key | All scans |
| `--fail_on_severity` | Severities that fail the build | Feature (empty), PR (Very High,High) |
| `--policy_name` | Policy to evaluate against | PR scans |
| `--issue_details` | Include detailed findings info | All scans |
| `-jo` | JSON output only | All scans |

---

## API Wrapper Parameters

| Parameter | Description | Used In |
|-----------|-------------|----------|
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

**Symptoms**: Package job fails with "No packaged artifacts found"

**Solutions**:
1. Make sure the project builds successfully before packaging
2. Verify compiled artifacts exist in expected locations
3. Review Veracode CLI autopackager output for errors
4. Add an explicit build step before the autopackager

### Pipeline Scan Produces No Results

**Symptoms**: `results.json` is not created for an artifact

**Causes**:
- Artifact is not scannable (test JARs, resource bundles)
- Artifact does not contain application code
- Unsupported module type

**Solutions**:
1. Review Pipeline Scanner output for warnings
2. Verify the artifact contains application code
3. This is normal for some artifacts; the workflow continues

### Policy Gate Fails Unexpectedly

**Symptoms**: PR pipeline scan fails with no obvious issues

**Solutions**:
1. Verify the policy name matches exactly (case-sensitive)
2. Confirm the policy exists in your Veracode account
3. Review `results.json` for specific findings
4. Verify the policy thresholds

### Fork PRs Fail or Are Skipped

**Symptoms**: PRs from forks do not run or fail with authentication errors

**Cause**: Fork PRs do not have access to repository secrets by default

**Solutions**:
1. The workflow intentionally skips fork PRs
2. For open source, consider `pull_request_target` (with caution)
3. Require contributors to run scans locally

### Sandbox/Policy Upload Fails

**Symptoms**: API Wrapper fails during upload

**Solutions**:
1. Verify that `VERACODE_API_ID` and `VERACODE_API_KEY` are configured
2. Verify the credentials have upload permissions
3. Ensure the application profile name is valid (no special characters)
4. Review API Wrapper output for errors

---

## Files

| File | Description |
|---------|--------|-------------|
| `veracode-scans.yml` | GitHub Actions workflow |
| `veracode-strategy.md` | Documentation |

---

## Quick Start

1. Copy `veracode-scans.yml` to `.github/workflows/veracode.yml`
2. Add secrets in **Settings > Secrets and variables > Actions**:
   - `VERACODE_API_ID`
   - `VERACODE_API_KEY`
   - `SRCCLR_API_TOKEN` (optional)
3. Push to a feature branch to trigger the first scan
4. Review results in the workflow artifacts

---

## Best Practices

**Shift-Left**:
- Run Pipeline Scan on every commit
- Enable gates on PRs to block High/Very High findings
- Educate the team on common findings

**Compliance**:
- Require Policy Scan before production
- Maintain scan history
- Document exceptions and mitigations

**Optimization**:
- Use Pipeline Scan for fast iteration
- Reserve Policy Scan for official releases
- Cache dependencies to speed up builds

**SCA**:
- Continuously monitor new CVEs
- Update dependencies regularly
- Review licenses before adopting libraries

---

## Resources

- [Veracode Documentation](https://docs.veracode.com)
- [Veracode CLI Installation](https://docs.veracode.com/r/Install_the_Veracode_CLI)
- [Pipeline Scan](https://docs.veracode.com/r/Pipeline_Scan)
- [Java API Wrapper](https://docs.veracode.com/r/c_about_wrappers)
- [SCA Agent-Based Scans](https://docs.veracode.com/r/Agent_Based_Scans)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
