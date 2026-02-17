# Complete Konflux Implementation Guide

**Transform your tssc-sample-pipelines to use Konflux with centralized CI/CD**

This guide provides step-by-step instructions to implement Konflux (Red Hat Trusted Software Supply Chain) using the pipelines and resources in this repository.

---

## Table of Contents

1. [Overview](#overview)
2. [What is Konflux?](#what-is-konflux)
3. [Architecture](#architecture)
4. [Prerequisites](#prerequisites)
5. [Repository Structure](#repository-structure)
6. [Step-by-Step Setup](#step-by-step-setup)
7. [Testing the Flow](#testing-the-flow)
8. [Developer Workflow](#developer-workflow)
9. [Operations Workflow](#operations-workflow)
10. [Troubleshooting](#troubleshooting)
11. [Quick Reference](#quick-reference)

---

## Overview

This repository contains:
- **Original Pipelines**: Git tag-based promotion pipeline
- **Konflux Pipelines**: Modern snapshot/release-based pipeline
- **Konflux Resources**: All CRs needed for Konflux setup
- **Complete Documentation**: This guide

### What Changes?

| Before (Git Tags) | After (Konflux) |
|-------------------|-----------------|
| `git tag v1.0.0-stage` to promote | Automatic promotions |
| Manual promotion for every environment | Auto: dev/stage, Manual: prod only |
| No integration test gates | Tests gate all releases |
| No policy enforcement | Enterprise Contract per environment |
| Limited audit trail | Complete Snapshot/Release history |
| Git history polluted with tags | Clean git history |
| Two environments | Three environments (dev, stage, prod) |

---

## What is Konflux?

**Konflux** is Red Hat's Trusted Software Supply Chain platform that provides:

### Core Features

1. **🔐 Automatic Image Signing** (Tekton Chains)
   - Every image is cryptographically signed
   - Signatures stored in registry
   - Verifiable supply chain

2. **📋 SBOM Generation** (Trustification)
   - CycloneDX and SPDX formats
   - Automatic generation during build
   - Stored in Trustification service

3. **🛡️ Policy Enforcement** (Enterprise Contract)
   - CVE thresholds per environment
   - License compliance
   - Signature verification
   - SLSA compliance

4. **🔄 Snapshot-Based Releases**
   - Immutable image references
   - Atomic multi-component releases
   - Easy rollbacks

5. **✅ Integration Test Gates**
   - Tests run before any promotion
   - Only validated Snapshots promote
   - Configurable test scenarios

6. **📊 Complete Audit Trail**
   - Every build tracked
   - Every promotion recorded
   - Full provenance

---

## Architecture

### Centralized CI/CD with Shared Environment Namespaces

**Multi-Tenancy Model**: All applications created from the Developer Hub template deploy to shared environment namespaces.

```
┌─────────────────────────────────────────────────────────────────┐
│ tssc-app-ci (CI/CD Namespace)                                    │
│                                                                   │
│ ALL pipelines for ALL apps run here:                            │
│  • maven-build-ci-konflux (Build Pipeline)                      │
│  • release-pipeline (Release Pipeline)                          │
│  • integration-tests (Test Pipeline)                            │
│  • All build and release tasks                                  │
│                                                                   │
│ Konflux Resources (per app):                                    │
│  • Application CR (app1, app2, app3, ...)                       │
│  • Component CR (per app)                                       │
│  • Snapshots (auto-created after builds)                        │
│  • IntegrationTestScenario (per app)                            │
│  • ReleasePlan → development (auto-release: true)               │
│  • ReleasePlan → stage (auto-release: true)                     │
│  • ReleasePlan → prod (auto-release: false)                     │
│  • Release CRs (auto-created or manual)                         │
└─────────────────────────────────────────────────────────────────┘
                              ↓
        ┌─────────────────────┼─────────────────────┐
        ↓                     ↓                     ↓
┌───────────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│ tssc-app-development  │ │ tssc-app-stage  │ │ tssc-app-prod   │
│                       │ │                 │ │                 │
│ SHARED Environment:   │ │ SHARED Env:     │ │ SHARED Env:     │
│ • ReleasePlan         │ │ • ReleasePlan   │ │ • ReleasePlan   │
│   Admission (one)     │ │   Admission     │ │   Admission     │
│                       │ │                 │ │                 │
│ Multiple Apps:        │ │ Multiple Apps:  │ │ Multiple Apps:  │
│ • app1-deployment     │ │ • app1-deploy   │ │ • app1-deploy   │
│ • app1-service        │ │ • app1-service  │ │ • app1-service  │
│ • app1-route          │ │ • app1-route    │ │ • app1-route    │
│ • app2-deployment     │ │ • app2-deploy   │ │ • app2-deploy   │
│ • app2-service        │ │ • app2-service  │ │ • app2-service  │
│ • app2-route          │ │ • app2-route    │ │ • app2-route    │
│ • ...                 │ │ • ...           │ │ • ...           │
│                       │ │                 │ │                 │
│ NO pipelines!         │ │ NO pipelines!   │ │ NO pipelines!   │
└───────────────────────┘ └─────────────────┘ └─────────────────┘
```

**Key Points:**
- ✅ One set of environment namespaces for ALL applications
- ✅ Developer Hub template bootstraps new apps into existing namespaces
- ✅ Each app gets its own Deployment/Service/Route within the shared namespace
- ✅ ReleasePlanAdmission accepts releases from ALL apps (not app-specific)
- ✅ RBAC configured once per namespace (not per app)

### Complete Flow

```
Developer: git push (to any app repo)
  ↓
Build Pipeline (tssc-app-ci)
  ↓
Image: quay.io/myorg/myapp:a1b2c3d4
Digest: sha256:abc123...
  ↓
Konflux creates Snapshot (tssc-app-ci)
  containerImage: quay.io/myorg/myapp@sha256:abc123...
  ↓
Integration Tests (tssc-app-ci)
  ↓
Auto-Release to Development (tssc-app-ci)
  EC Policy: slsa3, strict=false (warn only)
  Copy: sha256:abc123... → tag:dev
  Deploy: tssc-app-development namespace
  ↓
Auto-Release to Stage (tssc-app-ci)
  EC Policy: slsa3, strict=true (enforce)
  Copy: sha256:abc123... → tag:stage
  Deploy: tssc-app-stage namespace
  ↓
Manual Release to Prod (tssc-app-ci)
  Ops creates Release CR
  EC Policy: slsa3-strict, strict=true (strictest)
  Copy: sha256:abc123... → tag:prod
  Deploy: tssc-app-prod namespace
```

---

## Prerequisites

Before you begin, ensure you have:

- ✅ **OpenShift 4.12+** cluster with admin access
- ✅ **oc CLI** installed and logged in as admin
- ✅ **Konflux Operator** will be installed in Step 1 (from community catalog)
- ✅ **OpenShift Pipelines (Tekton)** will be installed in Step 1 if not present
- ✅ **Application name** chosen (e.g., `my-java-app`)
- ✅ **Git repository** URL for your application
- ✅ **Container registry** (e.g., quay.io account with credentials)
- ✅ **GitOps repository** (optional, for deployment automation)

---

## Repository Structure

```
tssc-sample-pipelines/
├── pipelines/
│   ├── maven-build-ci.yaml              # Original build pipeline
│   ├── promote-to-env.yaml              # Original git tag promotion
│   ├── maven-build-ci-konflux.yaml      # NEW: Konflux build pipeline
│   ├── release-pipeline.yaml            # NEW: Konflux release pipeline
│   └── integration-tests.yaml           # NEW: Integration test pipeline
│
├── tasks/
│   ├── acs-*.yaml                       # Security scans
│   ├── buildah-rhtap.yaml               # Container build
│   ├── git-clone.yaml                   # Git operations
│   ├── maven.yaml                       # Maven build
│   ├── skopeo-copy.yaml                 # Image copy
│   ├── update-deployment.yaml           # GitOps update
│   ├── verify-enterprise-contract.yaml  # Policy verification
│   ├── upload-sbom-to-trustification.yaml
│   └── ... (other tasks)
│
├── konflux-resources/
│   ├── application.yaml                 # Konflux Application CR
│   ├── component.yaml                   # Konflux Component CR
│   ├── integration-test-scenario.yaml   # Integration test config
│   ├── release-plans.yaml               # 3 ReleasePlans (dev/stage/prod)
│   ├── release-plan-admission-dev.yaml  # Dev admission policy
│   ├── release-plan-admission-stage.yaml # Stage admission policy
│   └── release-plan-admission-prod.yaml  # Prod admission policy
│
└── KONFLUX-SETUP-GUIDE.md               # This file
```

---

## Step-by-Step Setup

### Step 1: Install Konflux Operator

Install the Konflux operator from the community catalog using the stable channel.

```bash
# Install Konflux Operator from community catalog
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: konflux-operator
  namespace: openshift-operators
spec:
  channel: stable-v0
  name: konflux-operator
  source: community-operators
  sourceNamespace: openshift-marketplace
  installPlanApproval: Automatic
EOF

# Wait for operator to install (this may take a few minutes)
echo "Waiting for Konflux operator to install..."
sleep 60

# Verify Konflux operator is running
oc get pods -n openshift-operators | grep konflux

# Check that Konflux CRDs are installed
oc get crd | grep appstudio

# You should see:
# applications.appstudio.redhat.com
# components.appstudio.redhat.com
# snapshots.appstudio.redhat.com
# integrationtestscenarios.appstudio.redhat.com
# releaseplans.appstudio.redhat.com
# releaseplanadmissions.appstudio.redhat.com
# releases.appstudio.redhat.com
```

**Verify OpenShift Pipelines is installed:**

```bash
# Check OpenShift Pipelines (Tekton) is running
oc get pods -n openshift-pipelines

# You should see:
# tekton-pipelines-controller-*
# tekton-pipelines-webhook-*
# tekton-chains-controller-* (for image signing)
```

If OpenShift Pipelines is not installed:

```bash
# Install OpenShift Pipelines Operator
cat <<EOF | oc apply -f -
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
  installPlanApproval: Automatic
EOF

# Wait for operator to install
sleep 60

# Verify
oc get pods -n openshift-pipelines
```

---

### Step 2: Create Shared Environment Namespaces

Create the shared environment namespaces. This is a **one-time setup** - all applications from Developer Hub will deploy to these same namespaces.

```bash
echo "Creating shared TSSC environment namespaces..."
echo "These namespaces will host ALL applications created from Developer Hub template:"
echo "  - tssc-app-ci (CI/CD namespace - all pipelines)"
echo "  - tssc-app-development (shared development environment)"
echo "  - tssc-app-stage (shared staging environment)"
echo "  - tssc-app-prod (shared production environment)"

# Create CI/CD namespace
oc create namespace tssc-app-ci

# Create shared environment namespaces
oc create namespace tssc-app-development
oc create namespace tssc-app-stage
oc create namespace tssc-app-prod

# Label namespaces for easier identification
oc label namespace tssc-app-ci \
  environment=ci \
  app.kubernetes.io/part-of=tssc \
  konflux.dev/multi-tenant=true

oc label namespace tssc-app-development \
  environment=development \
  app.kubernetes.io/part-of=tssc \
  konflux.dev/multi-tenant=true

oc label namespace tssc-app-stage \
  environment=stage \
  app.kubernetes.io/part-of=tssc \
  konflux.dev/multi-tenant=true

oc label namespace tssc-app-prod \
  environment=production \
  app.kubernetes.io/part-of=tssc \
  konflux.dev/multi-tenant=true

# Verify namespaces created
oc get namespaces | grep tssc-app
```

Expected output:
```
tssc-app-ci            Active   1m
tssc-app-development   Active   1m
tssc-app-stage         Active   1m
tssc-app-prod          Active   1m
```

**Important Note:** These namespaces are shared across ALL applications. When you create a new application from the Developer Hub template, it will deploy to these existing namespaces alongside other applications.

---

### Step 3: Setup RBAC (Cross-Namespace Permissions)

The release pipeline runs in `tssc-app-ci` but needs to deploy to the shared environment namespaces. This is a **one-time setup** for all applications.

```bash
# Create service account in CI namespace
oc create serviceaccount release-service-account -n tssc-app-ci

# Grant permissions to deploy to development environment
oc create rolebinding release-deployer \
  --serviceaccount=tssc-app-ci:release-service-account \
  --clusterrole=edit \
  -n tssc-app-development

# Grant permissions to deploy to stage environment
oc create rolebinding release-deployer \
  --serviceaccount=tssc-app-ci:release-service-account \
  --clusterrole=edit \
  -n tssc-app-stage

# Grant permissions to deploy to prod environment
oc create rolebinding release-deployer \
  --serviceaccount=tssc-app-ci:release-service-account \
  --clusterrole=edit \
  -n tssc-app-prod

# If you have registry credentials, link them
# oc create secret docker-registry quay-credentials \
#   --docker-server=quay.io \
#   --docker-username=YOUR_USERNAME \
#   --docker-password=YOUR_PASSWORD \
#   -n tssc-app-ci
# oc secrets link release-service-account quay-credentials -n tssc-app-ci

# Create GitOps credentials (if using GitOps)
# oc create secret generic gitops-auth \
#   --from-literal=username=YOUR_GIT_USER \
#   --from-literal=password=YOUR_GIT_TOKEN \
#   -n tssc-app-ci
# oc secrets link release-service-account gitops-auth -n tssc-app-ci

# Verify RBAC setup
echo "Verifying RBAC configuration..."
oc get rolebindings -n tssc-app-development | grep release-deployer
oc get rolebindings -n tssc-app-stage | grep release-deployer
oc get rolebindings -n tssc-app-prod | grep release-deployer
oc get serviceaccount release-service-account -n tssc-app-ci
```

**Note:** This RBAC setup allows the release-service-account to deploy ANY application to the shared environments. This is intentional for the multi-tenant model.

---

### Step 4: Install Pipelines and Tasks

Install all pipelines and tasks in the `tssc-app-ci` namespace:

```bash
# Navigate to the repository directory
cd /path/to/tssc-sample-pipelines

# Install all tasks (these are shared by all pipelines)
echo "Installing tasks in tssc-app-ci..."
oc apply -f tasks/ -n tssc-app-ci

# Install Konflux pipelines
echo "Installing Konflux pipelines in tssc-app-ci..."
oc apply -f pipelines/maven-build-ci-konflux.yaml -n tssc-app-ci
oc apply -f pipelines/release-pipeline.yaml -n tssc-app-ci
oc apply -f pipelines/integration-tests.yaml -n tssc-app-ci

# Verify installation
echo "Verifying installation..."
echo ""
echo "Tasks installed:"
oc get tasks -n tssc-app-ci
echo ""
echo "Pipelines installed:"
oc get pipelines -n tssc-app-ci

# You should see:
# Pipelines:
#   - maven-build-ci-konflux
#   - release-pipeline
#   - integration-tests
```

---

### Step 5: Configure and Apply Konflux Resources

Now we'll create all the Konflux CRs. These tell Konflux about your application.

#### 5.1: Edit and Apply Application

```bash
# Set your application details
export APP_DISPLAY_NAME="My Java Application"
export APP_DESCRIPTION="Java application built with Maven"

# Create Application CR
cat <<EOF | oc apply -f -
apiVersion: appstudio.redhat.com/v1alpha1
kind: Application
metadata:
  name: ${APP_NAME}
  namespace: tssc-app-ci
  annotations:
    description: "${APP_DESCRIPTION}"
spec:
  displayName: "${APP_DISPLAY_NAME}"
  description: "${APP_DESCRIPTION}"
EOF

# Verify
oc get application ${APP_NAME} -n tssc-app-ci
```

#### 5.2: Edit and Apply Component

```bash
# Set your component details
export COMPONENT_NAME="${APP_NAME}-backend"
export GIT_URL="https://gitlab.example.com/myorg/my-java-app"  # CHANGE THIS
export GIT_REVISION="main"  # or "master"
export IMAGE_REPO="quay.io/myorg/my-java-app-backend"  # CHANGE THIS (without tag)

# Create Component CR
cat <<EOF | oc apply -f -
apiVersion: appstudio.redhat.com/v1alpha1
kind: Component
metadata:
  name: ${COMPONENT_NAME}
  namespace: tssc-app-ci
  annotations:
    description: "Backend service for ${APP_NAME}"
spec:
  application: ${APP_NAME}
  componentName: ${COMPONENT_NAME}
  source:
    git:
      url: ${GIT_URL}
      revision: ${GIT_REVISION}
      context: ./
  build:
    containerImage: ${IMAGE_REPO}
  buildPipelineRef:
    name: maven-build-ci-konflux
EOF

# Verify
oc get component ${COMPONENT_NAME} -n tssc-app-ci
oc describe component ${COMPONENT_NAME} -n tssc-app-ci
```

#### 5.3: Apply Integration Test Scenario

```bash
# Create IntegrationTestScenario
cat <<EOF | oc apply -f -
apiVersion: appstudio.redhat.com/v1beta1
kind: IntegrationTestScenario
metadata:
  name: integration-tests
  namespace: tssc-app-ci
  annotations:
    description: "Integration tests that run after build completes"
spec:
  application: ${APP_NAME}
  resolverRef:
    resolver: cluster
    params:
      - name: name
        value: integration-tests
      - name: namespace
        value: tssc-app-ci
      - name: kind
        value: Pipeline
  contexts:
    - name: integration
      description: "Run integration tests on newly built images"
EOF

# Verify
oc get integrationtestscenario integration-tests -n tssc-app-ci
```

#### 5.4: Apply ReleasePlans (All 3)

**Note:** Each application needs its own set of ReleasePlans, but they all target the same shared environment namespaces.

```bash
# Create ReleasePlan for Development (auto-release)
cat <<EOF | oc apply -f -
apiVersion: appstudio.redhat.com/v1alpha1
kind: ReleasePlan
metadata:
  name: ${APP_NAME}-release-to-development
  namespace: tssc-app-ci
  labels:
    release.appstudio.openshift.io/auto-release: "true"
    app.kubernetes.io/part-of: ${APP_NAME}
  annotations:
    description: "Automatically release ${APP_NAME} to development after tests pass"
spec:
  application: ${APP_NAME}
  target: tssc-app-development
  pipelineRef:
    name: release-pipeline
    namespace: tssc-app-ci
  serviceAccount: release-service-account
EOF

# Create ReleasePlan for Stage (auto-release)
cat <<EOF | oc apply -f -
apiVersion: appstudio.redhat.com/v1alpha1
kind: ReleasePlan
metadata:
  name: ${APP_NAME}-release-to-stage
  namespace: tssc-app-ci
  labels:
    release.appstudio.openshift.io/auto-release: "true"
    app.kubernetes.io/part-of: ${APP_NAME}
  annotations:
    description: "Automatically release ${APP_NAME} to stage after development"
spec:
  application: ${APP_NAME}
  target: tssc-app-stage
  pipelineRef:
    name: release-pipeline
    namespace: tssc-app-ci
  serviceAccount: release-service-account
EOF

# Create ReleasePlan for Prod (manual only)
cat <<EOF | oc apply -f -
apiVersion: appstudio.redhat.com/v1alpha1
kind: ReleasePlan
metadata:
  name: ${APP_NAME}-release-to-prod
  namespace: tssc-app-ci
  labels:
    release.appstudio.openshift.io/auto-release: "false"
    app.kubernetes.io/part-of: ${APP_NAME}
  annotations:
    description: "Manual release of ${APP_NAME} to production (requires approval)"
spec:
  application: ${APP_NAME}
  target: tssc-app-prod
  pipelineRef:
    name: release-pipeline
    namespace: tssc-app-ci
  serviceAccount: release-service-account
EOF

# Verify all three ReleasePlans
oc get releaseplans -n tssc-app-ci -l app.kubernetes.io/part-of=${APP_NAME}
```

Expected output:
```
NAME                                  AUTO-RELEASE   TARGET
my-java-app-release-to-development    true           tssc-app-development
my-java-app-release-to-stage          true           tssc-app-stage
my-java-app-release-to-prod           false          tssc-app-prod
```

#### 5.5: Apply ReleasePlanAdmissions (One per Environment)

**IMPORTANT:** ReleasePlanAdmissions are created **once per environment namespace** and accept releases from ALL applications. This is a one-time setup, not per-app.

**Development Environment ReleasePlanAdmission:**

```bash
cat <<EOF | oc apply -f -
apiVersion: appstudio.redhat.com/v1alpha1
kind: ReleasePlanAdmission
metadata:
  name: accept-releases
  namespace: tssc-app-development
  annotations:
    description: "Accept releases from tssc-app-ci to development environment"
spec:
  origin: tssc-app-ci
  # NO applications filter - accepts releases from ALL apps
  environment: development
  pipelineRef:
    name: release-pipeline
    namespace: tssc-app-ci
  serviceAccount: release-service-account
  data:
    enterpriseContract:
      policy: "github.com/enterprise-contract/config//slsa3"
      publicKey: "k8s://openshift-pipelines/signing-secrets"
      strict: "false"  # Warn only in development
EOF

# Verify
oc get releaseplanadmission accept-releases -n tssc-app-development
```

**Stage Environment ReleasePlanAdmission:**

```bash
cat <<EOF | oc apply -f -
apiVersion: appstudio.redhat.com/v1alpha1
kind: ReleasePlanAdmission
metadata:
  name: accept-releases
  namespace: tssc-app-stage
  annotations:
    description: "Accept releases from tssc-app-ci to stage environment"
spec:
  origin: tssc-app-ci
  # NO applications filter - accepts releases from ALL apps
  environment: stage
  pipelineRef:
    name: release-pipeline
    namespace: tssc-app-ci
  serviceAccount: release-service-account
  data:
    enterpriseContract:
      policy: "github.com/enterprise-contract/config//slsa3"
      publicKey: "k8s://openshift-pipelines/signing-secrets"
      strict: "true"  # Enforce in stage
EOF

# Verify
oc get releaseplanadmission accept-releases -n tssc-app-stage
```

**Production Environment ReleasePlanAdmission:**

```bash
cat <<EOF | oc apply -f -
apiVersion: appstudio.redhat.com/v1alpha1
kind: ReleasePlanAdmission
metadata:
  name: accept-releases
  namespace: tssc-app-prod
  annotations:
    description: "Accept releases from tssc-app-ci to production environment"
spec:
  origin: tssc-app-ci
  # NO applications filter - accepts releases from ALL apps
  environment: production
  pipelineRef:
    name: release-pipeline
    namespace: tssc-app-ci
  serviceAccount: release-service-account
  data:
    enterpriseContract:
      policy: "github.com/enterprise-contract/config//slsa3-strict"
      publicKey: "k8s://openshift-pipelines/signing-secrets"
      strict: "true"  # Strictest enforcement in production
EOF

# Verify
oc get releaseplanadmission accept-releases -n tssc-app-prod
```

**Verify all three:**

```bash
echo "Checking ReleasePlanAdmissions in all environments..."
oc get releaseplanadmission -n tssc-app-development
oc get releaseplanadmission -n tssc-app-stage
oc get releaseplanadmission -n tssc-app-prod
```

**Note:** These ReleasePlanAdmissions do not filter by application name, so they will accept releases for ANY application from tssc-app-ci. This is intentional for the shared environment model.

---

### Step 6: Update Your Application Repository

In your **application repository** (not tssc-sample-pipelines), update the Pipelines as Code trigger.

**Edit `.tekton/on-push.yaml`:**

```yaml
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  generateName: ${APP_NAME}-build-
  namespace: tssc-app-ci  # ← IMPORTANT: Pipeline runs in CI namespace
  annotations:
    pipelinesascode.tekton.dev/on-cel-expression: |
      body.object_kind == "push" && body.ref == "refs/heads/main"
    pipelinesascode.tekton.dev/pipeline: "https://gitlab.example.com/rhdh/tssc-sample-pipelines/-/raw/konflux/pipelines/maven-build-ci-konflux.yaml"
    # Add all your task annotations here...
  labels:
    # CRITICAL: These labels link PipelineRun to Konflux Component
    appstudio.openshift.io/application: ${APP_NAME}
    appstudio.openshift.io/component: ${COMPONENT_NAME}
    backstage.io/kubernetes-id: ${APP_NAME}
spec:
  params:
    - name: component-name
      value: ${COMPONENT_NAME}
    - name: git-url
      value: '{{repo_url}}'
    - name: output-image
      value: "${IMAGE_REPO}:{{revision}}"  # Tag with git commit SHA
    - name: revision
      value: '{{revision}}'
    - name: event-type
      value: '{{event_type}}'

  pipelineRef:
    resolver: cluster
    params:
      - name: name
        value: maven-build-ci-konflux  # ← NEW Konflux pipeline
      - name: namespace
        value: tssc-app-ci  # ← Pipeline in CI namespace
      - name: kind
        value: Pipeline

  workspaces:
    - name: workspace
      volumeClaimTemplate:
        spec:
          accessModes: [ReadWriteOnce]
          resources:
            requests:
              storage: 1Gi
    - name: maven-settings
      emptyDir: {}
    - name: git-auth
      secret:
        secretName: git-auth-secret
```

**Commit and push:**

```bash
cd your-app-repo
git add .tekton/on-push.yaml
git commit -m "Update to use Konflux pipeline"
git push origin main
```

---

## Testing the Flow

### Test 1: Trigger a Build

```bash
# Make a small change to trigger the build
cd your-app-repo
echo "# Konflux test" >> README.md
git add README.md
git commit -m "Test Konflux flow"
git push origin main
```

### Test 2: Watch the Build Pipeline

```bash
# Watch builds in real-time
oc get pipelineruns -n tssc-app-ci -w

# You should see:
# NAME                              STATUS    AGE
# my-java-app-build-xxxxx           Running   10s
# my-java-app-build-xxxxx           Succeeded 5m
```

### Test 3: Watch Snapshot Creation

```bash
# In a new terminal, watch for Snapshot creation
oc get snapshots -n tssc-app-ci -w

# You should see:
# NAME                           STATUS   AGE
# my-java-app-snapshot-xxxxx     Pending  5s
# my-java-app-snapshot-xxxxx     Passed   2m
```

### Test 4: Watch Integration Tests

```bash
# Check integration test runs
oc get pipelineruns -n tssc-app-ci | grep integration-tests

# View test logs
oc logs -f pipelinerun/<integration-test-pipelinerun> -n tssc-app-ci
```

### Test 5: Watch Auto-Release to Dev

```bash
# Watch releases
oc get releases -n tssc-app-ci -w

# You should see:
# NAME                                    STATUS    TARGET
# my-java-app-snapshot-xxxxx-dev-auto     Running   10s
# my-java-app-snapshot-xxxxx-dev-auto     Succeeded my-java-app-dev
```

### Test 6: Watch Auto-Release to Stage

```bash
# After dev succeeds, stage release is created
oc get releases -n tssc-app-ci | grep stage

# NAME                                      STATUS    TARGET
# my-java-app-snapshot-xxxxx-stage-auto     Succeeded my-java-app-stage
```

### Test 7: Verify Deployments

```bash
# Check what's deployed in dev
oc get pods -n tssc-app-development
oc get deployment -n tssc-app-development -o yaml | grep image:

# Check what's deployed in stage
oc get pods -n tssc-app-stage
oc get deployment -n tssc-app-stage -o yaml | grep image:

# Prod should be empty (no auto-release)
oc get pods -n tssc-app-prod
```

---

## Developer Workflow

### Daily Development

Developers only need to push code. Everything else is automatic.

```bash
# 1. Developer makes changes
cd my-java-app
vim src/main/java/com/example/MyService.java

# 2. Commit and push
git add .
git commit -m "Add new authentication feature"
git push origin main

# That's it! Konflux handles:
# → Build (tssc-app-ci)
# → Snapshot creation (tssc-app-ci)
# → Integration tests (tssc-app-ci)
# → Auto-release to dev (if tests pass)
# → Auto-release to stage (if dev succeeds)
```

### Check Build Status

```bash
# See all builds
oc get pipelineruns -n tssc-app-ci

# See latest build
oc get pipelineruns -n tssc-app-ci --sort-by=.metadata.creationTimestamp | tail -1

# View build logs
oc logs -f pipelinerun/<name> -n tssc-app-ci
```

### Check Snapshots

```bash
# List all snapshots
oc get snapshots -n tssc-app-ci

# Get latest snapshot
oc get snapshots -n tssc-app-ci --sort-by=.metadata.creationTimestamp | tail -1

# View snapshot details
oc describe snapshot <name> -n tssc-app-ci
```

### Check What's Deployed

```bash
# Dev environment
oc get deployment -n tssc-app-development -o jsonpath='{.items[0].spec.template.spec.containers[0].image}'

# Stage environment
oc get deployment -n tssc-app-stage -o jsonpath='{.items[0].spec.template.spec.containers[0].image}'

# Prod environment
oc get deployment -n tssc-app-prod -o jsonpath='{.items[0].spec.template.spec.containers[0].image}'
```

---

## Operations Workflow

### Manual Release to Production

When ready to promote a validated snapshot to production:

```bash
# 1. List validated snapshots
oc get snapshots -n tssc-app-ci --sort-by=.metadata.creationTimestamp

# 2. Check which snapshot is currently in stage
oc get releases -n tssc-app-ci | grep stage | tail -1

# 3. Get snapshot details
SNAPSHOT_NAME="my-java-app-snapshot-xyz123"  # Choose from step 1
oc describe snapshot ${SNAPSHOT_NAME} -n tssc-app-ci

# 4. Create production release
cat <<EOF | oc apply -f -
apiVersion: appstudio.redhat.com/v1alpha1
kind: Release
metadata:
  name: prod-release-$(date +%Y%m%d-%H%M)
  namespace: tssc-app-ci
  annotations:
    release.appstudio.io/approved-by: "ops.team@example.com"
    release.appstudio.io/jira-ticket: "PROD-12345"
    release.appstudio.io/change-window: "2026-02-17 14:00 UTC"
    release.appstudio.io/validated-in: "stage"
spec:
  snapshot: ${SNAPSHOT_NAME}
  releasePlan: release-to-prod
EOF

# 5. Monitor the release
oc get release -n tssc-app-ci -w

# 6. Watch release pipeline
oc get pipelineruns -n tssc-app-ci -w

# 7. Verify production deployment
oc get pods -n tssc-app-prod
oc get deployment -n tssc-app-prod -o yaml | grep image:
```

### Rollback

To rollback to a previous version:

```bash
# 1. Find previous releases
oc get releases -n tssc-app-ci -l release.appstudio.io/target=tssc-app-prod \
  --sort-by=.metadata.creationTimestamp

# 2. Get the snapshot from previous successful release
PREV_RELEASE="prod-release-20260217-1030"
PREV_SNAPSHOT=$(oc get release ${PREV_RELEASE} -n tssc-app-ci \
  -o jsonpath='{.spec.snapshot}')

# 3. Create rollback release
cat <<EOF | oc apply -f -
apiVersion: appstudio.redhat.com/v1alpha1
kind: Release
metadata:
  name: prod-rollback-$(date +%Y%m%d-%H%M)
  namespace: tssc-app-ci
  annotations:
    release.appstudio.io/rollback: "true"
    release.appstudio.io/rollback-from: "${PREV_RELEASE}"
    release.appstudio.io/approved-by: "ops.team@example.com"
spec:
  snapshot: ${PREV_SNAPSHOT}
  releasePlan: release-to-prod
EOF

# Monitor rollback
oc get release -n tssc-app-ci -w
```

---

## Troubleshooting

### Issue: Build Doesn't Trigger

**Symptoms:** Push code but no PipelineRun created

**Solutions:**

```bash
# 1. Check webhook configuration
oc get repository -A

# 2. Verify Pipelines as Code is working
oc get pods -n pipelines-as-code

# 3. Check PipelineRun for errors
oc get events -n tssc-app-ci --sort-by='.lastTimestamp'
```

### Issue: No Snapshot Created After Build

**Symptoms:** Build succeeds but no Snapshot appears

**Solutions:**

```bash
# 1. Check build results
oc get pipelinerun <name> -n tssc-app-ci -o yaml | grep -A 10 "pipelineResults:"

# Must have IMAGE_URL and IMAGE_DIGEST results

# 2. Check PipelineRun labels
oc get pipelinerun <name> -n tssc-app-ci -o yaml | grep "appstudio.openshift.io"

# Must have:
#   appstudio.openshift.io/application: my-java-app
#   appstudio.openshift.io/component: my-java-app-backend

# 3. Verify Component exists with matching name
oc get component -n tssc-app-ci

# 4. Check Konflux operator logs
oc logs -n konflux-system deployment/appstudio-controller-manager
```

### Issue: Integration Tests Don't Run

**Symptoms:** Snapshot created but tests don't start

**Solutions:**

```bash
# 1. Verify IntegrationTestScenario exists
oc get integrationtestscenario -n tssc-app-ci

# 2. Check it references correct application
oc describe integrationtestscenario integration-tests -n tssc-app-ci

# 3. Check integration-service logs
oc logs -n konflux-system deployment/integration-service-controller-manager

# 4. Verify integration test pipeline exists
oc get pipeline integration-tests -n tssc-app-ci
```

### Issue: Release Fails

**Symptoms:** Release CR created but pipeline fails

**Solutions:**

```bash
# 1. Check Release status
oc describe release <name> -n tssc-app-ci

# 2. View release pipeline logs
RELEASE_PIPELINERUN=$(oc get pipelineruns -n tssc-app-ci \
  -l release.appstudio.openshift.io/release=<release-name> \
  -o name)
oc logs -f $RELEASE_PIPELINERUN -n tssc-app-ci

# 3. Check ReleasePlanAdmission exists in target namespace
oc get releaseplanadmission -n tssc-app-development
oc get releaseplanadmission -n tssc-app-stage
oc get releaseplanadmission -n tssc-app-prod

# 4. Verify RBAC permissions
oc get rolebindings -n tssc-app-development | grep release-deployer
oc get rolebindings -n tssc-app-stage | grep release-deployer
oc get rolebindings -n tssc-app-prod | grep release-deployer

# 5. Check service account
oc get serviceaccount release-service-account -n tssc-app-ci
```

### Issue: Enterprise Contract Fails

**Symptoms:** Release blocked by policy violations

**Solutions:**

```bash
# 1. View Enterprise Contract task logs
oc logs pipelinerun/<release-pipeline> \
  -c step-verify-enterprise-contract \
  -n tssc-app-ci

# 2. Common failures and fixes:

# CVEs found:
# - Update base image to patched version
# - Review CVE severity (can relax policy for dev/stage)

# Signature verification failed:
# - Check Tekton Chains is running
# - Verify signing secrets exist
oc get secret signing-secrets -n openshift-pipelines

# SBOM missing:
# - Check buildah-rhtap task completed
# - Verify upload-sbom-to-trustification task ran

# Policy violation:
# - Review policy in ReleasePlanAdmission
# - Can use strict: "false" for dev to warn only
```

### Issue: Cross-Namespace Permission Denied

**Symptoms:** Release pipeline can't deploy to target namespace

**Solutions:**

```bash
# 1. Verify service account exists
oc get serviceaccount release-service-account -n tssc-app-ci

# 2. Check RoleBindings in target namespaces
oc get rolebinding release-deployer -n tssc-app-development -o yaml
oc get rolebinding release-deployer -n tssc-app-stage -o yaml
oc get rolebinding release-deployer -n tssc-app-prod -o yaml

# 3. Recreate RoleBinding if needed
oc create rolebinding release-deployer \
  --serviceaccount=tssc-app-ci:release-service-account \
  --clusterrole=edit \
  -n tssc-app-development --dry-run=client -o yaml | oc apply -f -
```

---

## Quick Reference

### Common Commands

```bash
# Watch builds
oc get pipelineruns -n tssc-app-ci -w

# Watch snapshots
oc get snapshots -n tssc-app-ci -w

# Watch releases
oc get releases -n tssc-app-ci -w

# Check deployments
oc get pods -n tssc-app-development
oc get pods -n tssc-app-stage
oc get pods -n tssc-app-prod

# View logs
oc logs -f pipelinerun/<name> -n tssc-app-ci

# List all Konflux resources
oc get applications -n tssc-app-ci
oc get components -n tssc-app-ci
oc get integrationtestscenarios -n tssc-app-ci
oc get releaseplans -n tssc-app-ci
oc get releaseplanadmissions -n tssc-app-development
oc get releaseplanadmissions -n tssc-app-stage
oc get releaseplanadmissions -n tssc-app-prod
```

### Resource Locations

| Resource | Namespace | Count | Notes |
|----------|-----------|-------|-------|
| Application | tssc-app-ci | 1 per app | Each app from Developer Hub |
| Component | tssc-app-ci | 1+ per app | Components for each app |
| Build Pipeline | tssc-app-ci | 1 (shared) | maven-build-ci-konflux |
| Release Pipeline | tssc-app-ci | 1 (shared) | release-pipeline |
| Integration Pipeline | tssc-app-ci | 1 (shared) | integration-tests |
| Snapshot | tssc-app-ci | Many | Per app build |
| ReleasePlan | tssc-app-ci | 3 per app | dev/stage/prod per app |
| Release | tssc-app-ci | Many | Per app release |
| ReleasePlanAdmission | tssc-app-development | 1 (shared) | Accepts ALL apps |
| ReleasePlanAdmission | tssc-app-stage | 1 (shared) | Accepts ALL apps |
| ReleasePlanAdmission | tssc-app-prod | 1 (shared) | Accepts ALL apps |
| Deployment | tssc-app-development | 1+ | One per app |
| Deployment | tssc-app-stage | 1+ | One per app |
| Deployment | tssc-app-prod | 1+ | One per app |

### Image Tags

After build:
```
quay.io/myorg/myapp:a1b2c3d4    ← Git commit SHA
                   @sha256:abc123...  ← Digest (immutable)
```

After releases:
```
quay.io/myorg/myapp:dev    → sha256:abc123...
quay.io/myorg/myapp:stage  → sha256:abc123...
quay.io/myorg/myapp:prod   → sha256:def456...
```

### Enterprise Contract Policies

| Environment | Namespace | Policy | Strict | Behavior |
|-------------|-----------|--------|--------|----------|
| Development | tssc-app-development | slsa3 | false | Warnings only, allows promotion |
| Stage | tssc-app-stage | slsa3 | true | Blocks on violations |
| Production | tssc-app-prod | slsa3-strict | true | Strictest enforcement |

---

## Additional Resources

- **Konflux Documentation**: https://konflux-ci.dev/
- **Enterprise Contract**: https://enterprisecontract.dev/
- **SLSA Framework**: https://slsa.dev/
- **Tekton Pipelines**: https://tekton.dev/
- **Tekton Chains**: https://tekton.dev/docs/chains/

---

## Summary

You now have:

- ✅ **Centralized CI/CD** in tssc-app-ci namespace
- ✅ **Three runtime environments** (dev, stage, prod)
- ✅ **Automatic promotions** (dev, stage)
- ✅ **Manual production releases** with approval
- ✅ **Integration testing gates**
- ✅ **Policy enforcement** (Enterprise Contract)
- ✅ **Image signing and verification** (Tekton Chains)
- ✅ **SBOM generation and storage** (Trustification)
- ✅ **Complete audit trail** (Snapshots, Releases)

### What Changed

| Aspect | Before | After |
|--------|--------|-------|
| Promotions | Git tags | Automatic (Snapshots/Releases) |
| Environments | 2 (stage, prod) | 3 (dev, stage, prod) |
| Testing | Manual | Automatic integration tests |
| Policy | None | Enterprise Contract per env |
| Audit | Limited | Complete Snapshot/Release trail |
| Git History | Polluted with tags | Clean |

### Developer Experience

**Before:**
```bash
git push
git tag v1.0.0-stage && git push --tags
# Create GitLab Release for prod
```

**After:**
```bash
git push
# Everything automated for dev/stage
# Ops creates Release CR for prod
```

---

**Congratulations!** Your Konflux setup is complete. 🎉

For questions or issues, refer to the [Troubleshooting](#troubleshooting) section or check the [Konflux documentation](https://konflux-ci.dev/).
