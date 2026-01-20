# OpenShift CI/CD Pipeline with Tekton and ArgoCD

## Overview

This guide covers setting up a complete CI/CD pipeline using:
- **Tekton (OpenShift Pipelines)** — for CI (build and push images)
- **ArgoCD (OpenShift GitOps)** — for CD (deploy to cluster)

### Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  GitHub App Repo                                                            │
│  (source code + Dockerfile)                                                 │
│       │                                                                     │
│       ▼                                                                     │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐  │
│  │   Tekton    │    │   Build     │    │   Push to   │    │   Update    │  │
│  │   Pipeline  │───►│   Image     │───►│   Registry  │───►│   GitOps    │  │
│  │             │    │             │    │             │    │   Repo      │  │
│  └─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘  │
│                                                                     │       │
│                                                                     ▼       │
│  GitHub GitOps Repo                                        ┌─────────────┐  │
│  (K8s manifests)  ◄────────────────────────────────────────│   ArgoCD    │  │
│       │                                                    │   Watches   │  │
│       │                                                    └─────────────┘  │
│       ▼                                                                     │
│  ┌─────────────┐                                                            │
│  │   Deploy    │                                                            │
│  │   to        │                                                            │
│  │   Cluster   │                                                            │
│  └─────────────┘                                                            │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Prerequisites

- OpenShift 4.x cluster
- `oc` CLI with cluster-admin access
- GitHub account
- Working StorageClass (for pipeline workspace PVCs)

---

## Step 1: Install OpenShift Pipelines Operator

```bash
cat <<'EOF' | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-pipelines-operator
  namespace: openshift-operators
spec:
  channel: latest
  name: openshift-pipelines-operator-rh
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
```

Verify installation:

```bash
# Wait for pods to be ready
oc get pods -n openshift-pipelines

# Check for ClusterTasks
oc get tasks -n openshift-pipelines
```

---

## Step 2: Install OpenShift GitOps Operator

```bash
cat <<'EOF' | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-gitops-operator
  namespace: openshift-operators
spec:
  channel: latest
  name: openshift-gitops-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
```

Verify installation:

```bash
# Check pods
oc get pods -n openshift-gitops

# Get ArgoCD UI URL
oc get route -n openshift-gitops openshift-gitops-server -o jsonpath='{.spec.host}'

# Get admin password
oc extract secret/openshift-gitops-cluster -n openshift-gitops --to=-
```

---

## Step 3: Enable Internal Image Registry

The internal registry stores images built by pipelines.

### Check registry status:

```bash
oc get pods -n openshift-image-registry
oc get co image-registry
```

### If registry pod is missing, enable it:

```bash
# Create PVC for registry storage
cat <<'EOF' | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: image-registry-storage
  namespace: openshift-image-registry
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
EOF

# Configure registry to use the PVC
oc patch configs.imageregistry.operator.openshift.io cluster --type merge --patch '{"spec":{"managementState":"Managed","storage":{"pvc":{"claim":"image-registry-storage"}}}}'
```

Wait for registry pod:

```bash
oc get pods -n openshift-image-registry -w
```

---

## Step 4: Create GitHub Repositories

Create two repositories on GitHub:

### App Repository (e.g., `demo-app`)

Contains your application source code and Dockerfile.

**main.go:**
```go
package main

import (
    "fmt"
    "net/http"
    "os"
)

func main() {
    version := os.Getenv("APP_VERSION")
    if version == "" {
        version = "1.0.0"
    }
    
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "Hello from demo-app! Version: %s\n", version)
    })
    
    fmt.Println("Server starting on :8080")
    http.ListenAndServe(":8080", nil)
}
```

**Dockerfile:**
```dockerfile
FROM golang:1.21-alpine AS builder
WORKDIR /app
COPY main.go .
RUN go build -o server main.go

FROM alpine:latest
WORKDIR /app
COPY --from=builder /app/server .
EXPOSE 8080
CMD ["./server"]
```

### GitOps Repository (e.g., `demo-gitops`)

Contains Kubernetes manifests that ArgoCD watches.

**apps/demo-app/deployment.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-app
  namespace: demo-cicd
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demo-app
  template:
    metadata:
      labels:
        app: demo-app
    spec:
      containers:
      - name: demo-app
        image: image-registry.openshift-image-registry.svc:5000/demo-cicd/demo-app:latest
        ports:
        - containerPort: 8080
        env:
        - name: APP_VERSION
          value: "1.0.0"
