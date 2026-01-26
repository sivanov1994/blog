---
date: 2026-01-26T00:00:00+00:00
draft: false
title: "GitOps for Homelab: Stop kubectl apply, Start Pushing to Git"
description: "How I eliminated manual cluster management and configuration drift by treating Git as the single source of truth for my Kubernetes homelab"
authors: ["Simeon Ivanov"]
tags: ["kubernetes", "homelab", "gitops", "flux", "talos", "automation"]
---

After migrating to Talos Linux, I had an immutable operating system—but I was still managing Kubernetes applications the old way. `kubectl apply -f` everywhere. Configuration files scattered across my laptop. "Did I deploy this? What version am I running? How did I configure that?"

Six months from now, I'd have no idea how to reproduce my setup.

GitOps solved this. My entire homelab infrastructure lives in a Git repository. Every change is a commit, every deployment automatic, every configuration versioned. I can destroy the cluster and rebuild it exactly from one repository.

This is how I set it up.

## The Problem with Manual Kubernetes Management

Here's how I used to deploy applications:

```bash
kubectl apply -f app.yaml
kubectl create secret generic password --from-literal=pass=...
kubectl apply -f ingress.yaml
```

Works great. Until:

- I need to update something but can't find the original YAML
- The cluster breaks and I don't remember what I deployed
- I manually edit a deployment and forget about it
- Someone asks "how did you set this up?" and I have no answer

Configuration drift accumulates. Documentation falls out of date. The cluster works but isn't reproducible.

**With GitOps:**

```bash
git commit -m "Add application"
git push
# Flux deploys it automatically
```

The Git repository becomes the single source of truth. Every change is tracked, reviewable, and reversible. The cluster continuously reconciles to match what's in Git.

## What is GitOps?

GitOps is a way of managing infrastructure where:

1. **Git is the source of truth** - All infrastructure configuration lives in Git
2. **Declarative configuration** - Describe the desired state, not how to get there
3. **Pull-based deployment** - The cluster pulls changes from Git, not you pushing to the cluster
4. **Continuous reconciliation** - The system automatically corrects drift

**In practice:**

- I push YAML to Git
- Flux CD (the GitOps tool) watches my repository
- Flux applies changes to my cluster
- Flux continuously monitors for drift and auto-corrects

I had to shift my thinking: Stop thinking "I need to kubectl apply this." Start thinking "I need to commit this to Git."

## Repository Structure Basics

Before I installed Flux, I needed to understand the repository structure. Here's the minimal setup I used:

```
homelab/
├── clusters/
│   └── staging/
│       ├── flux-system/        # Flux bootstrap (created automatically)
│       ├── infrastructure.yaml # Points to infrastructure configs
│       └── apps.yaml           # Points to application configs
│
├── infrastructure/
│   └── base/
│       └── example-operator/
│
└── apps/
    └── base/
        └── example-app/
```

**Key concepts:**

### Clusters Directory

`clusters/staging/` contains Flux Kustomizations—pointers that tell Flux where to look for manifests.

**Example:** `clusters/staging/apps.yaml`

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  interval: 1m0s
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./apps/base
  prune: true
```

This tells Flux: "Watch `./apps/base` in the Git repository. Check every minute. Apply any changes."

### Infrastructure and Apps Directories

These contain the actual Kubernetes manifests. Organized by function.

**Why separate infrastructure from apps?** Dependencies. Operators install first, applications second.

### The Flow

1. I commit YAML to `apps/base/my-app/`
2. Flux sees the commit (within 1 minute)
3. Flux reads `clusters/staging/apps.yaml` to know where to look
4. Flux applies manifests from `apps/base/my-app/`
5. My app deploys

## Installing Flux CD

Prerequisites:

- Working Kubernetes cluster (Talos or any distribution)
- GitHub account with personal access token
- `flux` CLI installed on your workstation

### Install Flux CLI

```bash
curl -s https://fluxcd.io/install.sh | sudo bash
flux --version
```

### Bootstrap Flux

I ran this single command to install Flux to my cluster and configure it to watch my repository:

```bash
flux bootstrap github \
  --owner=YOUR_GITHUB_USERNAME \
  --repository=homelab \
  --branch=main \
  --path=clusters/staging \
  --personal
