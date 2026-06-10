# Veracode Security Pipeline for Jenkins

Automated security strategy that integrates multiple Veracode products into the SDLC, balancing feedback speed with analysis depth depending on the development context.

**Supported Technologies**: Java (Maven/Gradle/Ant), .NET (Core/Framework), Node.js, Python, Go, PHP, Ruby, Scala, and more.

**Pipeline Type**: Declarative Pipeline, designed for **Multibranch Pipeline** jobs (or equivalent: GitHub Organization, Bitbucket Team, GitLab Group). The `when { branch ... }` and `when { changeRequest() }` directives require a multibranch context.

---

## Scanning Strategy

| Context | Veracode Product | Time | Gate | Purpose |
|---------|------------------|------|------|---------|
| `feature/*` branches | Pipeline Scan | 3-10 min | No | Fast feedback for developers |
| Change Requests (PR/MR) | Pipeline Scan | 3-10 min | Yes (Very High/High) | Prevent vulnerabilities before merge |
| `release/*` branches | Sandbox Scan | 30-90 min | No | Isolated pre-production validation |
| `main` branch | Policy Scan | 25-60 min | Optional | Production certification |

**All contexts**: Agent-Based SCA (dependency analysis) runs in parallel.

---

## Pipeline Structure

```text
+------------------------------------------------------------------+
|              Trigger: SCM push / change request                  |
+------------------------------------------------------------------+
                                |
                                v
+------------------------------------------------------------------+
|  STAGE: Package Artifacts (always runs)                          |
|   - Resolve APP_NAME (VERACODE_APP_NAME or JOB_NAME)             |
|   - Install Veracode CLI                                         |
|   - Run autopackager: veracode package --source .                |
|   - Build artifact_list.txt                                      |
|   - Validate that artifacts exist                                |
|   - Stash verascan/, artifact_list.txt, app_name.txt             |
+------------------------------------------------------------------+
                                |
                                v
+------------------------------------------------------------------+
|  STAGE: Veracode Security Scans (parallel)                       |
+------------------------------------------------------------------+
   |              |                |                |             |
   v              v                v                v             v
+-------+  +-----------+  +-----------+  +-----------+  +-----------+
| SCA   |  | Pipeline  |  | Pipeline  |  | Sandbox   |  | Policy    |
|       |  | (feature) |  | (PR Gate) |  | (release) |  | (main)    |
| always|  | feature/* |  | PR / MR   |  | release/* |  | main      |
+-------+  +-----------+  +-----------+  +-----------+  +-----------+
```

Each parallel stage is gated by a `when {}` directive, so only one scan stage executes per build (in addition to SCA, which always runs).

---

## Required Jenkins Credentials

Configure under **Manage Jenkins > Credentials > System > Global credentials**.

| Credential ID | Type | Required | Description |
|---------------|------|----------|-------------|
| `veracode-api-id` | Secret text | Yes | Veracode API ID |
| `veracode-api-key` | Secret text | Yes | Veracode API Key |
| `srcclr-api-token` | Secret text | No | SCA Agent token (only if you use SCA) |

> If `srcclr-api-token` is not configured, the SCA stage is skipped gracefully and the rest of the pipeline continues.

### Optional Environment Variables

Set at folder, job, or pipeline level via **Configure > Environment variables** or via the `environment {}` block.

| Variable | Description |
|----------|-------------|
| `VERACODE_APP_NAME` | Override the default Veracode application profile name (defaults to `JOB_NAME`) |

### How to Obtain Credentials