```

**apps/demo-app/service.yaml:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: demo-app
  namespace: demo-cicd
spec:
  selector:
    app: demo-app
  ports:
  - port: 8080
    targetPort: 8080
```

**apps/demo-app/route.yaml:**
```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: demo-app
  namespace: demo-cicd
spec:
  to:
    kind: Service
    name: demo-app
  port:
    targetPort: 8080
```

**apps/demo-app/kustomization.yaml:**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
  - route.yaml
```

---

## Step 5: Create Project and Configure Permissions

```bash
# Create namespace
oc new-project demo-cicd

# Give pipeline SA permission to push images
oc policy add-role-to-user system:image-builder system:serviceaccount:demo-cicd:pipeline -n demo-cicd

# Give pipeline SA edit access
oc policy add-role-to-user edit system:serviceaccount:demo-cicd:pipeline -n demo-cicd

# Give ArgoCD permission to deploy to namespace
oc adm policy add-role-to-user admin system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller -n demo-cicd
```

---

## Step 6: Create Tekton Pipeline

```bash
cat <<'EOF' | oc apply -f -
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: build-and-push
  namespace: demo-cicd
spec:
  params:
    - name: git-url
      type: string
      description: Git repository URL
    - name: git-revision
      type: string
      default: main
    - name: image-name
      type: string
      description: Image to build
  workspaces:
    - name: source
  tasks:
    - name: clone
      taskRef:
        resolver: cluster
        params:
          - name: kind
            value: task
          - name: name
            value: git-clone
          - name: namespace
            value: openshift-pipelines
      params:
        - name: URL
          value: $(params.git-url)
        - name: REVISION
          value: $(params.git-revision)
      workspaces:
        - name: output
          workspace: source

    - name: build-push
      taskRef:
        resolver: cluster
        params:
          - name: kind
            value: task
          - name: name
            value: buildah
          - name: namespace
            value: openshift-pipelines
      runAfter:
        - clone
      params:
        - name: IMAGE
          value: $(params.image-name)
        - name: DOCKERFILE
          value: ./Dockerfile
      workspaces:
        - name: source
          workspace: source
EOF
```

Verify:

```bash
oc get pipeline -n demo-cicd
```

---

## Step 7: Run the Pipeline

```bash
cat <<'EOF' | oc create -f -
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  generateName: build-demo-app-
  namespace: demo-cicd
spec:
  pipelineRef:
    name: build-and-push
  params:
    - name: git-url
      value: https://github.com/<YOUR_USERNAME>/demo-app.git
    - name: git-revision
      value: main
    - name: image-name
      value: image-registry.openshift-image-registry.svc:5000/demo-cicd/demo-app:latest
  workspaces:
    - name: source
      volumeClaimTemplate:
        spec:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 1Gi
EOF
```

Watch the pipeline:

```bash
# Watch status
oc get pipelinerun -n demo-cicd -w

# View logs
tkn pipelinerun logs -f -n demo-cicd
```

Or view in **OpenShift Console → Pipelines → PipelineRuns**

---

## Step 8: Verify Image in Registry

```bash
# Check image streams
oc get imagestream -n demo-cicd

# Check tags
oc get imagestreamtag -n demo-cicd
```

---

## Step 9: Create ArgoCD Application

```bash
cat <<'EOF' | oc apply -f -
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: demo-app
  namespace: openshift-gitops
spec:
  project: default
  source:
    repoURL: https://github.com/<YOUR_USERNAME>/demo-gitops.git
    targetRevision: main
    path: apps/demo-app
  destination:
    server: https://kubernetes.default.svc
    namespace: demo-cicd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOF
```

Check status:

```bash
oc get application -n openshift-gitops
```

---

## Step 10: Verify Deployment

```bash
# Check pods
oc get pods -n demo-cicd

# Check route
oc get route -n demo-cicd

# Test the app
curl $(oc get route demo-app -n demo-cicd -o jsonpath='{.spec.host}')
```

---

## Troubleshooting

### Pipeline Issues

#### Error: ClusterTask validation failed

**Symptom:**
```
admission webhook "validation.webhook.pipeline.tekton.dev" denied the request: validation failed: invalid value: custom task ref must specify apiVersion
```

**Fix:** Use resolver syntax instead of ClusterTask:
```yaml
taskRef:
  resolver: cluster
  params:
    - name: kind
      value: task
    - name: name
      value: git-clone
    - name: namespace
      value: openshift-pipelines
