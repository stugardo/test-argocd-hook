# Test ArgoCD Hook

This repository demonstrates how **ArgoCD hooks** ensure that Jobs are automatically recreated when their dependent ConfigMaps are updated.

## Overview

This test application includes:
- A **ConfigMap** that contains configuration data
- A **Job** that reads and displays the ConfigMap content
- **ArgoCD hooks** that manage the lifecycle of these resources

## Problem Statement

When deploying applications with ArgoCD, updating a ConfigMap doesn't automatically trigger the recreation of Jobs that depend on it. This can lead to Jobs running with stale configuration data.

## Solution

Use **ArgoCD hooks** to ensure proper resource management:
- `argocd.argoproj.io/hook: PostSync` ensures the Job is created after the application has been fully synced and deployed.
- `argocd.argoproj.io/hook-delete-policy: BeforeHookCreation` on the Job - deletes the old Job before creating a new one (the job should have the same name!)

## Prerequisites

- Kubernetes cluster (e.g., Kind)
- ArgoCD installed in the cluster

## Repository Structure

```
test-argocd-hook/
├── Chart.yaml              # Helm chart metadata
├── values.yaml             # ConfigMap values
└── templates/
    ├── configmap.yaml      # ConfigMap with PreSync hook
    └── job.yaml            # Job with Sync hook
```

## Deployment

### 1. Deploy the ArgoCD Application

Create an ArgoCD Application manifest (`argocd-application.yaml`):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: test-argocd-hook
  namespace: argocd
spec:
  project: default
  source:
    repoURL: <https:your-repo-url>.git
    targetRevision: HEAD
    path: .
  destination:
    server: https://kubernetes.default.svc
    namespace: test-hook
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

Apply the application:

```bash
kubectl apply -f argocd-application.yaml
```

### 2. Access ArgoCD Locally

Port-forward the ArgoCD server:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Login to ArgoCD:

```bash
argocd login localhost:8080
```

### 3. Sync the Application

```bash
argocd app sync test-argocd-hook
```

## Testing the ArgoCD Hook

### Initial Deployment

1. **Verify the ConfigMap:**

```bash
kubectl get configmap test-config -n test-hook -o yaml
```

2. **Verify the Job:**

```bash
kubectl get jobs -n test-hook
kubectl logs job/test-job -n test-hook
```

**Expected output:**
```
Reading ConfigMap...
Hello from ConfigMap v1
Job completed successfully!
```

### Update the ConfigMap

1. **Edit `values.yaml`:**

```yaml
config:
  message: "Hello from ConfigMap v2"
```

2. **Commit and push:**

```bash
git add values.yaml
git commit -m "Update ConfigMap to v2"
git push origin master
```

3. **Sync the application:**

```bash
argocd app sync test-argocd-hook
```

4. **Verify the Job was recreated:**

```bash
kubectl get jobs -n test-hook
kubectl logs job/test-job -n test-hook
```

**Expected output:**
```
Reading ConfigMap...
Hello from ConfigMap v2
Job completed successfully!
```

## How It Works

1. **PreSync Hook**: The ConfigMap is created before other resources
2. **Sync Hook**: The Job is created during the sync phase
3. **BeforeHookCreation Policy**: When syncing, ArgoCD deletes the old Job before creating a new one
4. **Result**: The Job always uses the latest ConfigMap values

## Benefits :) 

✅ No manual intervention required to recreate Jobs after ConfigMap updates
✅ Jobs always use the latest configuration   

## Notes

- The Job uses a fixed name (`test-job`) to allow ArgoCD to manage its lifecycle
- `ttlSecondsAfterFinished: 300` ensures completed Jobs are cleaned up after 5 minutes
- For Kind clusters, pre-load the busybox image:

```bash
kind load docker-image busybox:latest --name <cluster-name>
```
