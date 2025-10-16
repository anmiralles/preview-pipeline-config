# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This repository contains GitOps configuration for automated preview environments on Kubernetes using ArgoCD, Tekton, and Helm. It enables ephemeral preview environments that are automatically created when merge requests are opened and destroyed when they are merged.

## Architecture Overview

### Two-Repository Model
- **App Source Repository**: Contains application code where feature branches and merge requests are created
- **Config Repository (this repo)**: Contains reusable Helm charts and ArgoCD ApplicationSet definitions that provision preview environments

### Core Components

1. **ArgoCD ApplicationSet** (`argocd/my-quarkus-demo-preview-appset.yaml`)
   - Uses GitLab Pull Request generator to monitor merge requests
   - Automatically creates/destroys Application resources based on MR state
   - Each preview environment gets a dedicated namespace: `my-quarkus-demo-app-pr-{number}`
   - References Helm charts from this repo to deploy preview environments

2. **Helm Charts** (two separate charts)
   - **Application Chart** (`helm/app/`): Deploys the application to preview environments
     - Creates Deployment, Service, Route, ServiceAccount, RoleBinding, ExternalSecret
     - Parameterized by namespace, environment, and image tag
   - **CI/Build Chart** (`helm/ci/build/`): Defines Tekton pipelines and triggers
     - EventListener with triggers for merge requests, commits, tags, and releases
     - TriggerTemplates and TriggerBindings for different webhook events
     - Pipeline definitions for building, scanning, and signing container images

3. **Tekton Pipeline Workflow**
   - **Merge Request Opened**: Triggers `my-quarkus-demo-preview-app-build` pipeline
     - Tags images as `pr-{number}` for preview environments
     - Runs git-clone → package (Maven) → scan-source (SonarQube) → build-sign-image (Buildah)
   - **Merge Request Merged**: Triggers main build pipeline
     - Tags images as `latest` for development environment
   - **Tag Push/Release**: Triggers promotion pipelines for pre-prod/prod

## Key Configuration Patterns

### Preview Environment Naming
- ArgoCD Application: `my-quarkus-demo-app-pr-{number}`
- Namespace: `my-quarkus-demo-app-pr-{number}`
- Image tag: `pr-{number}`

### Webhook Filtering (EventListener)
The EventListener uses CEL interceptors to route GitLab webhooks:
- `body.object_kind == 'merge_request' && body.object_attributes.state == 'opened'` → Preview build
- `body.object_kind == 'merge_request' && body.object_attributes.state == 'merged'` → Main build
- `body.object_kind == 'tag_push'` → Pre-prod promotion
- `body.object_kind == 'release'` → Prod promotion

### ArgoCD Sync Policy
Preview environments use:
- `CreateNamespace=true`: Auto-create namespaces
- `prune=true` and `PruneLast=true`: Auto-cleanup when MR is merged/closed
- `selfHeal=true`: Auto-correct drift

## Environment-Specific Values

Current configuration references a specific OpenShift cluster:
- Cluster: `cluster-2kpzz.2kpzz.sandbox5399.opentlc.com`
- GitLab: `gitlab-gitlab.apps.cluster-2kpzz.2kpzz.sandbox5399.opentlc.com`
- Quay Registry: `quay-2kpzz.apps.cluster-2kpzz.2kpzz.sandbox5399.opentlc.com`
- SonarQube, Rekor, TUF, Keycloak, and StackRox endpoints are cluster-specific

When adapting this for different environments, update these values in:
- `argocd/my-quarkus-demo-preview-appset.yaml`
- `helm/ci/build/values.yaml`
- `helm/app/values.yaml`
- Tekton TriggerTemplate parameters

## Working with This Repository

### Modifying Preview Environment Behavior

To change what gets deployed in preview environments:
- Edit templates in `helm/app/templates/`
- Modify default values in `helm/app/values.yaml`
- The ApplicationSet will automatically apply changes to new preview environments

### Modifying Build Pipeline

To change the CI/CD pipeline:
- Edit pipeline definition in `helm/ci/build/templates/my-quarkus-demo-preview-app-build-p.yaml`
- Modify trigger templates in `helm/ci/build/templates/*-tt.yaml`
- Update EventListener filters in `helm/ci/build/templates/my-quarkus-demo-app-el.yaml`

### Testing Helm Charts

To validate Helm chart syntax:
```bash
helm template helm/app --values helm/app/values.yaml
helm template helm/ci/build --values helm/ci/build/values.yaml
```

To test with specific parameters (simulating a preview environment):
```bash
helm template helm/app \
  --set namespace.name=my-quarkus-demo-app-pr-42 \
  --set environment=dev \
  --set image.tag=pr-42
```

### Validating ArgoCD ApplicationSet

To check ApplicationSet syntax:
```bash
kubectl apply --dry-run=client -f argocd/my-quarkus-demo-preview-appset.yaml
```

## Security Integrations

The pipeline includes multiple security scanning stages:
- **SonarQube**: Static code analysis
- **CycloneDX**: SBOM generation and storage
- **StackRox**: Container image scanning
- **Sigstore (Rekor + TUF)**: Image signing and verification
- **Keycloak OIDC**: Identity provider for signing operations

All security tool endpoints and credentials are parameterized through pipeline parameters and Kubernetes secrets.