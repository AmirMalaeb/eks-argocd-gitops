# EKS Managed Argo CD — GitOps POC

GitOps platform using AWS Managed Argo CD on Amazon EKS in a hub-and-spoke topology. A single management cluster runs the Argo CD control plane and manages deployments to spoke clusters across dev, staging, and production environments. The [AWS Retail Store Sample Application](https://github.com/aws-containers/retail-store-sample-app) serves as the demonstration workload.

## Architecture Overview

```
┌──────────────────────────────────────────────────────────┐
│                  Management Cluster (EKS)                │
│  ┌─────────────────────────────────────────────────────┐ │
│  │  AWS Managed Argo CD                                │ │
│  │  ├── Root App-of-Apps                               │ │
│  │  ├── ApplicationSets (retail-store, platform)       │ │
│  │  └── Notifications → Slack                          │ │
│  └─────────────────────────────────────────────────────┘ │
└──────────────┬──────────────┬──────────────┬─────────────┘
               │              │              │
       ┌───────▼──┐   ┌──────▼───┐   ┌──────▼──────┐
       │ Spoke:Dev │   │Spoke:Stg │   │ Spoke:Prod  │
       │  1 replica│   │ 2 replica│   │ 3+ replicas │
       │  relaxed  │   │ moderate │   │ strict      │
       │  policies │   │ policies │   │ policies    │
       └──────────┘   └──────────┘   └─────────────┘
```

The management cluster hosts only the Argo CD control plane. Spoke clusters receive all application and platform component deployments. Cross-cluster authentication uses IAM roles for service accounts (IRSA).

## Repository Layout

```
gitops-repo/
├── README.md                    # This file
├── promotion-history.yaml       # Audit log of environment promotions
├── apps/                        # Argo CD Application and ApplicationSet manifests
│   ├── applicationsets/         # ApplicationSet definitions (retail-store, platform)
│   └── projects/                # AppProject definitions (RBAC boundaries)
├── charts/                      # Base Helm chart references and values
│   └── retail-store/            # Retail Store app Helm chart + base values
├── overlays/                    # Kustomize overlays per environment
│   ├── dev/                     # Dev: 1 replica, minimal resources
│   ├── staging/                 # Staging: 2 replicas, moderate resources
│   └── production/              # Production: 3+ replicas, strict policies
├── platform/                    # Platform component manifests (base/overlays)
│   ├── kyverno/                 # Policy engine — cluster guardrails
│   ├── external-secrets/        # ESO — secrets from AWS Secrets Manager
│   ├── argo-rollouts/           # Progressive delivery controller
│   ├── prometheus/              # Metrics collection
│   └── grafana/                 # Dashboards and visualization
├── rollouts/                    # Argo Rollouts resources (canary, blue-green)
│   ├── base/                    # Rollout definitions + AnalysisTemplates
│   └── overlays/                # Per-environment rollout config
└── .github/
    └── workflows/               # CI pipelines (image + environment promotion)
```

| Directory | Purpose |
|-----------|---------|
| `apps/` | Argo CD Application, ApplicationSet, and AppProject manifests. The root App-of-Apps points here. |
| `charts/` | Base Helm chart values shared across all environments. |
| `overlays/` | Kustomize overlays for per-environment customization (replicas, resources, image tags, secrets). |
| `platform/` | Platform components deployed to every spoke cluster. Each follows a base/overlays pattern. |
| `rollouts/` | Argo Rollouts resources — canary (UI) and blue-green (Catalog) strategies with AnalysisTemplates. |
| `.github/workflows/` | GitHub Actions CI pipelines for image tag promotion and environment promotion. |

## Prerequisites

- **AWS Account** with permissions to create EKS clusters, IAM roles, and Secrets Manager entries
- **EKS Clusters** provisioned via the Terraform modules in `terraform/` (1 management + 3 spoke clusters)
- **AWS Managed Argo CD** enabled on the management cluster
- **GitHub Repository** with Actions enabled and repository secrets configured
- **Slack Webhook URL** for deployment notifications (stored as a Kubernetes Secret, never in Git)
- **AWS CLI** v2 and `kubectl` configured with cluster contexts
- **Helm** v3.x and **Kustomize** v5.x installed locally for validation

## Quick Start

1. **Provision infrastructure**

   ```bash
   # Management cluster
   cd terraform/environments/management
   terraform init && terraform apply

   # Spoke clusters (repeat for each environment)
   cd terraform/environments/spoke-dev
   terraform init && terraform apply
   ```

2. **Bootstrap Argo CD**

   Apply the root App-of-Apps to the management cluster:

   ```bash
   kubectl apply -f apps/root-app.yaml --context <management-cluster-context>
   ```

   Argo CD will automatically sync and create all child Applications, ApplicationSets, and platform components.

3. **Verify deployment**

   ```bash
   # Check Argo CD Applications
   argocd app list

   # Check Retail Store services on a spoke cluster
   kubectl get pods -n retail-store-dev --context <dev-spoke-context>
   ```

4. **Promote an image tag**

   Trigger the GitHub Actions workflow:

   ```bash
   gh workflow run promote-image.yml \
     -f service_name=ui \
     -f image_tag=0.8.1 \
     -f target_environment=dev
   ```

## How to Add a New Environment

1. Create a new overlay directory:

   ```bash
   mkdir -p overlays/<env-name>/patches overlays/<env-name>/secrets
   ```

2. Add a `kustomization.yaml` and environment-specific patches (replicas, resources, image tags).

3. The ApplicationSet's Git generator will automatically detect the new overlay directory and generate an Argo CD Application for it on the next sync cycle.

4. Register the new spoke cluster in Argo CD via the `argocd-management` Terraform module.

## How to Add a New Application

1. Add the Helm chart reference and base values under `charts/<app-name>/`.
2. Create Kustomize overlays for each environment under `overlays/`.
3. Create an ApplicationSet in `apps/applicationsets/<app-name>.yaml` using the matrix generator.
4. Create an AppProject in `apps/projects/<app-name>-project.yaml` defining RBAC boundaries.
5. Commit and push — the root App-of-Apps will pick up the new manifests.

## How to Promote an Image Tag

### Single service promotion

Use the `promote-image` workflow to update a specific service's image tag in a target environment:

```bash
gh workflow run promote-image.yml \
  -f service_name=catalog \
  -f image_tag=0.9.0 \
  -f target_environment=staging
```

- **Dev**: direct commit to main branch
- **Staging/Production**: creates a pull request for review

### Environment promotion

Use the `promote-environment` workflow to promote all service image tags from one environment to the next:

```bash
gh workflow run promote-environment.yml \
  -f source_environment=dev \
  -f target_environment=staging
```

The workflow checks that all apps in the source environment are Healthy before proceeding. Each promotion event is recorded in `promotion-history.yaml`.

## Contribution Guidelines

1. **All changes go through Git.** Never modify cluster state directly — Argo CD will revert manual changes (self-healing is enabled).

2. **Branch strategy:**
   - Use feature branches for all changes.
   - Dev environment changes can merge directly to `main`.
   - Staging and production changes require pull request review.

3. **Validation before commit:**

   ```bash
   # Validate Helm + Kustomize rendering
   helm template retail-store charts/retail-store/ -f charts/retail-store/values.yaml > /tmp/base.yaml
   kustomize build overlays/dev

   # Validate Terraform
   cd terraform/modules/eks-cluster && terraform validate
   ```

4. **Production sync windows:** Production deployments are restricted to business hours (Mon–Fri 9 AM–5 PM). Syncs outside this window are blocked by Argo CD.

5. **Secrets:** Never commit plaintext secrets. Use ExternalSecret resources that reference AWS Secrets Manager paths.

6. **Policies:** All workloads must pass Kyverno policies (approved registries, resource limits, no privileged containers). Run `kyverno test` locally before pushing.