```

**What bootstrap does:**

1. Creates the GitHub repository (or uses existing)
2. Installs Flux controllers to the `flux-system` namespace
3. Creates a deploy key for the repository
4. Commits Flux manifests to `clusters/staging/flux-system/`
5. Configures Flux to watch that path
6. Sets up automatic reconciliation every minute

The bootstrap is idempotent—running it again won't break anything.

### Verify Installation

```bash
flux check
```

Expected output:
```
✔ Kubernetes 1.32.1 >=1.28.0-0
✔ prerequisites checks passed
✔ helm-controller: deployment ready
✔ kustomize-controller: deployment ready
✔ notification-controller: deployment ready
✔ source-controller: deployment ready
✔ all checks passed
```

Check the pods:

```bash
kubectl get pods -n flux-system
```

All Flux controller pods should be running. Flux is now watching my `clusters/staging/` directory.

## Secret Management with SOPS

I couldn't commit passwords and API tokens to Git in plain text. That would expose them to anyone with repository access.

I solved this with SOPS (Secrets OPerationS), which encrypts secret values while keeping the structure readable.

### Generate Age Key

```bash
mkdir -p ~/.config/sops/age
age-keygen -o ~/.config/sops/age/keys.txt
```

This created a public/private key pair. I viewed my public key:

```bash
age-keygen -y ~/.config/sops/age/keys.txt
```

I copied the public key (starts with `age1...`).

### Configure SOPS

I created `.sops.yaml` in my repository root:

```yaml
creation_rules:
  - path_regex: .*.yaml
    encrypted_regex: ^(data|stringData)$
    age: age1xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

I replaced the `age:` value with my public key.

**What this does:**
- Applies to all YAML files
- Only encrypts `data` and `stringData` fields (Kubernetes secrets)
- Uses my public key for encryption

I committed `.sops.yaml` to Git—it contains only the public key, so it's safe.

### Add Age Key to Cluster

I added the private key to the cluster so Flux could decrypt secrets:

```bash
cat ~/.config/sops/age/keys.txt | \
  kubectl create secret generic sops-age \
  --namespace=flux-system \
  --from-file=age.agekey=/dev/stdin
```

### Configure Flux to Decrypt

I added the decryption configuration to my Kustomizations:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  interval: 1m0s
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./apps/base
  prune: true
  decryption:
    provider: sops
    secretRef:
      name: sops-age
```

The `decryption` section tells Flux to use SOPS with the `sops-age` secret.

### Using SOPS

**Create a secret:**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: example-secret
  namespace: default
stringData:
  password: super-secret-password
```

**Encrypt it:**

```bash
sops --encrypt --in-place secret.yaml
```

**Result:**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: example-secret
  namespace: default
stringData:
  password: ENC[AES256_GCM,data:ENCRYPTED,tag:...,type:str]
sops:
  age:
    - recipient: age1...
      enc: |
        -----BEGIN AGE ENCRYPTED FILE-----
        ...
        -----END AGE ENCRYPTED FILE-----
```

Only the password value is encrypted. The structure is readable.

**Commit safely:**

```bash
git add secret.yaml
git commit -m "Add secret"
git push
```

Flux pulls the encrypted secret, decrypts it, and applies it to the cluster.

## Deploying Your First Application

I deployed a simple nginx web server to test the workflow.

### Create Application Structure

```bash
mkdir -p apps/base/nginx
```

`apps/base/nginx/namespace.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: nginx
```

`apps/base/nginx/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
```

`apps/base/nginx/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  namespace: nginx
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
```

`apps/base/nginx/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: nginx
resources:
  - namespace.yaml
  - deployment.yaml
  - service.yaml
```

### Create Flux Kustomization

`clusters/staging/apps.yaml`:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  interval: 1m0s
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./apps/base
  prune: true
```

### Deploy

```bash
git add apps/ clusters/staging/apps.yaml
git commit -m "Add nginx application"
git push
```

**What happens:**

1. Flux detects the commit (within 1 minute)
2. Reads `clusters/staging/apps.yaml`
3. Applies manifests from `apps/base/nginx/`
4. Nginx deploys

**Watch it happen:**

```bash
flux get kustomizations --watch
```

**Verify:**

```bash
kubectl get pods -n nginx
kubectl get svc -n nginx
```

That's it. I deployed an application by pushing to Git.

## Understanding Dependencies

What if I need applications that depend on infrastructure operators? I use `dependsOn`.

**Example:** My apps need a database operator to be installed first.

`clusters/staging/infrastructure.yaml`:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: infrastructure
  namespace: flux-system
spec:
  interval: 1m0s
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./infrastructure/base
  prune: true
```

`clusters/staging/apps.yaml`:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  interval: 1m0s
  dependsOn:
    - name: infrastructure
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./apps/base
  prune: true
```

