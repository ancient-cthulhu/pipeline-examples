# Veracode Security Strategy for Azure Pipelines

Automated security strategy that integrates multiple Veracode products into the SDLC via Azure Pipelines, balancing feedback speed with analysis depth based on development context.

**Supported Technologies**: Java (Maven/Gradle/Ant), .NET (Core/Framework), Node.js, Python, Go, PHP, Ruby, Scala, and more.

---

## Scan Strategy

| Context | Veracode Product | Duration | Gate | Purpose |
|---------|-----------------|----------|------|---------|
| Feature branches | Pipeline Scan | 3-10 min | No | Fast developer feedback |
| Pull Requests | Pipeline Scan | 3-10 min | Yes (Very High/High) | Block vulnerabilities before merge |
| Release branches | Sandbox Scan | 30-90 min | No | Isolated pre-production validation |
| Main branch | Policy Scan | 25-60 min | Optional | Production certification |

**All contexts**: SCA Agent-Based (dependency analysis)

---

## Pipeline Architecture

The pipeline is a single `azure-pipelines.yml` file with conditional stages. Branch detection is handled natively via `Build.SourceBranch` and `Build.Reason`, no manual overrides needed.

```
Stage 1: Package        (always)   - Veracode CLI autopackager
Stage 2: SCA            (always)   - Agent-Based SCA
Stage 3a: Pipeline Scan (feature)  - Fast SAST, no gate
Stage 3b: Pipeline Scan (PR)       - Fast SAST, gate on Very High/High
Stage 4: Sandbox Scan   (release)  - Full SAST in sandbox
Stage 5: Policy Scan    (main)     - Full SAST, production certification
```

Only one SAST stage runs per build depending on the branch context. SCA always runs in parallel.

---

## Veracode Products Used

### 1. Veracode CLI - Autopackager

Detects application type and technology automatically, compiles and packages code for analysis, generates scan-ready artifacts (JAR, WAR, ZIP, DLL, etc.).

**Why**: Eliminates manual per-technology configuration and guarantees consistent packaging.

**Command**: `veracode package --source . --output verascan --trust`

---

### 2. Pipeline Scan

Fast, incremental SAST analysis. Detects critical vulnerabilities in minutes and generates JSON reports with exact line of code.

**Why on feature branches**: Immediate feedback during active development without blocking workflow.

**Why on Pull Requests (with gate)**: Prevents vulnerabilities before merge. Blocks insecure code automatically and integrates security into code review.

**Basic command**:
```bash
java -jar pipeline-scan.jar \
  -f artifact.jar \
  -vid $VERACODE_API_ID \
  -vkey $VERACODE_API_KEY
```

**With gate**:
```bash
java -jar pipeline-scan.jar \
  -f artifact.zip \
  --fail_on_severity "Very High, High" \
  --policy "Veracode Recommended Very High"
```

---

### 3. Sandbox Scan

Full SAST analysis in an isolated environment. Comparison with previous scans and detailed reports without affecting the production policy profile.

**Why on release branches**: Exhaustive pre-production validation, test fixes without impacting official metrics.

**Usage**:
```bash
java -jar vosp-api-wrapper-java.jar \
  -action UploadAndScan \
  -sandboxname "release-candidate" \
  -createsandbox true \
  -filepath artifact.zip
```

---

### 4. Policy Scan

Exhaustive SAST analysis against corporate policies. Generates official security certification with complete history for compliance and auditing.

**Why on main branch**: Formal certification for production code, organizational security policy compliance, full traceability for regulations (SOC2, PCI-DSS, etc.).

**Usage**:
```bash
java -jar vosp-api-wrapper-java.jar \
  -action UploadAndScan \
  -appname "ProductionApp" \
  -autoscan true \
  -filepath artifact.jar \
  -version "v1.2.3"
```

---

### 5. SCA Agent-Based

Software Composition Analysis for third-party dependencies. Detects known CVEs, outdated libraries, and problematic licenses.

**Why on all scans**: ~80% of vulnerabilities are in dependencies. Detects supply chain risks and complements SAST with third-party code analysis.

**Usage**:
```bash
curl -sSL https://download.sourceclear.com/ci.sh | bash -s -- scan --recursive --update-advisor
```

---

## Security Flow

```
Commit -> Autopackager -> SCA + SAST (branch-dependent) -> Results
```

**Phase 1: Packaging** (Veracode CLI) - Detects technology automatically, compiles and packages code.