1. Sign in to the [Veracode Platform](https://analysiscenter.veracode.com)
2. Click your profile (top right corner) > **API Credentials**
3. Generate or copy your API ID and Key
4. For SCA: **Workspace > Agents > Generate Token**

---

## Required Jenkins Plugins

The pipeline assumes the following plugins are installed on the Jenkins controller:

| Plugin | Purpose |
|--------|---------|
| Pipeline (Declarative) | Base declarative pipeline syntax |
| Pipeline: Multibranch | Branch and change request detection |
| Credentials Binding | `withCredentials` and `credentials()` |
| Workspace Cleanup | `cleanWs` step in `post { always }` |
| Timestamper | `timestamps()` option |

All agents must have `java` (JRE 8+), `curl`, and `unzip` available on `PATH`.

---

## Stage Descriptions

### 1. Package Artifacts

**Runs on**: All triggers (every branch and every change request)

**Purpose**: Prepare artifacts for scanning using the Veracode CLI autopackager.

| Step | Description |
|------|-------------|
| Resolve App Name | Use `VERACODE_APP_NAME` if set, otherwise `JOB_NAME` |
| Install Veracode CLI | Download from `tools.veracode.com` |
| Run Autopackager | `veracode package --source . --output verascan --trust` |
| List Artifacts | Find all `.war`, `.jar`, `.zip` and write to `artifact_list.txt` |
| Validate | Fail the build if no artifacts are produced |
| Stash | Save `verascan/`, `artifact_list.txt`, `app_name.txt` for downstream stages |

---

### 2. Agent-Based SCA

**Runs on**: All triggers, in parallel

**Purpose**: Software Composition Analysis for vulnerabilities in third-party dependencies.

| Step | Description |
|------|-------------|
| Check token | If `srcclr-api-token` credential is absent, skip the stage gracefully |
| Run SCA | `curl ... | bash -s -- scan --recursive --update-advisor` |

**Why always run SCA**: About 80% of vulnerabilities come from dependencies. SCA complements SAST by analyzing third-party code on every build.

---

### 3. Pipeline Scan (feature)

**Runs on**: Pushes to `feature/*` branches (excluding change requests)

**Purpose**: Fast feedback during active development. No gate.

| Step | Description |
|------|-------------|
| Unstash bundle | Retrieve `verascan/` and `artifact_list.txt` |
| Download Scanner | Get `pipeline-scan.jar` from `downloads.veracode.com` |
| Scan each artifact | Loop through `artifact_list.txt`; scan each one |
| Save Results | `scan_results/${ARTIFACT_NAME}_results.json` |
| Archive | Save results as Jenkins build artifacts |

**Gate**: None (`--fail_on_severity ""`)

**Why no gate**: Developers need fast, non-blocking feedback during active development. This enables rapid iteration on secure code.

---

### 4. Pipeline Scan (PR Gate)

**Runs on**: Change requests (`changeRequest()` matches PR/MR from GitHub, Bitbucket, GitLab, etc.)

**Purpose**: Security gate on pull requests.

| Step | Description |
|------|-------------|
| Unstash bundle | Retrieve `verascan/` and `artifact_list.txt` |
| Download Scanner | Get `pipeline-scan.jar` |
| Scan with Gate | `--policy_name "Veracode Recommended Very High"` |
| Track Failures | Collect failing artifacts; report at end |
| Save Results | `scan_results/${ARTIFACT_NAME}_results.json` |
| Exit Code | Non-zero if any artifact fails policy |

**Gate**: Fails on Very High and High severity findings

**Why gate PRs**: Prevents vulnerabilities from entering the main codebase. Integrates security into code review.

---

### 5. Sandbox Scan (release)

**Runs on**: Pushes to `release/*` branches

**Purpose**: Full SAST analysis in an isolated sandbox environment.

| Step | Description |
|------|-------------|
| Unstash bundle | Retrieve `verascan/` and `app_name.txt` |
| Download API Wrapper | Get the latest `vosp-api-wrappers-java` from Maven Central |
| Upload to Sandbox | `-filepath "verascan"` (directory upload, no zip required) |

**Sandbox Name**: `jenkins-release` (configurable)

**Why sandbox for releases**: Full validation without affecting production metrics. Allows safe experimentation with new features.

---

### 6. Policy Scan (main)

**Runs on**: Pushes to the `main` branch

**Purpose**: Production certification for compliance.

| Step | Description |
|------|-------------|
| Unstash bundle | Retrieve `verascan/` and `app_name.txt` |
| Download API Wrapper | Get the latest version dynamically |
| Upload to Platform | `-filepath "verascan"` for policy evaluation |

**Why policy scan on main**: Official security certification for code in production. Provides traceability for regulations such as SOC2, PCI-DSS, etc.

---

## Application Profile Names

The pipeline automatically builds the Veracode application profile name:

**Default format**: `JOB_NAME` (multibranch: `org/repo/branch`)

| Jenkins Job | Veracode App Name (default) |
|-------------|------------------------------|
| `acme-corp/api-service/main` | `acme-corp/api-service/main` |
| `myorg/frontend/feature%2Fauth` | `myorg/frontend/feature%2Fauth` |

For most teams the branch suffix in the default is undesirable. Set `VERACODE_APP_NAME` to a stable value (for example `acme-corp/api-service`) at the folder or job level so all branches scan against one profile.

---

## Multi-Artifact Handling

The pipeline handles repositories with multiple deployable artifacts.

### Pipeline Scans (feature / PR)

Each artifact is scanned individually:

```text
verascan/
  backend-api.jar    -> scanned separately
  frontend.zip       -> scanned separately
  common-lib.jar     -> scanned separately

scan_results/
  backend-api.jar_results.json
  frontend.zip_results.json
  common-lib.jar_results.json
```

### Platform Scans (release / main)

All artifacts are uploaded together via directory upload:

```text
-filepath "verascan"    # API Wrapper handles multi-file upload natively
```

**Note**: No zip bundle is created. The Java API Wrapper accepts a directory path directly, which avoids zip bomb rejections.

---

## Scan Results

### Pipeline Scan Results

Results are saved per artifact in `scan_results/` and archived as Jenkins build artifacts:

```text
scan_results/
  backend-api.jar_results.json
  frontend.zip_results.json
  common-lib.jar_results.json
```

**View in Jenkins**: Open the build > **Build Artifacts** > `scan_results/`

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
3. View findings, compliance status, and trends

---

## Customization

### Change PR Gate Policy

In the `Pipeline Scan (PR Gate)` stage:

```groovy
--policy_name "Your Custom Policy"
```

### Change Severity Gate

```bash
--fail_on_severity "Very High"           # Fail only on Very High
--fail_on_severity "Very High, High"     # Fail on Very High and High (default)
--fail_on_severity ""                    # No gate (informational only)
```

### Custom Sandbox Name

In the `Sandbox Scan (release)` stage:

```groovy
-sandboxname "your-sandbox-name"
```

### Add a Build Step

If the autopackager does not work for your project, add a build stage before `Package Artifacts`:

```groovy
stage('Build') {
    steps {
        sh '''
            # Maven
            mvn -B clean package -DskipTests

            # Gradle
            ./gradlew build -x test

            # npm
            npm ci && npm run build
        '''
    }
}
```

### Change Trigger Branches

Modify the `when {}` directives:

```groovy
branch pattern: 'develop|release/*|feature/*', comparator: 'REGEXP'
```

### Pin Agent Label

The pipeline uses `agent { label 'linux' }`. Adjust to match your Jenkins agents:

```groovy
agent { label 'docker-linux' }
// or
agent any
```

---

## Pipeline Scan Parameters

| Parameter | Description | Used In |
|-----------|-------------|---------|
| `-f` | File to scan | All pipeline scans |
| `-vid` | Veracode API ID | All scans |
| `-vkey` | Veracode API key | All scans |
| `--fail_on_severity` | Severities that fail the build | Feature (empty), PR (Very High, High) |
| `--policy_name` | Policy to evaluate against | PR scans |
| `--issue_details` | Include detailed finding info | All scans |
| `-jo` | JSON output only | All scans |

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

## Alternative: Veracode Jenkins Plugin

This pipeline uses the Veracode CLI, the Pipeline Scanner JAR, and the Java API Wrapper directly. This keeps the pipeline portable and avoids a hard dependency on the Veracode Jenkins plugin.

If you prefer the official Veracode Jenkins plugin, the platform scan stages can be replaced with the `veracode` step:

```groovy
veracode(
    applicationName: env.VERACODE_APP_NAME_RESOLVED,
    criticality: 'High',
    sandboxName: 'jenkins-release',
    createSandbox: true,
    scanName: "Release ${BUILD_NUMBER}",
    waitForScan: false,
    timeout: 60,
    createProfile: true,
    teams: '',
    canFailJob: true,
    debug: false,
    uploadIncludesPattern: 'verascan/**/*',
    vid: "${VERACODE_API_ID}",
    vkey: "${VERACODE_API_KEY}"
)
```

See the [Veracode Scan plugin reference](https://www.jenkins.io/doc/pipeline/steps/veracode-scan/) for the complete parameter list.

---

## Troubleshooting

### No Artifacts Found

**Symptoms**: Package stage fails with "No packaged artifacts found"

**Solutions**:
1. Make sure the project builds successfully before packaging
2. Verify compiled artifacts exist in expected locations
3. Review Veracode CLI autopackager output for errors
4. Add an explicit build stage before the autopackager

### Pipeline Scan Produces No Results

**Symptoms**: `results.json` is not created for an artifact

**Causes**:
- Artifact is not scannable (test JARs, resource bundles)
- Artifact does not contain application code
- Unsupported module type

**Solutions**:
1. Review the Pipeline Scanner output for warnings
2. Verify the artifact contains application code
3. This is normal for some artifacts; the pipeline continues

### Policy Gate Fails Unexpectedly

**Symptoms**: PR pipeline scan fails with no obvious issues

**Solutions**:
1. Verify the policy name matches exactly (case-sensitive)
2. Confirm the policy exists in your Veracode account
3. Review `results.json` for specific findings
4. Verify the policy thresholds

### Change Request Stage Does Not Trigger

**Symptoms**: PR / MR builds do not run the gated stage

**Cause**: `changeRequest()` only fires inside a **Multibranch Pipeline** job. A classic Pipeline job does not provide `CHANGE_ID`.

**Solutions**:
1. Convert the job to a Multibranch Pipeline
2. For GitHub: use the **GitHub Branch Source** plugin
3. For Bitbucket: use the **Bitbucket Branch Source** plugin
4. For GitLab: use the **GitLab Branch Source** plugin

### Sandbox / Policy Upload Fails

**Symptoms**: API Wrapper fails during upload

**Solutions**:
1. Verify that `veracode-api-id` and `veracode-api-key` credentials are configured
2. Verify the credentials have upload permissions
3. Ensure the application profile name is valid (no special characters)
4. Review API Wrapper output for errors

### `unstash` Fails in Parallel Stages

**Symptoms**: One of the parallel scan stages fails with "No such saved stash"

**Cause**: The `Package Artifacts` stage failed before `stash` ran (in `post { success }`).

**Solutions**:
1. Inspect the Package stage logs
2. Confirm the autopackager produced at least one `.war` / `.jar` / `.zip`
3. Add an explicit build step if needed

---

## Files

| File | Description |
|------|-------------|
| `Jenkinsfile` | Declarative pipeline definition |
| `veracode-strategy.md` | This documentation |

---

## Quick Start

1. Copy `Jenkinsfile` to the root of your repository
2. Create a **Multibranch Pipeline** job in Jenkins pointing at your repository
3. Add credentials in **Manage Jenkins > Credentials**:
   - `veracode-api-id`
   - `veracode-api-key`
   - `srcclr-api-token` (optional)
4. (Optional) Set `VERACODE_APP_NAME` at the folder or job level
5. Push to a `feature/*` branch or open a PR to trigger the first scan
6. Review results in **Build Artifacts > scan_results/** (Pipeline Scan) or in the Veracode Platform (Sandbox / Policy)

---

## Best Practices

**Shift-Left**:
- Run Pipeline Scan on every commit
- Enable gates on PRs to block High and Very High findings
- Educate the team on common findings

**Compliance**:
- Require Policy Scan before production
- Maintain scan history
- Document exceptions and mitigations

**Optimization**:
- Use Pipeline Scan for fast iteration
- Reserve Policy Scan for official releases
- Cache dependencies on the agent to speed up builds

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
- [Veracode Jenkins Plugin](https://plugins.jenkins.io/veracode-scan/)
- [Jenkins Declarative Pipeline Syntax](https://www.jenkins.io/doc/book/pipeline/syntax/)
