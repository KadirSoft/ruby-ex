# OpenShift CI/CD with ArgoCD — Complete Guide

> A hands-on reference covering OpenShift BuildConfig, GitHub Webhooks, and ArgoCD GitOps.  
> Built from real troubleshooting and learning sessions.

---

## Table of Contents

1. [Project Structure](#project-structure)
2. [OpenShift BuildConfig](#openshift-buildconfig)
3. [Common Build Errors & Fixes](#common-build-errors--fixes)
4. [Expose Your App](#expose-your-app)
5. [Prepare GitHub Repo](#prepare-github-repo)
6. [GitHub Webhook Setup](#github-webhook-setup)
7. [ArgoCD Setup](#argocd-setup)
8. [Full CI/CD Flow](#full-cicd-flow)
9. [Quick Reference Commands](#quick-reference-commands)

---

## Project Structure

```
my-app/
├── config.ru           # app entry point (Ruby)
├── Gemfile             # gem dependencies
├── Gemfile.lock        # locked gem versions
├── README.md
└── manifests/          # OpenShift/Kubernetes YAMLs (GitOps)
    ├── buildconfig.yaml
    ├── imagestream.yaml
    ├── deployment.yaml
    ├── service.yaml
    └── route.yaml
```

---

## OpenShift BuildConfig

### What is a BuildConfig?

A **BuildConfig** is an OpenShift-specific resource that defines how to build a container image from source code. It tells OpenShift:
- Where to get the source code (Git)
- How to build it (strategy)
- What base image to use
- Where to push the final image

### How It Works

```
Source Code (Git)
      ↓
  BuildConfig  ──triggers──→  Build (object)
                                    ↓
                              Build Pod (runs the actual build)
                                    ↓
                          Container Image (pushed to registry)
                                    ↓
                         Deployment → Running App Pod
```

### Build Strategies

| Strategy | Use Case |
|----------|----------|
| **Source (S2I)** | Provide code + builder image, OpenShift handles Dockerfile |
| **Docker** | You provide your own Dockerfile |
| **Pipeline** | Jenkins-based complex CI/CD |
| **Custom** | Full control with your own builder image |

### Clean BuildConfig Example

```yaml
apiVersion: build.openshift.io/v1
kind: BuildConfig
metadata:
  annotations:
    openshift.io/generated-by: OpenShiftNewApp
  labels:
    app: ruby-ex
    app.kubernetes.io/component: ruby-ex
    app.kubernetes.io/instance: ruby-ex
  name: ruby-ex
  namespace: myproject
spec:
  failedBuildsHistoryLimit: 5
  successfulBuildsHistoryLimit: 5
  runPolicy: Serial
  source:
    git:
      uri: https://github.com/YOUR_USERNAME/YOUR_REPO.git
    type: Git
  strategy:
    type: Source
    sourceStrategy:
      from:
        kind: ImageStreamTag
        name: ruby:3.0-ubi9
        namespace: openshift
  output:
    to:
      kind: ImageStreamTag
      name: ruby-ex:latest
  triggers:
  - type: ConfigChange
  - type: ImageChange
    imageChange: {}
  - type: GitHub
    github:
      secretReference:
        name: github-webhook-secret   # reference Secret, never plain text
```

### Trigger Types

| Trigger | What It Does |
|---------|-------------|
| `ConfigChange` | Rebuilds when BuildConfig YAML is modified |
| `ImageChange` | Rebuilds when the base builder image is updated |
| `GitHub` | Rebuilds when GitHub sends a webhook on git push |
| `Generic` | Rebuilds from any system that can send a webhook |

### runPolicy Options

| Policy | Behaviour |
|--------|-----------|
| `Serial` | Builds run one at a time |
| `Parallel` | Builds run simultaneously |
| `SerialLatestOnly` | Cancels queued builds if a newer one comes in |

### BuildConfig Commands

```bash
# View
oc get buildconfig
oc describe buildconfig ruby-ex
oc get buildconfig ruby-ex -o yaml

# Manage builds
oc start-build ruby-ex
oc start-build ruby-ex --follow       # stream logs
oc cancel-build ruby-ex-3
oc get builds
oc get builds -w                      # watch in real time

# Edit
oc edit buildconfig ruby-ex
oc patch buildconfig ruby-ex -p '{"spec":{"source":{"git":{"uri":"https://github.com/USER/REPO.git"}}}}'

# Logs
oc logs -f bc/ruby-ex
oc logs -f build/ruby-ex-5
```

---

## Common Build Errors & Fixes

### Error 1 — Ruby Version Incompatibility (GenericBuildFailed)

```
Gem::InstallError: nio4r requires Ruby version >= 2.4.
```

**Cause:** Builder image provides Ruby 2.2, gem requires 2.4+.

**Fix:** Update BuildConfig to use a newer Ruby image.

```bash
# Check available Ruby image streams
oc get imagestream -n openshift | grep ruby

# Patch to use newer Ruby
oc patch buildconfig ruby-ex -p \
  '{"spec":{"strategy":{"sourceStrategy":{"from":{"kind":"ImageStreamTag","name":"ruby:3.0-ubi9","namespace":"openshift"}}}}}'

# Start a new build
oc start-build ruby-ex --follow
```

### Error 2 — FetchSourceFailed

```
error: fatal: remote error: upload-pack: not our ref <commit-hash>
```

**Cause:** BuildConfig git URI points to wrong repository (e.g., original upstream instead of your fork).

**Fix:**

```bash
oc patch buildconfig ruby-ex -p \
  '{"spec":{"source":{"git":{"uri":"https://github.com/YOUR_USERNAME/YOUR_REPO.git"}}}}'
```

### Error 3 — Webhook 403 Forbidden

```
User "system:anonymous" cannot create resource "buildconfigs/webhooks"
```

**Cause:** OpenShift RBAC not allowing anonymous webhook calls.

**Fix:**

```bash
oc policy add-role-to-user system:webhook system:anonymous -n myproject
```

### Error 4 — Webhook 400 Bad Request

```
unsupported Content-Type application/x-www-form-urlencoded
```

**Cause:** GitHub webhook Content-Type is wrong.

**Fix:** In GitHub webhook settings → change Content-Type to `application/json`.

---

## Expose Your App

```
Service (ClusterIP) = internal only
Route               = external public URL
```

```bash
# Create route from existing service
oc expose svc ruby-ex

# Get the public URL
oc get route
oc get route ruby-ex -o jsonpath='{.spec.host}'
```

### Clean Route Manifest

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  labels:
    app: ruby-ex
  name: ruby-ex
  namespace: myproject
spec:
  port:
    targetPort: 8080-tcp
  to:
    kind: Service
    name: ruby-ex
    weight: 100
  wildcardPolicy: None
```

---

## Prepare GitHub Repo

### Step 1 — Export Existing OpenShift Resources

```bash
oc get buildconfig ruby-ex -o yaml  > manifests/buildconfig.yaml
oc get imagestream ruby-ex -o yaml  > manifests/imagestream.yaml
oc get deployment ruby-ex -o yaml   > manifests/deployment.yaml
oc get svc ruby-ex -o yaml          > manifests/service.yaml
oc get route ruby-ex -o yaml        > manifests/route.yaml
```

### Step 2 — Clean the YAMLs

Remove these fields from all exported files before committing to Git:

| Field | Why Remove |
|-------|-----------|
| `creationTimestamp` | Set by cluster at runtime |
| `resourceVersion` | Changes on every modification |
| `uid` | Unique to each cluster instance |
| `generation` | Managed by cluster |
| `managedFields` | Internal cluster bookkeeping |
| `status` block | Runtime state, not configuration |
| `clusterIP` / `clusterIPs` | Assigned automatically |
| Hardcoded image SHA | Use `:latest` tag instead |
| Webhook secret plain text | Use `secretReference` instead |

### Step 3 — Push to GitHub

```bash
git add manifests/
git commit -m "add clean openshift manifests"
git push origin master
```

---

## GitHub Webhook Setup

### Step 1 — Create Webhook Secret in OpenShift

```bash
# Create secret
oc create secret generic github-webhook-secret \
  --from-literal=WebHookSecretKey=$(openssl rand -hex 20)

# Get the secret value (needed for GitHub)
oc get secret github-webhook-secret \
  -o jsonpath='{.data.WebHookSecretKey}' | base64 -d
```

### Step 2 — Update BuildConfig to Use Secret Reference

```bash
oc patch buildconfig ruby-ex -p '{
  "spec": {
    "triggers": [{
      "type": "GitHub",
      "github": {
        "secretReference": {
          "name": "github-webhook-secret"
        }
      }
    }]
  }
}'
```

### Step 3 — Get Webhook URL

```bash
oc describe buildconfig ruby-ex | grep -A 1 "Webhook GitHub"
```

> ⚠️ OpenShift shows `<secret>` as a placeholder for security. Replace it manually with your actual secret value.

Full URL format:
```
https://api.<cluster-domain>:6443/apis/build.openshift.io/v1/namespaces/<namespace>/buildconfigs/<name>/webhooks/<secret-value>/github
```

### Step 4 — Add Webhook in GitHub

Go to **GitHub repo → Settings → Webhooks → Add webhook**:

| Field | Value |
|-------|-------|
| Payload URL | Full webhook URL with secret value |
| Content type | `application/json` ← critical! |
| Secret | Your secret value |
| SSL verification | Disable if self-signed cluster cert |
| Events | Just the push event |
| Active | ✅ |

### Step 5 — Fix RBAC for Webhook

```bash
oc policy add-role-to-user system:webhook system:anonymous -n myproject
```

### Step 6 — Test

```bash
echo "# test" >> README.md
git add README.md
git commit -m "test webhook"
git push origin master

# Watch for new build
oc get builds -w
```

---

## ArgoCD Setup

### Grant ArgoCD Permissions on Target Namespace

```bash
# Application controller
oc adm policy add-role-to-user admin \
  system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller \
  -n myproject

# ArgoCD server
oc adm policy add-role-to-user admin \
  system:serviceaccount:openshift-gitops:openshift-gitops-server \
  -n myproject

# Verify
oc get rolebinding -n myproject
```

> ⚠️ Always grant permissions BEFORE syncing from ArgoCD, otherwise you'll get 403 Forbidden errors.

### Via OpenShift Console GUI

1. Go to **myproject** namespace
2. **User Management → RoleBindings → Create binding**

| Field | Value |
|-------|-------|
| Binding type | Namespace role binding |
| Name | `argocd-controller-admin` |
| Namespace | `myproject` |
| Role name | `admin` |
| Subject | Service Account |
| Subject namespace | `openshift-gitops` |
| Subject name | `openshift-gitops-argocd-application-controller` |

Repeat for `openshift-gitops-server`.

### Create ArgoCD Application

Get ArgoCD URL:
```bash
oc get route openshift-gitops-server -n openshift-gitops
```

Get admin password:
```bash
oc get secret openshift-gitops-cluster \
  -n openshift-gitops \
  -o jsonpath='{.data.admin\.password}' | base64 -d
```

In ArgoCD UI → **New App**:

| Field | Value |
|-------|-------|
| Application Name | `ruby-ex` |
| Project | `default` |
| Sync Policy | `Automatic` |
| Repository URL | `https://github.com/YOUR_USERNAME/YOUR_REPO.git` |
| Revision | `master` |
| Path | `manifests` |
| Cluster | `https://kubernetes.default.svc` |
| Namespace | `myproject` |

---

## Full CI/CD Flow

```
Developer pushes code to GitHub
            ↓
GitHub Webhook → OpenShift BuildConfig
            ↓
OpenShift S2I Build (new pod spun up)
            ↓
Image pushed to internal registry (ImageStream updated)
            ↓
Deployment auto-updated via ImageStream trigger
            ↓
New app pod running
            ↓
ArgoCD detects manifest drift → Syncs
            ↓
Cluster state matches Git state ✅
```

---

## Quick Reference Commands

```bash
# Project
oc new-project myproject
oc project myproject
oc delete project myproject

# App lifecycle
oc new-app ruby:3.0-ubi9~https://github.com/USER/REPO.git
oc get all
oc get pods
oc get pods -w

# Builds
oc get builds
oc get builds -w
oc start-build ruby-ex --follow
oc cancel-build ruby-ex-3
oc logs -f bc/ruby-ex

# Images
oc get imagestream
oc get is

# Networking
oc get svc
oc get route
oc expose svc ruby-ex

# BuildConfig
oc get buildconfig
oc describe buildconfig ruby-ex
oc edit buildconfig ruby-ex
oc patch buildconfig ruby-ex -p '{...}'

# RBAC
oc get rolebinding -n myproject
oc adm policy add-role-to-user admin <subject> -n myproject
oc policy add-role-to-user system:webhook system:anonymous -n myproject

# ArgoCD
oc get pods -n openshift-gitops
oc get route openshift-gitops-server -n openshift-gitops

# Secrets
oc create secret generic github-webhook-secret \
  --from-literal=WebHookSecretKey=$(openssl rand -hex 20)
oc get secret github-webhook-secret \
  -o jsonpath='{.data.WebHookSecretKey}' | base64 -d

# Events & debugging
oc get events -n myproject
oc describe pod <pod-name>
oc logs <pod-name>
```

---

## Key Lessons Learned

| Lesson | Detail |
|--------|--------|
| Always use modern builder images | `centos/ruby-22-centos7` is EOL — use `ruby:3.0-ubi9` from OpenShift namespace |
| Never store secrets in Git | Use `secretReference` pointing to a Kubernetes Secret |
| Clean YAMLs before committing | Remove runtime fields: uid, resourceVersion, status, clusterIP |
| Webhook Content-Type matters | Must be `application/json` not `application/x-www-form-urlencoded` |
| Grant ArgoCD RBAC before syncing | Otherwise all resources fail with 403 Forbidden |
| Service ≠ Route | Service is internal only, Route exposes to the internet |
| Git URI must match your fork | BuildConfig must point to YOUR repo, not the upstream |
| OpenShift hides webhook secrets | `<secret>` in describe output is intentional security masking |