**Phase 2: Composition Analysis** (SCA) - Scans third-party dependencies, reports known CVEs.

**Phase 3: Static Analysis** (SAST) - Pipeline Scan for dev/PR, Sandbox Scan for release, Policy Scan for production.

---

## Azure Pipelines Configuration

### Variable Group Setup

Create a variable group named `veracode-credentials` in Azure DevOps:

1. Go to **Pipelines > Library > + Variable group**
2. Name: `veracode-credentials`
3. Add variables:

| Variable | Type | Source |
|----------|------|--------|
| `VERACODE_API_ID` | Secret | Veracode Platform |
| `VERACODE_API_KEY` | Secret | Veracode Platform |
| `SRCCLR_API_TOKEN` | Secret | Veracode Platform (optional) |

**Recommended**: Link to Azure Key Vault instead of storing secrets directly.

### Pipeline Variables

| Variable | Purpose | Example |
|----------|---------|---------|
| `APP_NAME` | Name in Veracode Platform | "MyApplication" |
| `SANDBOX_NAME` | Sandbox identifier | "release-candidate" |

### Branch Detection

The pipeline uses Azure Pipelines built-in variables for automatic scan selection:

| Variable | Usage |
|----------|-------|
| `Build.SourceBranch` | Full branch ref (e.g., `refs/heads/feature/xyz`) |
| `Build.Reason` | Trigger reason (`PullRequest`, `IndividualCI`, etc.) |
| `Build.BuildNumber` | Used as Veracode scan version identifier |

### Secret Variable Handling

Secret variables from variable groups are NOT macro-expanded inside `script:` blocks. The pipeline uses `env:` mappings to inject secrets as environment variables, then references them as `$VERACODE_API_ID` (shell env var) in the script body. Non-secret variables like `$(APP_NAME)` and `$(PACKAGED_FILE)` are macro-expanded normally.

### Pull Request Triggers (Azure Repos Git)

When using **Azure Repos Git**, Pull Request builds are **not triggered by the `pr:` block in YAML alone**.

To ensure the **PR Pipeline Scan (with gate)** runs correctly, configure a **Branch Policy with Build Validation** on the target branch (e.g. `main`):

1. Go to **Repos → Branches**
2. Select the target branch (`main`)
3. Open **Branch policies**
4. Under **Build validation**, add this pipeline
5. (Optional) Mark it as **Required** to block PR completion on failed security gates

Once the branch policy triggers the pipeline, the YAML logic uses  
`Build.Reason = PullRequest` to automatically run the **PR-gated Pipeline Scan** stage.

---

## Results and Reports

**Pipeline Scan generates**:
- `results.json`: All findings
- `filtered_results.json`: Findings per policy
- Exit code: 0 (pass) or 1 (fail)
- Published as Azure Pipeline Artifact

**Policy/Sandbox Scan generates**:
- Reports in Veracode Platform
- Downloadable PDF
- Dashboard with metrics and trends
- APIs for SIEM/ticket integration

**SCA generates**:
- List of vulnerable dependencies
- Risk scores per library
- Update recommendations
- License analysis

---

## Best Practices

**Shift-Left**: Run Pipeline Scan on every commit. Enable gates on PRs to block High/Very High vulnerabilities. Educate the team on common findings.

**Compliance**: Policy Scan mandatory before production. Maintain history of all scans. Document exceptions and mitigations.

**Optimization**: Use Pipeline Scan for fast iteration. Reserve Policy Scan for official releases. Cache dependencies to speed up builds.

**SCA**: Monitor new CVEs continuously. Update dependencies regularly. Review licenses before adopting libraries.

**Azure-specific**: Use variable groups linked to Key Vault for credential rotation. Add branch policies to require Pipeline Scan pass before PR completion. Use environments with approval gates for production Policy Scan stage.

---

## Support

- **Veracode Docs**: [docs.veracode.com](https://docs.veracode.com)
- **Veracode CLI**: [https://docs.veracode.com/r/Install_the_Veracode_CLI](https://docs.veracode.com/r/Install_the_Veracode_CLI)
- **Pipeline Scan**: [https://docs.veracode.com/r/Pipeline_Scan](https://docs.veracode.com/r/Pipeline_Scan)
- **SCA**: [https://docs.veracode.com/r/Agent_Based_Scans](https://docs.veracode.com/r/Agent_Based_Scans)
