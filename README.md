# Veracode CI/CD Pipeline Examples

Reference CI/CD pipelines for Veracode Static Analysis and SCA. Every platform implements the same scan strategy in its native syntax. Adapt branch names, policies, and build steps to your project.

---

## Scan Strategy

| Trigger | Scan | Gate | Purpose |
|---------|------|------|---------|
| Push to `feature/**` | Pipeline Scan | Fails on any flaw | Fast developer feedback |
| Pull/Merge Request to default branch | Pipeline Scan | `Veracode Recommended Very High` policy | Prove the change is safe to merge |
| Push to default branch | Policy Scan (Upload and Scan) | Platform policy | Compliance record in the Veracode Platform |
| All of the above | Agent-Based SCA | Non-blocking | Third-party dependency analysis |

Pipeline Scan exit codes: `0` no flaws, `1-200` flaws matching the criteria, `253-255` timeout or error ([status codes](https://docs.veracode.com/r/Pipeline_Scan_Status_Codes)).

Common flow on every platform:

1. **Package**: Veracode CLI autopackager (`veracode package --trust`) builds scannable artifacts into `verascan/`.
2. **Pipeline Scan**: each artifact scanned separately, results kept, job fails if any artifact fails.
3. **Policy Scan**: Java API Wrapper `UploadAndScan` uploads the whole `verascan/` folder as one build.

---

## Available Implementations

| Platform | Folder | English | Spanish |
|----------|--------|---------|---------|
| GitHub Actions | [`github-actions/`](./github-actions) | Yes | Yes |
| GitLab CI/CD | [`gitlab/`](./gitlab) | Yes | Yes |
| Bitbucket Pipelines | [`bitbucket/`](./bitbucket) | Yes | Yes |
| Azure DevOps | [`ado/`](./ado) | Yes | Yes |
| AWS CodeBuild | [`aws/`](./aws) | Yes | Yes |
| Jenkins (Linux, Windows) | [`jenkins/`](./jenkins) | Yes | Yes |

Each language folder contains the pipeline file and a `veracode-strategy.md` with setup, job details, customization, and troubleshooting.

---

## Prerequisites

- Veracode API credentials ([create them](https://docs.veracode.com/r/t_create_api_creds))
- SCA agent token for dependency scanning ([create an agent](https://docs.veracode.com/r/t_sc_cli_agent))
- Java 8+ and `curl`/`unzip` on the runner (Pipeline Scan and API Wrapper are Java)
- A project type supported by [autopackaging](https://docs.veracode.com/r/About_auto_packaging), or your own build step

---

## Official Veracode Docs

- [Veracode CLI](https://docs.veracode.com/r/Install_the_Veracode_CLI)
- [veracode package](https://docs.veracode.com/r/veracode_package)
- [Pipeline Scan parameters](https://docs.veracode.com/r/r_pipeline_scan_commands)
- [Java API Wrapper](https://docs.veracode.com/r/c_about_wrappers)
- [Agent-Based SCA](https://docs.veracode.com/r/Agent_Based_Scans)
