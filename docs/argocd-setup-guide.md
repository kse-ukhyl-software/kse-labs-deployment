# ArgoCD GitOps Setup Guide

This guide explains how to configure ArgoCD with ApplicationSets for a fully automated GitOps workflow.

## Overview

The goal is to run `kubectl apply` **once** to bootstrap ArgoCD, and then manage everything through Git. No more manual `kubectl` commands for deployments.

```
kubectl apply -f bootstrap.yaml    ← The only manual command you ever run
        │
        ▼
   Everything else is GitOps
```

## Repository Structure

```
kse-labs-deployment/
├── bootstrap.yaml                    ← Apply once manually
├── argocd/
│   ├── applicationsets/
│   │   ├── services.yaml             ← Auto-discovers services/*
│   │   └── infra.yaml                ← Auto-discovers infra/*
│   └── projects/
│       ├── services-project.yaml
│       └── infra-project.yaml
├── infra/                            ← Infrastructure components
│   ├── cert-manager/
│   └── keycloak/
└── services/                         ← Application services
    ├── example-service/
    ├── auth-service/
    └── policy-engine/
```

## Prerequisites

1. A Kubernetes cluster with ArgoCD installed
2. `kubectl` configured to access the cluster
3. Git repository accessible by ArgoCD

## Step-by-Step Setup

### Step 1: Create the Repository Structure

```bash
mkdir -p argocd/applicationsets
mkdir -p argocd/projects
mkdir -p services
mkdir -p infra
```

### Step 2: Create AppProjects

AppProjects define permissions and boundaries for applications.

**argocd/projects/services-project.yaml**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: services
  namespace: argocd
spec:
  description: Application services
  sourceRepos:
    - https://github.com/YOUR_ORG/YOUR_REPO.git
  destinations:
    - namespace: '*'
      server: https://kubernetes.default.svc
  clusterResourceWhitelist:
    - group: ''
      kind: Namespace
```

**argocd/projects/infra-project.yaml**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: infra
  namespace: argocd
spec:
  description: Infrastructure components
  sourceRepos:
    - https://github.com/YOUR_ORG/YOUR_REPO.git
  destinations:
    - namespace: '*'
      server: https://kubernetes.default.svc
  clusterResourceWhitelist:
    - group: '*'
      kind: '*'
```

### Step 3: Create ApplicationSets

ApplicationSets automatically create Applications based on a generator pattern.

**argocd/applicationsets/services.yaml**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: services
  namespace: argocd
