# Liferay Cloud Native GitOps Boilerplate

This repository contains the GitOps configuration for deploying Liferay DXP on AWS using ArgoCD and Crossplane.

## Repository Structure

```
.
├── charts/
│   ├── liferay-aws-infrastructure-provider/   # Crossplane XRDs, Compositions, Providers
│   └── liferay-aws-infrastructure/            # LiferayInfrastructure CR template
└── liferay/
    └── projects/
        └── default/
            ├── base/
            │   ├── infrastructure.yaml        # Base infrastructure config
            │   └── liferay.yaml               # Base Liferay config
            └── environments/
                └── dev/
                    ├── infrastructure.yaml    # Dev environment overrides
                    └── liferay.yaml           # Dev environment overrides
```

## How It Works

1. **ArgoCD** watches this repository for changes
2. When you modify `infrastructure.yaml`, ArgoCD creates a `LiferayInfrastructure` CR
3. **Crossplane** processes the CR and provisions AWS resources (RDS, S3, OpenSearch)
4. Crossplane creates a `managed-service-details` Kubernetes Secret with connection info
5. When you modify `liferay.yaml`, ArgoCD deploys the Liferay Helm chart
6. Liferay pods read the secret and connect to the managed services

## Prerequisites

- AWS account with appropriate permissions
- EKS cluster with:
  - ArgoCD installed
  - Crossplane installed
  - External Secrets Operator installed
- GitHub PAT stored in AWS Secrets Manager

## Quick Start

### 1. Store Git Credentials in AWS Secrets Manager

Create a secret named `liferay-cloud-native/gitops-repo-credentials`:

```json
{
  "git_machine_user_id": "your-github-username",
  "git_access_token": "your-github-pat"
}
```

### 2. Deploy GitOps Platform

```bash
cd cloud/terraform/aws/gitops/platform
terraform init
terraform apply
```

### 3. Deploy GitOps Resources

Edit `terraform.tfvars`:

```hcl
deployment_name            = "my-liferay"
liferay_git_repo_url       = "https://github.com/misaellr/cloud-native-gitops-boilerplate.git"
liferay_helm_chart_version = "0.1.5"

# Use published chart versions
infrastructure_helm_chart_config = {
  version = "0.1.2"
}
infrastructure_provider_helm_chart_config = {
  version = "0.1.2"
}
```

```bash
cd cloud/terraform/aws/gitops/resources
terraform init
terraform apply
```

### 4. Access ArgoCD

```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open https://localhost:8080 and login with `admin` / (password from above).

## Adding a New Environment

1. Create a new directory under `liferay/projects/default/environments/`:

```bash
mkdir -p liferay/projects/default/environments/prod
```

2. Add `infrastructure.yaml`:

```yaml
# liferay/projects/default/environments/prod/infrastructure.yaml
database:
  blue:
    instanceClass: db.r5.large
    storageGB: 100
search:
  instanceType: r5.large.search
  instanceCount: 3
```

3. Add `liferay.yaml`:

```yaml
# liferay/projects/default/environments/prod/liferay.yaml
liferay-default:
  replicaCount: 3
  resources:
    requests:
      cpu: 4000m
      memory: 8Gi
```

4. Commit and push. ArgoCD will automatically create the new environment.

## Configuration Reference

### infrastructure.yaml

| Field | Description | Default |
|-------|-------------|---------|
| `database.activeSlot` | Active database slot | `blue` |
| `database.blue.instanceClass` | RDS instance type | `db.t3.medium` |
| `database.blue.storageGB` | RDS storage size | `20` |
| `search.instanceType` | OpenSearch instance type | `t3.small.search` |
| `search.instanceCount` | OpenSearch node count | `2` |
| `storage.blue.enableVersioning` | S3 versioning | `true` |

### liferay.yaml

See the [Liferay Helm chart documentation](https://github.com/liferay/liferay-portal/tree/master/cloud/helm) for all available options.

## License

See LICENSE file.
