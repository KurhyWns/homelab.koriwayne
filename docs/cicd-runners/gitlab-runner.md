# GitLab CI Runner

Self-hosted GitLab CI runner deployed on Kubernetes.

## Overview

The GitLab CI runner executes pipelines from the local GitLab instance (`192.168.1.244`) on the Kubernetes cluster using the Kubernetes executor.

## Architecture

- **Image**: `gitlab/gitlab-runner:latest`
- **Namespace**: `gitlab-runner`
- **Executor**: `kubernetes` (jobs run as Kubernetes pods)
- **Service Account**: `gitlab-runner` with RBAC permissions

## Setup

### Prerequisites

1. GitLab instance running (local at `192.168.1.244`)
2. Registration token from GitLab:
   - **Instance-level**: Admin Area → Runners → New instance runner
   - **Project-level**: Project Settings → CI/CD → Runners → Expand

### Deployment

The runner is deployed via ArgoCD. Manual deployment (if needed):

```bash
# Update ConfigMap with registration token
# Edit: cicd/gitlab-runner/configmap.yaml
# Set: token = "YOUR_REGISTRATION_TOKEN"

# Create GitLab CA certificate secret (if using self-signed cert)
kubectl create secret generic gitlab-ca-cert \
  --from-file=ca.crt=/path/to/gitlab-ca.crt \
  -n gitlab-runner

# Deploy runner
kubectl apply -k cicd/gitlab-runner/
```

### Configuration

**ConfigMap** (`cicd/gitlab-runner/configmap.yaml`):
```toml
concurrent = 4
check_interval = 3

[[runners]]
  name = "local-k8-runner"
  url = "https://192.168.1.244"
  token = "YOUR_REGISTRATION_TOKEN"
  executor = "kubernetes"
  tags = ["k8s"]
  environment = ["GIT_SSL_NO_VERIFY=true"]
  
  [runners.kubernetes]
    namespace = "gitlab-runner"
    service_account = "gitlab-runner"
    image = "ubuntu:22.04"
    privileged = true
```

**RBAC Resources:**
- ServiceAccount: `gitlab-runner`
- Role: Permissions for pods, secrets, configmaps
- RoleBinding: Binds role to service account

### SSL Certificate Handling

For self-signed GitLab certificates, the runner:
1. Mounts GitLab CA cert from Kubernetes Secret
2. Combines with system CA bundle
3. Sets `SSL_CERT_FILE` environment variable for Go's HTTP client

## Using the Runner

In your `.gitlab-ci.yml`, tag jobs:

```yaml
my-job:
  stage: build
  tags:
    - k8s
  script:
    - echo "Running on self-hosted runner"
```

## Verification

**Check Runner Status:**
```bash
# Check pod status
kubectl get pods -n gitlab-runner

# Check runner logs
kubectl logs -n gitlab-runner -l app=gitlab-runner -f
```

**Verify in GitLab:**
1. Go to: `https://192.168.1.244`
2. Navigate: Admin Area → Runners (instance) or Project Settings → CI/CD → Runners (project)
3. Runner should appear as "Online"

## Troubleshooting

### Certificate Errors

If you see `certificate signed by unknown authority`:
1. Ensure GitLab CA cert Secret exists: `kubectl get secret gitlab-ca-cert -n gitlab-runner`
2. Verify cert is mounted correctly in deployment
3. Check `SSL_CERT_FILE` is set in pod environment

### Permission Errors

If jobs fail with RBAC errors:
1. Verify ServiceAccount exists: `kubectl get sa gitlab-runner -n gitlab-runner`
2. Check Role and RoleBinding: `kubectl get role,rolebinding -n gitlab-runner`
3. Ensure Role has `update` and `patch` permissions for secrets

### Jobs Not Picking Up

1. Verify runner tags match job tags in `.gitlab-ci.yml`
2. Check runner is enabled for the project (project runners only)
3. Ensure runner is online in GitLab UI

## GitOps Management

Managed by ArgoCD via:
- Application manifest: `cicd/argoCD/applications/gitlab-runner-app.yaml`
- Resource directory: `cicd/gitlab-runner/`

Changes to runner configuration are automatically synced by ArgoCD.

## Pipeline Integration

Example `.gitlab-ci.yml` using the runner:

```yaml
stages:
  - test
  - build
  - deploy

test-job:
  stage: test
  image: ubuntu:22.04
  tags:
    - k8s
  script:
    - echo "Running tests"
```

