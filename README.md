# Veracode CI/CD Pipeline Examples

Repository that provides example CI/CD pipeline configurations for Veracode integration. Use as reference implementations and adapt to your needs.

---

## Repository Structure

```
.
├── aws/                          
│   ├── english/                     # Coming soon
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
├── bitbucket/
│   ├── english/
│   │   ├── bitbucket-pipelines.yml
│   │   └── veracode-strategy.md
│   └── spanish/
│       ├── bitbucket-pipelines.yml
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
├── gitlab-ci/
│   ├── english/
│   │   ├── .gitlab-ci.yml
│   │   └── veracode-strategy.md
│   └── spanish/
│       ├── .gitlab-ci.yml
│       └── veracode-strategy.md
│
└── jenkins/                      # Coming soon
```

---

## What's Included

Each implementation provides:

- **Pipeline Configuration**: Complete CI/CD pipeline file ready to use
- **Security Strategy Guide**: When and why to use each Veracode scan type
- **Multi-technology Support**: Java, .NET, Node.js, Python, Go, PHP, Ruby, Scala, and more
- **Bilingual Documentation**: Available in English and Spanish

---

## Available Implementations

| Platform | Status | English | Spanish |
|----------|--------|---------|---------|
| Azure DevOps | Available | Yes | Yes |
| Bitbucket Pipelines | Available | Yes | Yes |
| GitHub Actions | Available | Yes | Yes |
| GitLab CI/CD | Available | Yes | Yes |
| AWS CodeBuild | 1/2 Available | - | Yes |
| Jenkins | Coming soon | - | - |

---

## Veracode Scan Strategy

| Branch Type | Scan Type | Duration | Purpose |
|-------------|-----------|----------|---------|
| `feature/*` | Pipeline Scan | 3-10 min | Fast developer feedback |
| Pull Requests / Merge Requests | Pipeline Scan + Gate | 3-10 min | Block vulnerable code |
| `release/*` | Sandbox Scan | 30-90 min | Pre-production validation |
| `main` | Policy Scan | 45-120 min | Production certification |
| All | SCA | Varies | Third-party dependency analysis |

The same strategy applies across all supported platforms. Branch and trigger conventions are translated to each platform's native syntax (for example, `pull_request` in GitHub Actions, `merge_request_event` in GitLab CI, `pull-requests` in Bitbucket Pipelines).

---

## Prerequisites

- Veracode account with API credentials ([Get them here](https://docs.veracode.com/r/t_create_api_creds))
- CI/CD platform (Azure DevOps, Bitbucket, GitHub Actions, GitLab, etc.)
- Supported application (Java, .NET, Node.js, Python, Go, etc.)

---

## Documentation

Each folder contains:

- Pipeline configuration file
- `veracode-strategy.md`: Detailed explanation of scan strategy and Veracode products used

**Official Veracode Docs**:

- [Veracode CLI](https://docs.veracode.com/r/Install_the_Veracode_CLI)
- [Pipeline Scan](https://docs.veracode.com/r/Pipeline_Scan)
- [SCA Agent-Based](https://docs.veracode.com/r/Agent_Based_Scans)
- [Veracode Java API Wrapper](https://docs.veracode.com/r/c_about_wrappers)

**Platform-Specific Veracode Docs**:

- [Azure DevOps Pipeline Scan Examples](https://docs.veracode.com/r/Azure_DevOps_Pipeline_Scan_Examples)
- [Bitbucket Pipeline Scan Examples](https://docs.veracode.com/r/Bitbucket_Pipeline_Scan_Examples)
- [GitHub Actions Pipeline Scan Examples](https://docs.veracode.com/r/Github_Pipeline_Scan_Examples)
- [GitLab Pipeline Scan Examples](https://docs.veracode.com/r/Gitlab_Pipeline_Scan_Examples)