The `dependsOn: infrastructure` tells Flux to wait for infrastructure to be healthy before deploying apps.

**The deployment chain:**

```
flux-system (bootstrap)
    ↓
infrastructure (operators)
    ↓
apps (applications)
```

Flux respects this order automatically.

## Day-to-Day Workflow

**How I deploy a new application:**

1. Create directory in `apps/base/app-name/`
2. Add Kubernetes manifests
3. Create `kustomization.yaml` listing resources
4. Commit and push

Flux handles deployment.

**How I update an application:**

1. Edit the YAML (change image version, config)
2. Commit and push

Flux applies the change.

**How I remove an application:**

1. Delete the directory
2. Commit and push

Flux deletes the application (because `prune: true`).

**How I debug failures:**

```bash
# Check overall status
flux get kustomizations

# Describe specific Kustomization
kubectl describe kustomization apps -n flux-system

# Check application logs
kubectl logs -n nginx deployment/nginx
```

**Useful commands:**

```bash
# Force immediate sync (don't wait for interval)
flux reconcile kustomization apps --with-source

# Suspend automatic reconciliation
flux suspend kustomization apps

# Resume
flux resume kustomization apps

# See what Flux will apply (dry-run)
flux diff kustomization apps --path ./apps/base
```

## Disaster Recovery

**Scenario:** My cluster dies. How do I recover?

**Without GitOps:** Days of manual work, probably getting something wrong.

**With GitOps:** 15 minutes.

### Recovery Steps

**1. Rebuild the Kubernetes cluster**

I'd install Talos again and get a working cluster.

**2. Bootstrap Flux**

```bash
flux bootstrap github \
  --owner=YOUR_GITHUB_USERNAME \
  --repository=homelab \
  --branch=main \
  --path=clusters/staging \
  --personal
```

**3. Restore the Age key**

```bash
cat ~/.config/sops/age/keys.txt | \
  kubectl create secret generic sops-age \
  --namespace=flux-system \
  --from-file=age.agekey=/dev/stdin
```

**4. Wait**

Flux reads the Git repository and deploys everything. Infrastructure, then applications.

**Recovery time:** ~15 minutes.

### What I Need to Keep Safe

- Age private key (`~/.config/sops/age/keys.txt`)
- GitHub access

That's it. Everything else is in Git.

I store my Age key in a password manager with an offline backup.

## What Changed

**Before GitOps:**

- Manual `kubectl apply` commands
- Configuration files on laptop
- Lost track of what's deployed
- "It worked yesterday" debugging
- No audit trail
- Disaster recovery measured in days

**After GitOps:**

- Git commit → automatic deployment
- All configuration in repository
- Complete visibility via Git history
- Changes are reviewable commits
- Full audit trail
- Disaster recovery in 15 minutes

**The key insight:**

My cluster became a reflection of my Git repository. Want to know what's running? Look at Git. Want to change something? Commit to Git. Want to roll back? Git revert.

## Best Practices

**1. Small, atomic commits**

I keep one change per commit. This makes reviewing easier and rolling back safer.

❌ `git commit -m "Update everything"`
✅ `git commit -m "Update nginx to 1.26"`

**2. Test locally first**

```bash
# Validate manifest syntax
kubectl apply --dry-run=client -f app.yaml

# Preview Kustomize output
kubectl kustomize apps/base/nginx

# See what Flux will apply
flux diff kustomization apps --path ./apps/base
```

**3. Keep the Age key safe**

Without it, disaster recovery fails. I store mine in a password manager with an offline backup. Never commit it to Git.

**4. Monitor Flux reconciliation**

I check this regularly:

```bash
flux get kustomizations
```

I also set up Prometheus alerts for failed reconciliations.

**5. Use branches for risky changes**

I test major changes in a branch first. I review the diff, then merge when I'm confident.

## Next Steps

I now have a GitOps-managed homelab. The foundation is set. Infrastructure as code. Disaster recovery in 15 minutes. Every change versioned and reviewable.

**My upcoming articles will cover:**

- Setting up infrastructure operators (cert-manager, Traefik, CloudNativePG)
- Running production databases with automatic backups
- Monitoring with Prometheus and Grafana
- Automated dependency updates with Renovate

For now, I have the fundamentals: Flux, SOPS, and the workflow.

**Resources:**

- [Flux CD Documentation](https://fluxcd.io/docs/)
- [SOPS Documentation](https://github.com/mozilla/sops)
- [Awesome Flux](https://github.com/fluxcd/awesome-flux)
