# GitHub Actions Runner

Self-hosted GitHub Actions runner deployed on Kubernetes.

## Overview

The GitHub Actions runner executes workflows from remote GitHub repositories (e.g., `shared-pipeline`) on the local Kubernetes cluster.

## Architecture

- **Image**: `ubuntu:22.04` (runner binary downloaded at runtime)
- **Namespace**: `github-runner`
- **Executor**: Runs as a pod with persistent runner process
- **Dependencies**: Includes ICU libraries for .NET Core support

## Setup

### Prerequisites

1. GitHub repository with Actions enabled
2. Registration token from GitHub (Settings → Actions → Runners → New self-hosted runner)

### Deployment

The runner is deployed via ArgoCD. Manual deployment (if needed):

```bash
# Create Kubernetes secret with registration token
kubectl create secret generic github-runner-secret \
  --from-literal=token=YOUR_GITHUB_REGISTRATION_TOKEN \
  -n github-runner

# Deploy runner
kubectl apply -k cicd/github-runner/
```

### Configuration

**ConfigMap** (`cicd/github-runner/configmap.yaml`):
- `REPO_URL`: GitHub repository URL
- `RUNNER_NAME`: Runner name in GitHub
- `RUNNER_LABELS`: Tags for job matching (e.g., `self-hosted,linux,k8s`)

### Using the Runner

In your GitHub Actions workflow, specify:

```yaml
jobs:
  my-job:
    runs-on: self-hosted
    steps:
      - uses: actions/checkout@v3
      # ... your workflow steps
```

## Verification

**Check Runner Status:**
```bash
# Check pod status
kubectl get pods -n github-runner

# Check runner logs
kubectl logs -n github-runner -l app=github-runner -f
```

**Verify in GitHub:**
1. Go to repository: Settings → Actions → Runners
2. Runner should appear as "Online" (green)

## Troubleshooting

### Runner Not Picking Up Jobs

1. Verify runner is online in GitHub UI
2. Check workflow uses `runs-on: self-hosted`
3. Verify runner labels match workflow tags

### Permission Issues

The runner runs as a non-root user (`runner`) for security. Ensure any required permissions are available.

### .NET Core Dependencies

The runner includes `libicu70` for .NET Core support. If you need additional dependencies, modify the `deployment.yaml` installation script.

## GitOps Management

Managed by ArgoCD via:
- Application manifest: `cicd/argoCD/applications/github-runner-app.yaml`
- Resource directory: `cicd/github-runner/`

Changes to runner configuration are automatically synced by ArgoCD.

