# CI/CD Runners

Self-hosted CI/CD runners deployed on Kubernetes for both GitHub Actions and GitLab CI.

## Table of Contents

- [Overview](#overview)
- [GitHub Actions Runner](github-runner.md)
- [GitLab CI Runner](gitlab-runner.md)
- [Architecture](#architecture)
- [Management](#management)

## Overview

This homelab uses self-hosted runners deployed on Kubernetes to execute CI/CD pipelines:

- **GitHub Actions Runner**: Executes workflows from remote GitHub repositories
- **GitLab CI Runner**: Executes pipelines from the local GitLab instance

Both runners are:
- Deployed as Kubernetes pods
- Managed by ArgoCD via GitOps
- Configured with proper RBAC permissions
- Automatically synced with their respective Git repositories

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                    │
│                                                          │
│  ┌──────────────────┐      ┌──────────────────┐         │
│  │  GitHub Runner   │      │  GitLab Runner   │         │
│  │   (Pod-based)    │      │   (Pod-based)    │         │
│  │                  │      │                  │         │
│  │  Namespace:      │      │  Namespace:      │         │
│  │  github-runner   │      │  gitlab-runner   │         │
│  └──────────────────┘      └──────────────────┘         │
│         │                           │                     │
│         │                           │                     │
│         ▼                           ▼                     │
│  ┌─────────────────────────────────────────┐             │
│  │         ArgoCD (GitOps)                 │             │
│  │    Manages both runners via Git         │             │
│  └─────────────────────────────────────────┘             │
└─────────────────────────────────────────────────────────┘
         │                           │
         │                           │
         ▼                           ▼
  GitHub (Remote)            GitLab (Local)
```

## Management

Both runners are managed via GitOps through ArgoCD:

- Configuration stored in Git: `cicd/github-runner/` and `cicd/gitlab-runner/`
- Changes committed to Git → ArgoCD auto-syncs
- Application manifests in: `cicd/argoCD/applications/`

### Updating Runner Configuration

1. Edit manifests in the respective runner directory
2. Commit and push changes
3. ArgoCD automatically syncs the updates

### Verifying Runner Status

**GitHub Actions Runner:**
```bash
kubectl get pods -n github-runner
kubectl logs -n github-runner -l app=github-runner
```

**GitLab CI Runner:**
```bash
kubectl get pods -n gitlab-runner
kubectl logs -n gitlab-runner -l app=gitlab-runner
```

## Benefits

- **Cost Savings**: No per-minute charges for GitHub Actions
- **Performance**: Dedicated resources on local infrastructure
- **Control**: Full control over runner environment and security
- **Integration**: Seamless integration with GitOps workflows