spec:
  generators:
    - git:
        repoURL: https://github.com/YOUR_ORG/YOUR_REPO.git
        revision: main
        directories:
          - path: services/*
  template:
    metadata:
      name: 'svc-{{path.basename}}'
    spec:
      project: services
      source:
        repoURL: https://github.com/YOUR_ORG/YOUR_REPO.git
        targetRevision: main
        path: '{{path}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{path.basename}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

**argocd/applicationsets/infra.yaml**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: infra
  namespace: argocd
spec:
  generators:
    - git:
        repoURL: https://github.com/YOUR_ORG/YOUR_REPO.git
        revision: main
        directories:
          - path: infra/*
  template:
    metadata:
      name: 'infra-{{path.basename}}'
      annotations:
        argocd.argoproj.io/sync-wave: "-1"
    spec:
      project: infra
      source:
        repoURL: https://github.com/YOUR_ORG/YOUR_REPO.git
        targetRevision: main
        path: '{{path}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{path.basename}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

### Step 4: Create the Bootstrap Application

This is the root application that manages everything else.

**bootstrap.yaml** (in repository root)
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: bootstrap
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR_ORG/YOUR_REPO.git
    targetRevision: main
    path: argocd/
    directory:
      recurse: true    # IMPORTANT: Required to find files in subdirectories
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

> ⚠️ **Important**: The `directory.recurse: true` setting is required because the ApplicationSets and Projects are in subdirectories.

### Step 5: Create a Service

Each service needs Kubernetes manifests in its own directory.

**services/example-service/namespace.yaml**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: example-service
```

**services/example-service/deployment.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: example-service
  namespace: example-service
spec:
  replicas: 1
  selector:
    matchLabels:
      app: example-service
  template:
    metadata:
      labels:
        app: example-service
    spec:
      containers:
        - name: example-service
          image: ghcr.io/your-org/example-service:v1.0.0
          ports:
            - containerPort: 8080
```

**services/example-service/service.yaml**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: example-service
  namespace: example-service
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: 8080
  selector:
    app: example-service
```

### Step 6: Push and Bootstrap

```bash
# Commit and push all files
git add .
git commit -m "Initial ArgoCD setup with ApplicationSets"
git push

# Apply the bootstrap application (ONE TIME ONLY)
kubectl apply -f bootstrap.yaml
```

### Step 7: Verify

```bash
# Check bootstrap app
kubectl get applications -n argocd

# Check ApplicationSets were created
kubectl get applicationsets -n argocd

# Check all apps created by ApplicationSets
kubectl get applications -n argocd
```

Expected output:
```
NAME                  SYNC STATUS   HEALTH STATUS
bootstrap             Synced        Healthy
svc-example-service   Synced        Healthy
```

## How It Works

```
kubectl apply bootstrap.yaml
        │
        ▼
   bootstrap Application
   (watches argocd/ with recurse: true)
        │
        ├── syncs argocd/applicationsets/services.yaml
        │         │
        │         ▼
        │   ApplicationSet "services"
        │   (watches services/*)
        │         │
        │         └── creates app: svc-example-service
        │
        ├── syncs argocd/applicationsets/infra.yaml
        │         │
        │         ▼
        │   ApplicationSet "infra"
        │   (watches infra/*)
        │
        └── syncs argocd/projects/*.yaml
                  │
                  └── creates AppProjects
```

## Day-to-Day Operations

### Add a New Service

```bash
mkdir services/new-service
# Create deployment.yaml, service.yaml, etc.
git add .
git commit -m "Add new-service"
git push

# ArgoCD automatically:
# 1. Detects new directory in services/
# 2. Creates Application "svc-new-service"
# 3. Deploys the manifests
```

### Update a Service

```bash
# Edit services/example-service/deployment.yaml (change image tag, replicas, etc.)
git add .
git commit -m "Update example-service to v1.1.0"
git push

# ArgoCD automatically syncs the changes
```

### Remove a Service

```bash
rm -rf services/old-service
git add .
git commit -m "Remove old-service"
git push

# ArgoCD automatically:
# 1. Detects directory removal
# 2. Deletes the Application
# 3. Prunes all Kubernetes resources
```

### Add New Infrastructure Component

```bash
mkdir infra/monitoring
# Add Prometheus, Grafana manifests
git add .
git commit -m "Add monitoring stack"
git push

# ArgoCD creates "infra-monitoring" Application
```

## Private Container Registry (ghcr.io)

When pulling images from a private GitHub Container Registry, the kubelet needs credentials to authenticate. This is separate from ArgoCD's git repository credentials.

### How It Works

```
ArgoCD                              Kubelet
  │                                   │
  ├─ reads manifests from git         ├─ pulls container images from ghcr.io
  │  (uses repo secret)               │  (uses imagePullSecret)
  │                                   │
  └─ applies Deployment to cluster    └─ needs docker-registry credentials
```

- **ArgoCD repo secret** — allows ArgoCD to read manifests from private git repos
- **Image pull secret** — allows kubelet to pull images from private container registries

These are two independent authentication mechanisms.

### Step 1: Create the Image Pull Secret

Create a `docker-registry` secret in the namespace where your workload runs.

> **Important: Use a Classic PAT, not a Fine-grained PAT.**
>
> Fine-grained PATs do **not** support the `read:packages` permission required to pull container images from ghcr.io. The "Artifact metadata" permission available in fine-grained PATs only allows reading package metadata (names, tags), not pulling actual images.
>
> You must use a **Classic PAT** with the `read:packages` scope.

#### Creating the Classic PAT

1. Go to GitHub > Settings > Developer settings > Personal access tokens > **Tokens (classic)**
2. Click **"Generate new token (classic)"**
3. Select the `read:packages` scope
4. If your org uses SSO, click **"Configure SSO"** next to the token and authorize it for the org
5. Copy the token (starts with `ghp_`)

#### Creating the Secret

```bash
kubectl create secret docker-registry ghcr-pull-secret \
  -n <namespace> \
  --docker-server=ghcr.io \
  --docker-username=x-access-token \
  --docker-password='<CLASSIC_PAT_WITH_READ_PACKAGES>'
```

> This secret must exist in **each namespace** that pulls from ghcr.io. If you have multiple services, create the secret in each service's namespace.

### Step 2: Reference the Secret in Deployments

Add `imagePullSecrets` to the pod spec in your deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: example-service
  namespace: example-service
spec:
  template:
    spec:
      imagePullSecrets:
        - name: ghcr-pull-secret      # references the secret created above
      containers:
        - name: example-service
          image: ghcr.io/your-org/example-service:v1.0.0
```

### Step 3: Verify

```bash
# Check the secret exists in the namespace
kubectl get secret ghcr-pull-secret -n example-service

# Check pod events for image pull status
kubectl describe pod -n example-service -l app.kubernetes.io/name=example-service
```

### Important Notes

- The PAT used for image pulling can be different from the one used for ArgoCD git access
- Image pull secrets are **not managed by ArgoCD** in this setup — they are created manually via `kubectl`
- If the secret is deleted (e.g., namespace recreation), it must be recreated before pods can pull images
- To avoid manual secret management, consider using a tool like [External Secrets Operator](https://external-secrets.io/) or adding the secret manifest to your service directory (sealed/encrypted)

## Troubleshooting

### ApplicationSets Not Created

Check if `directory.recurse: true` is set in bootstrap.yaml:

```bash
kubectl get application bootstrap -n argocd -o yaml | grep -A5 directory
```

### Applications Not Being Created

Check ApplicationSet status:

```bash
kubectl describe applicationset services -n argocd
```

### Image Pull Errors (ErrImagePull / ImagePullBackOff)

If pods show `ErrImagePull` or `ImagePullBackOff`:

```bash
# Check pod events for details
kubectl describe pod -n <namespace> <pod-name>
```

Common causes:
- **"denied"** — missing or invalid `imagePullSecrets`. Ensure the `ghcr-pull-secret` exists in the namespace and the PAT has `read:packages` scope
- **"no matching manifest"** — the image tag doesn't exist for the node's architecture (e.g., `linux/arm64` vs `linux/amd64`)
- **"not found"** — the image or tag doesn't exist in the registry

### Sync Issues

Force a refresh:

```bash
kubectl annotate application bootstrap -n argocd argocd.argoproj.io/refresh=normal --overwrite
```

Check application status:

```bash
kubectl get applications -n argocd
kubectl describe application svc-example-service -n argocd
```

## Key Concepts

| Concept | Description |
|---------|-------------|
| **Bootstrap** | Root Application that manages ArgoCD configuration itself |
| **ApplicationSet** | Template that auto-generates Applications based on patterns |
| **Git Generator** | Discovers directories matching a pattern (e.g., `services/*`) |
| **AppProject** | Defines permissions and allowed resources for a group of apps |
| **Sync Policy** | `automated` + `prune` + `selfHeal` = fully automatic GitOps |

## Summary

1. Create repository structure with `argocd/`, `services/`, `infra/`
2. Define ApplicationSets to watch directories
3. Apply `bootstrap.yaml` once
4. Everything else is Git-driven

**No more `kubectl apply` for deployments. Just `git push`.**
