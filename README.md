# Veracode CI/CD Pipeline Examples

Repository that provides example CI/CD pipeline configurations for Veracode integration. Use as reference implementations and adapt to your needs.

---

## Repository Structure

```
.
├── aws/                    # Coming soon
│   ├── english/ 
│   └── spanish/
│       ├── aws-pipeline.yml
│       └── veracode-strategy.md
│
├── ado/
│   ├── english/
│   │   ├── azure-pipelines.yml
│   │   └── veracode-strategy.md
│   └── spanish/
│       ├── azure-pipelines.yml
│       └── veracode-strategy.md
│
├── github-actions/
│   ├── english/
│   │   ├── veracode-scans.yml
│   │   └── veracode-strategy.md
│   └── spanish/
│       ├── veracode-scans.yml
│       └── veracode-strategy.md
│
├── gitlab-ci/                # Coming soon
└── jenkins/                  # Coming soon
```

---

## What's Included

Each implementation provides:

- **Pipeline Configuration**: Complete CI/CD pipeline file ready to use
- **Security Strategy Guide**: When and why to use each Veracode scan type
- **Multi-technology Support**: Java, .NET, Node.js, Python, Go, PHP, Ruby, Scala, and more

---

## Veracode Scan Strategy

| Branch Type | Scan Type | Duration | Purpose |
|-------------|-----------|----------|---------|
| `feature/*` | Pipeline Scan | 3-10 min | Fast developer feedback |
| Pull Requests | Pipeline Scan + Gate | 3-10 min | Block vulnerable code |
| `release/*` | Sandbox Scan | 30-90 min | Pre-production validation |
| `main` | Policy Scan | 45-120 min | Production certification |
| All | SCA | Varies | Third-party dependency analysis |

---

## Quick Start

### AWS CodeBuild

1. Copy `aws/spanish/aws-pipeline.yml` to your repo as `buildspec.yml`
2. Configure Veracode API credentials in Secrets Manager or Parameter Store
3. Set `APP_NAME` and `SANDBOX_NAME` variables
4. Run via CodeBuild

### Azure Pipelines

1. Copy `azure/english/azure-pipelines.yml` (or `azure/spanish/`) to your repo root
2. Create a variable group `veracode-credentials` in Azure DevOps with `VERACODE_API_ID` and `VERACODE_API_KEY` as secrets
3. Set `APP_NAME` and `SANDBOX_NAME` in the variables section
4. Create a new pipeline in Azure DevOps pointing to `azure-pipelines.yml`

Both pipelines auto-detect the branch type and run the appropriate scan. No manual scan type selection needed.

---

## Prerequisites

- Veracode account with API credentials ([Get them here](https://docs.veracode.com/r/t_create_api_creds))
- CI/CD platform (AWS CodeBuild, Azure Pipelines, GitHub Actions, etc.)
- Supported application (Java, .NET, Node.js, Python, Go, etc.)

---

## Documentation

Each folder contains:
- Pipeline configuration file
- `veracode-strategy.md` - Detailed explanation of scan strategy and Veracode products used

**Official Veracode Docs**:
- [Veracode CLI](https://docs.veracode.com/r/Install_the_Veracode_CLI)
- [Pipeline Scan](https://docs.veracode.com/r/Pipeline_Scan)
- [SCA Agent-Based](https://docs.veracode.com/r/Agent_Based_Scans)
