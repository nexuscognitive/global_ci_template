# Global CI Template

Reusable GitHub Actions workflow templates called by NX1 application repos via `workflow_call`.

## Workflows

| File | Purpose |
|------|---------|
| `app_deployer.yml` | Deploy an NX1 app to Kubernetes |
| `build-push-acr.yml` | Build a container image, scan with Trivy, push to ACR |
| `build-with-trivy.yml` | Build and scan without pushing |
| `mirror-public-images-acr.yml` | Mirror public images into the NX1 ACR |

## Usage

Caller repos reference these as reusable workflows:

```yaml
jobs:
  deploy:
    uses: nexuscognitive/global_ci_template/.github/workflows/app_deployer.yml@main
    with:
      app_name: my-service
    secrets: inherit
```