```

#### Error: Image push failed - registry not found

**Symptom:**
```
dial tcp: lookup image-registry.openshift-image-registry.svc on 172.30.0.10:53: no such host
```

**Fix:** Enable the internal registry (see Step 3).

#### Error: Permission denied pushing image

**Symptom:**
```
unauthorized: authentication required
```

**Fix:**
```bash
oc policy add-role-to-user system:image-builder system:serviceaccount:demo-cicd:pipeline -n demo-cicd
```

### Storage Issues

#### Error: PVC stuck in Pending

**Symptom:**
```
no datastores found to create file volume, vSAN file service may be disabled
```

**Fix:** Create StorageClass without vSAN policy:
```bash
cat <<'EOF' | oc apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: vsphere-block
provisioner: csi.vsphere.vmware.com
allowVolumeExpansion: true
reclaimPolicy: Delete
volumeBindingMode: Immediate
EOF
```

### ArgoCD Issues

#### Error: Cannot create resources in namespace

**Symptom:**
```
services is forbidden: User "system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller" cannot create resource "services"
```

**Fix:**
```bash
oc adm policy add-role-to-user admin system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller -n demo-cicd
```

Or for cluster-wide access:
```bash
oc adm policy add-cluster-role-to-user cluster-admin system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller
```

#### Error: Image pull failed - name unknown

**Symptom:**
```
reading manifest latest in image-registry.openshift-image-registry.svc:5000/demo-cicd/demo-app: name unknown
```

**Fix:** Verify the image name matches what was pushed:
```bash
# Check what images exist
oc get imagestream -n demo-cicd
oc get imagestreamtag -n demo-cicd

# Update deployment.yaml in GitOps repo with correct image name
```

### Registry Issues

#### Error: Registry pod not starting

**Check status:**
```bash
oc get configs.imageregistry.operator.openshift.io cluster -o yaml
oc get pvc -n openshift-image-registry
oc describe pvc -n openshift-image-registry
```

**Fix:** Ensure storage is configured and PVC is bound.

---

## Useful Commands Reference

### Pipeline Commands

```bash
# List pipelines
oc get pipeline -n demo-cicd

# List pipeline runs
oc get pipelinerun -n demo-cicd

# View pipeline run logs
tkn pipelinerun logs <pipelinerun-name> -n demo-cicd

# Cancel a pipeline run
tkn pipelinerun cancel <pipelinerun-name> -n demo-cicd

# Delete old pipeline runs
oc delete pipelinerun --all -n demo-cicd
```

### ArgoCD Commands

```bash
# List applications
oc get application -n openshift-gitops

# Get ArgoCD UI URL
oc get route -n openshift-gitops openshift-gitops-server -o jsonpath='{.spec.host}'

# Get admin password
oc extract secret/openshift-gitops-cluster -n openshift-gitops --to=-

# Sync application manually
argocd app sync demo-app

# Delete application
oc delete application demo-app -n openshift-gitops
```

### Registry Commands

```bash
# Check registry status
oc get pods -n openshift-image-registry
oc get co image-registry

# List images in namespace
oc get imagestream -n demo-cicd

# List image tags
oc get imagestreamtag -n demo-cicd

# Delete an image
oc delete imagestream demo-app -n demo-cicd
```

---

## Complete CI/CD Flow Summary

1. **Developer** pushes code to GitHub app repo
2. **Tekton Pipeline** is triggered (manually or via webhook)
3. **git-clone task** pulls the source code
4. **buildah task** builds the container image
5. **Image is pushed** to internal OpenShift registry
6. **Developer** updates image tag in GitOps repo (or automate this step)
7. **ArgoCD** detects change in GitOps repo
8. **ArgoCD** syncs and deploys the new version to the cluster
9. **Application** is live and accessible via Route

---

## Next Steps

- Add **webhook triggers** to start pipelines automatically on git push
- Add **image tag updates** to the pipeline (update GitOps repo automatically)
- Add **testing tasks** to the pipeline
- Configure **multiple environments** (dev, staging, prod) with ArgoCD ApplicationSets
- Add **Slack/email notifications** for pipeline status
