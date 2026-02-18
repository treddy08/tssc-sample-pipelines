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

## Key Implementation Approach

This guide uses the **manual installation approach** based on the logic from Red Hat's `tssc-cli` tool. Key highlights:

- ✅ **Dedicated Operator Namespace**: Konflux operator runs in `konflux-operator` namespace (not `openshift-operators`)
- ✅ **Service Namespaces**: Konflux automatically creates service namespaces (`build-service`, `integration-service`, `image-controller`)
- ✅ **Integration Secrets**: Pre-configure GitLab and Quay integration secrets before creating Konflux instance
- ✅ **Enhanced RBAC**: Comprehensive permissions including cluster-wide Konflux resource access and secret distribution
- ✅ **GitLab Integration**: Pipelines as Code configured with self-hosted GitLab webhooks for automatic build triggering

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
- ✅ **Git repository** URL for your application in self-hosted GitLab
- ✅ **Container registry** - Self-hosted Quay registry with robot account credentials
- ✅ **Quay robot accounts**:
  - Read-write robot account (e.g., `tssc+tssc_rw`)
  - Read-only robot account (e.g., `tssc+tssc_ro`)
- ✅ **GitLab details**:
  - Self-hosted GitLab hostname and port
  - GitLab group name
  - GitLab username
  - GitLab Personal Access Token with `api`, `read_repository`, and `write_repository` scopes
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
# Create namespace for Konflux operator
oc create namespace konflux-operator

# Create OperatorGroup for the namespace
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: konflux-operator-group
  namespace: konflux-operator
spec:
  targetNamespaces:
  - konflux-operator
EOF

# Install Konflux Operator from community catalog
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: konflux-operator
  namespace: konflux-operator
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
oc get pods -n konflux-operator

# Check that Konflux CRDs are installed
oc get crd | grep konflux

# You should see:
# konfluxes.konflux.konflux-ci.dev

# Also check for AppStudio CRDs
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

### Step 2: Verify Integration Secrets

The required integration secrets for GitLab and Quay already exist in the `tssc` namespace. Verify they are present.

```bash
# Verify integration secrets exist in tssc namespace
oc get secrets -n tssc | grep tssc

# You should see:
# tssc-gitlab-integration
# tssc-quay-integration

# Verify GitLab secret contents
oc get secret tssc-gitlab-integration -n tssc -o jsonpath='{.data}' | jq

# Verify Quay secret contents
oc get secret tssc-quay-integration -n tssc -o jsonpath='{.data}' | jq
```

**Expected Secret Structure:**

**`tssc-gitlab-integration` secret** should contain:
- `clientId` - OAuth client ID (optional, can be empty)
- `clientSecret` - OAuth client secret (optional, can be empty)
- `group` - GitLab group name (e.g., `rhdh`)
- `host` - GitLab hostname (e.g., `gitlab-gitlab.apps.cluster-c76tb.dynamic.redhatworkshops.io`)
- `port` - GitLab port (e.g., `443`)
- `token` - GitLab Personal Access Token with `api`, `read_repository`, and `write_repository` scopes
- `username` - GitLab username (e.g., `root`)

**`tssc-quay-integration` secret** should contain:
- `.dockerconfigjson` - Docker config JSON for read-write access (contains username like `tssc+tssc_rw`)
- `.dockerconfigjsonreadonly` - Docker config JSON for read-only access
- `token` - Quay robot account token
- `url` - Quay registry URL (e.g., `https://quay-c76tb-1.apps.cluster-c76tb.dynamic.redhatworkshops.io`)

**Note:** The `image-controller` namespace requires a separate `quaytoken` secret with ONLY two fields:
- `quaytoken` - Extracted from `token` field in `tssc-quay-integration`
- `organization` - Extracted from username in `.dockerconfigjson`
  - Username format: `organization+robot_account` (e.g., `tssc+tssc_rw`)
  - Organization value: Part before the `+` (e.g., `tssc`)
  - **For this deployment, organization must be: `tssc`**

The `quaytoken` secret does NOT include `.dockerconfigjson`, `.dockerconfigjsonreadonly`, `token`, or `url` fields.

**Note:** These secrets will be used by Konflux:
- **GitLab secret**:
  - `host` and `port` - Construct GitLab URL for API access
  - `token` - Authenticate with GitLab API for webhook and repository operations
  - `username` and `group` - GitLab user and group context
  - `clientId` and `clientSecret` - OAuth authentication (optional)
- **Quay secret**:
  - `.dockerconfigjson` - Docker credentials for pulling/pushing images (read-write)
  - `.dockerconfigjsonreadonly` - Docker credentials for pulling images only (read-only)
  - `token` - Quay API token for registry operations
  - `url` - Quay registry URL

<details>
<summary>If you need to create or update these secrets, click here</summary>

```bash
# Create GitLab integration secret
# Replace with your self-hosted GitLab details
export GITLAB_HOST="gitlab-gitlab.apps.cluster-c76tb.dynamic.redhatworkshops.io"
export GITLAB_PORT="443"
export GITLAB_GROUP="rhdh"  # Your GitLab group
export GITLAB_TOKEN="glpat-xxxxxxxxxxxxxxxxxxxx"  # Personal Access Token
export GITLAB_USERNAME="root"  # Your GitLab username

# Create or update GitLab integration secret
oc create secret generic tssc-gitlab-integration \
  --from-literal=clientId="" \
  --from-literal=clientSecret="" \
  --from-literal=group="${GITLAB_GROUP}" \
  --from-literal=host="${GITLAB_HOST}" \
  --from-literal=port="${GITLAB_PORT}" \
  --from-literal=token="${GITLAB_TOKEN}" \
  --from-literal=username="${GITLAB_USERNAME}" \
  -n tssc \
  --dry-run=client -o yaml | oc apply -f -

# Create Quay integration secret
# Replace with your Quay registry details
export QUAY_URL="https://quay-c76tb-1.apps.cluster-c76tb.dynamic.redhatworkshops.io"
export QUAY_TOKEN="your-quay-robot-token"
export QUAY_USERNAME="tssc+tssc_rw"  # Your Quay robot account username
export QUAY_PASSWORD="your-quay-robot-password"  # Your Quay robot account password
export QUAY_EMAIL="admin@apps.cluster-c76tb.dynamic.redhatworkshops.io"

# Create Docker config JSON for read-write access
export DOCKER_CONFIG_RW=$(echo -n "{\"auths\": {\"${QUAY_URL#https://}\": {\"username\": \"${QUAY_USERNAME}\", \"password\": \"${QUAY_PASSWORD}\", \"email\": \"${QUAY_EMAIL}\", \"auth\": \"$(echo -n ${QUAY_USERNAME}:${QUAY_PASSWORD} | base64 -w0)\"}}}" | base64 -w0)

# For read-only, use tssc+tssc_ro username with its password
export QUAY_USERNAME_RO="tssc+tssc_ro"
export QUAY_PASSWORD_RO="your-quay-readonly-password"
export DOCKER_CONFIG_RO=$(echo -n "{\"auths\": {\"${QUAY_URL#https://}\": {\"username\": \"${QUAY_USERNAME_RO}\", \"password\": \"${QUAY_PASSWORD_RO}\", \"email\": \"${QUAY_EMAIL}\", \"auth\": \"$(echo -n ${QUAY_USERNAME_RO}:${QUAY_PASSWORD_RO} | base64 -w0)\"}}}" | base64 -w0)

# Create or update Quay integration secret
oc create secret generic tssc-quay-integration \
  --from-literal=.dockerconfigjson="${DOCKER_CONFIG_RW}" \
  --from-literal=.dockerconfigjsonreadonly="${DOCKER_CONFIG_RO}" \
  --from-literal=token="${QUAY_TOKEN}" \
  --from-literal=url="${QUAY_URL}" \
  -n tssc \
  --dry-run=client -o yaml | oc apply -f -
```

**GitLab Personal Access Token Requirements:**
To create a Personal Access Token in GitLab:
1. Navigate to GitLab → User Settings → Access Tokens
2. Create a new token with the following scopes:
   - `api` - Full API access
   - `read_repository` - Read repository contents
   - `write_repository` - Update repository (for commit status updates)
3. Set an appropriate expiration date
4. Save the token securely (you won't be able to see it again)

</details>

---

### Step 2.1: Pre-create Image Controller Namespace and Secret

**IMPORTANT**: Create the `image-controller` namespace and `quaytoken` secret **before** creating the Konflux instance. This prevents image-controller pods from failing on startup.

```bash
# Create image-controller namespace in advance
echo "Pre-creating image-controller namespace..."
oc create namespace image-controller

# Label the namespace
oc label namespace image-controller \
  app.kubernetes.io/part-of=konflux \
  app.kubernetes.io/managed-by=konflux

# Create quaytoken secret in image-controller namespace
# The secret should ONLY contain 'quaytoken' and 'organization' fields
echo "Creating quaytoken secret in image-controller namespace..."

# Get the token value from tssc-quay-integration (base64 encoded)
QUAY_TOKEN=$(oc get secret tssc-quay-integration -n tssc -o jsonpath='{.data.token}' | base64 -d)

# Extract organization from .dockerconfigjson
# Username format is: organization+robot_account (e.g., tssc+tssc_rw)
# We extract the part before the '+' to get the organization (e.g., tssc)
DOCKER_CONFIG=$(oc get secret tssc-quay-integration -n tssc -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d)
QUAY_USERNAME=$(echo "$DOCKER_CONFIG" | jq -r '.auths | to_entries[0].value.username')
echo "Docker config username: $QUAY_USERNAME"

# Extract organization (part before '+')
QUAY_ORG=$(echo "$QUAY_USERNAME" | cut -d'+' -f1)
echo "Extracted organization: $QUAY_ORG (should be 'tssc')"

# Verify organization is correct
if [ "$QUAY_ORG" != "tssc" ]; then
  echo "WARNING: Expected organization 'tssc' but got '$QUAY_ORG'"
  echo "This may cause image-controller to fail"
fi

# Create the secret with ONLY quaytoken and organization fields
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: quaytoken
  namespace: image-controller
type: Opaque
stringData:
  quaytoken: "${QUAY_TOKEN}"
  organization: "${QUAY_ORG}"
EOF

# Verify the secret was created
echo "Verifying quaytoken secret..."
oc get secret quaytoken -n image-controller

# Check secret structure - should ONLY have quaytoken and organization
oc get secret quaytoken -n image-controller -o jsonpath='{.data}' | jq 'keys'
# Should show: ["organization", "quaytoken"]

# Verify the values
echo ""
echo "Verifying secret values:"
echo "Token: $(oc get secret quaytoken -n image-controller -o jsonpath='{.data.quaytoken}' | base64 -d)"
ACTUAL_ORG=$(oc get secret quaytoken -n image-controller -o jsonpath='{.data.organization}' | base64 -d)
echo "Organization: $ACTUAL_ORG"

if [ "$ACTUAL_ORG" = "tssc" ]; then
  echo "✓ Organization is correct: tssc"
else
  echo "✗ WARNING: Organization is '$ACTUAL_ORG' but should be 'tssc'"
fi

echo ""
echo "✓ image-controller namespace and quaytoken secret ready"
echo "✓ Konflux instance can now be created without image-controller pod failures"
```

**Why this is necessary:**
- The Konflux instance creates the `image-controller` namespace and immediately starts pods
- These pods require the `quaytoken` secret with ONLY two specific keys:
  - `quaytoken` - The Quay robot account token
  - `organization` - The Quay organization name (must be `tssc`)
- By pre-creating the namespace and secret, pods start successfully on first try
- This prevents `ContainerCreating` and `CreateContainerConfigError` states

**Important:** The `quaytoken` secret format is different from `tssc-quay-integration`:
- `quaytoken` secret: Contains ONLY `quaytoken` and `organization` fields (simple format)
- `tssc-quay-integration`: Contains `.dockerconfigjson`, `.dockerconfigjsonreadonly`, `token`, `url` (complex format)
- The image-controller expects the simple format with just the two required fields

---

### Step 3: Create Konflux Instance

After the operator is installed and integration secrets are configured, create a Konflux instance to deploy the Konflux controllers.

```bash
# Create konflux-ui namespace for the Konflux instance
oc create namespace konflux-ui --dry-run=client -o yaml | oc apply -f -

# Verify the Konflux CRD is available
oc get crd konfluxes.konflux.konflux-ci.dev

# View the Konflux CRD specification
oc explain konflux.spec

# Create Konflux instance
# IMPORTANT: Must be named 'konflux' - only one instance allowed per cluster
cat <<EOF | oc apply -f -
apiVersion: konflux.konflux-ci.dev/v1alpha1
kind: Konflux
metadata:
  name: konflux  # MUST be exactly 'konflux' (singleton)
  namespace: konflux-ui
spec:
  # Enable image controller for container image management
  imageController:
    enabled: true
EOF

# Note: The Konflux instance will automatically create or adopt the following namespaces:
# - build-service: For build operations
# - integration-service: For integration testing
# - image-controller: Already pre-created in Step 2.1 with quaytoken secret
# - release-service: For release operations
# - konflux-info: For Konflux information

# Wait for Konflux instance to be ready and deploy controllers
echo "Waiting for Konflux instance to deploy controllers..."
sleep 120

# Check Konflux instance status
oc get konflux konflux -o yaml

# Verify Konflux controllers are running
# Controllers may be deployed in different namespaces
oc get pods -A | grep -E "application-service|integration-service|release-service|build-service|image-controller"

# Common Konflux namespaces:
# - build-service: Build operations
# - integration-service: Integration testing
# - image-controller: Image management (pre-created)
# - release-service: Release service
# - konflux-info: Konflux information

# You should see controller pods like:
# build-service-controller-manager-*
# integration-service-controller-manager-*
# release-service-controller-manager-*
# image-controller-controller-manager-* (should be Running, not ContainerCreating)
```

**Verify all Konflux namespaces and controllers are ready:**

```bash
# List all Konflux-related namespaces
oc get namespaces | grep -E "konflux|build-service|integration-service|image-controller|release-service"

# Expected namespaces:
# - konflux-ui (Konflux instance)
# - konflux-operator (Operator)
# - build-service (Builds)
# - integration-service (Integration tests)
# - image-controller (Image management - pre-created in Step 2.1)
# - release-service (Release service)
# - konflux-info (Konflux information)

# Verify all Konflux service pods are running
echo "Checking all Konflux service namespaces..."
for ns in build-service integration-service image-controller release-service; do
  echo "Checking $ns namespace..."
  oc get pods -n $ns 2>/dev/null || echo "Namespace $ns not yet ready"
done

# Wait for image-controller pods to be ready (should start immediately with pre-created secret)
echo "Waiting for image-controller controller manager to be ready..."
oc wait --for=condition=ready pod -l app.kubernetes.io/name=image-controller \
  -n image-controller --timeout=300s || \
  echo "Warning: image-controller pods not ready. Check status with: oc get pods -n image-controller"

# Check for common startup errors
echo "Checking image-controller logs for errors..."
oc logs -n image-controller deployment/image-controller-controller-manager --tail=50 | grep -i error || echo "No errors found"

# Note: If you see "CSRF token was invalid or missing" errors, this is related to
# Quay's health probe and may not prevent the image-controller from functioning
# See troubleshooting section for details
```

**Note:** Because we pre-created the `image-controller` namespace and `quaytoken` secret in Step 2.1, the image-controller pods should start successfully. However, you may see CSRF token errors in the logs related to Quay health probes - these are typically non-blocking for normal image operations.

---

### Step 4: Verify and Create Shared Environment Namespaces

Verify or create the shared environment namespaces. This is a **one-time setup** - all applications from Developer Hub will deploy to these same namespaces.

```bash
echo "Verifying/creating shared TSSC environment namespaces..."
echo "These namespaces will host ALL applications created from Developer Hub template:"
echo "  - tssc-app-ci (CI/CD namespace - all pipelines)"
echo "  - tssc-app-development (shared development environment)"
echo "  - tssc-app-stage (shared staging environment)"
echo "  - tssc-app-prod (shared production environment)"

# Create CI/CD namespace (may already exist with integration secrets)
oc create namespace tssc-app-ci --dry-run=client -o yaml | oc apply -f -

# Create shared environment namespaces
oc create namespace tssc-app-development --dry-run=client -o yaml | oc apply -f -
oc create namespace tssc-app-stage --dry-run=client -o yaml | oc apply -f -
oc create namespace tssc-app-prod --dry-run=client -o yaml | oc apply -f -

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

### Step 5: Setup RBAC (Cross-Namespace Permissions)

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

# Create ClusterRole for viewing Konflux resources
cat <<EOF | oc apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: konflux-viewer
rules:
- apiGroups: ["konflux.konflux-ci.dev"]
  resources: ["konfluxes"]
  verbs: ["get", "list", "watch"]
EOF

# Grant service account permission to view Konflux instances cluster-wide
oc create clusterrolebinding konflux-viewer-binding \
  --serviceaccount=tssc-app-ci:release-service-account \
  --clusterrole=konflux-viewer

# Create role for secret read-write in pipelines namespace
cat <<EOF | oc apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-rw
  namespace: openshift-pipelines
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
EOF

# Grant service account secret access in pipelines namespace
oc create rolebinding secret-rw-binding \
  --serviceaccount=tssc-app-ci:release-service-account \
  --role=secret-rw \
  -n openshift-pipelines

# Copy GitLab integration secret from tssc to build-service and integration-service namespaces
# This enables Pipelines as Code to work with self-hosted GitLab
oc get secret tssc-gitlab-integration -n tssc -o yaml | \
  sed 's/namespace: tssc/namespace: build-service/' | \
  oc apply -f -

oc get secret tssc-gitlab-integration -n tssc -o yaml | \
  sed 's/namespace: tssc/namespace: integration-service/' | \
  oc apply -f -

# Create Pipelines as Code secret for GitLab in both namespaces
# Extract GitLab details from tssc-gitlab-integration secret
GITLAB_HOST=$(oc get secret tssc-gitlab-integration -n tssc -o jsonpath='{.data.host}' | base64 -d)
GITLAB_PORT=$(oc get secret tssc-gitlab-integration -n tssc -o jsonpath='{.data.port}' | base64 -d)
GITLAB_TOKEN=$(oc get secret tssc-gitlab-integration -n tssc -o jsonpath='{.data.token}' | base64 -d)

# Construct GitLab URL (https://host:port)
GITLAB_URL="https://${GITLAB_HOST}:${GITLAB_PORT}"

# Note: Webhook secret will be configured per repository
# For now, create a generic webhook secret or leave empty
WEBHOOK_SECRET=$(openssl rand -hex 20)

for ns in build-service integration-service; do
  oc create secret generic pipelines-as-code-secret \
    --from-literal=provider.url="${GITLAB_URL}" \
    --from-literal=provider.token="${GITLAB_TOKEN}" \
    --from-literal=webhook.secret="${WEBHOOK_SECRET}" \
    -n $ns --dry-run=client -o yaml | oc apply -f -
done

# Note: quaytoken secret was already created in image-controller namespace in Step 2.1
# Verify it exists
oc get secret quaytoken -n image-controller

# Verify it has only the two required fields
oc get secret quaytoken -n image-controller -o jsonpath='{.data}' | jq 'keys'
# Should show: ["organization", "quaytoken"]

# If you have additional registry credentials, link them
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
oc get clusterrolebinding konflux-viewer-binding
oc get rolebinding secret-rw-binding -n openshift-pipelines
oc get serviceaccount release-service-account -n tssc-app-ci

# Verify secrets are copied to the correct namespaces
echo "Verifying secrets distribution..."
oc get secret pipelines-as-code-secret -n build-service
oc get secret pipelines-as-code-secret -n integration-service
oc get secret tssc-gitlab-integration -n build-service
oc get secret tssc-gitlab-integration -n integration-service
oc get secret quaytoken -n image-controller  # Was created in Step 2.1 with only quaytoken and organization fields
```

**Note about quaytoken secret:**
The `quaytoken` secret in `image-controller` namespace contains only two fields:
- `quaytoken` - The Quay robot account token
- `organization` - The Quay organization (should be `tssc`)

This is different from the `tssc-quay-integration` secret which has multiple fields.

**Note:** This enhanced RBAC setup:
- Allows the release-service-account to deploy ANY application to the shared environments (multi-tenant model)
- Grants cluster-wide view permissions for Konflux resources
- Distributes GitLab integration secrets to build-service and integration-service namespaces:
  - Full `tssc-gitlab-integration` secret copied for component operations
  - `pipelines-as-code-secret` created with GitLab URL and token for webhook integration
- Distributes Quay integration secret to image-controller namespace:
  - Renamed to `quaytoken` for image controller operations
  - Contains Docker configs and Quay API token
- Enables automatic webhook-based builds from self-hosted GitLab

---

### Step 6: Install Pipelines and Tasks

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

### Step 7: Configure and Apply Konflux Resources

Now we'll create all the Konflux CRs. These tell Konflux about your application.

#### 7.1: Edit and Apply Application

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

#### 7.2: Edit and Apply Component

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

#### 7.3: Apply Integration Test Scenario

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

#### 7.4: Apply ReleasePlans (All 3)

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

#### 7.5: Apply ReleasePlanAdmissions (One per Environment)

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

### Step 8: Configure GitLab Webhooks and Pipeline Triggers

Now that GitLab integration is configured, you can set up automatic build triggering via webhooks.

**Step 8.1: Create Pipelines as Code Configuration in Your Application Repository**

In your **application repository** (not tssc-sample-pipelines), create the `.tekton` directory and PipelineRun configuration.

```bash
# In your application repository
cd your-app-repo

# Create .tekton directory
mkdir -p .tekton

# Create the PipelineRun configuration
cat > .tekton/push.yaml <<'EOF'
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  name: build-on-push
  namespace: tssc-app-ci
  annotations:
    # Pipelines as Code annotations for GitLab
    pipelinesascode.tekton.dev/on-event: "[push]"
    pipelinesascode.tekton.dev/on-target-branch: "[main]"
    pipelinesascode.tekton.dev/max-keep-runs: "5"
    pipelinesascode.tekton.dev/task: |
      - https://raw.githubusercontent.com/tektoncd/catalog/main/task/git-clone/0.9/git-clone.yaml
  labels:
    # CRITICAL: These labels link PipelineRun to Konflux Component
    appstudio.openshift.io/application: my-java-app
    appstudio.openshift.io/component: my-java-app-backend
    backstage.io/kubernetes-id: my-java-app
spec:
  params:
    - name: component-name
      value: my-java-app-backend
    - name: git-url
      value: "{{ repo_url }}"
    - name: output-image
      value: "quay.io/myorg/my-java-app-backend:{{ revision }}"
    - name: revision
      value: "{{ revision }}"
    - name: event-type
      value: "{{ event_type }}"

  pipelineRef:
    resolver: cluster
    params:
      - name: name
        value: maven-build-ci-konflux
      - name: namespace
        value: tssc-app-ci
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
EOF

# Commit and push the configuration
git add .tekton/push.yaml
git commit -m "Add Pipelines as Code configuration for Konflux"
git push origin main
```

**Important:** Replace the following placeholders in `.tekton/push.yaml`:
- `my-java-app` → Your application name
- `my-java-app-backend` → Your component name
- `quay.io/myorg/my-java-app-backend` → Your image repository

**Step 8.2: Configure GitLab Webhook**

Pipelines as Code needs to create a webhook in your GitLab repository. This happens automatically when you create a Repository CR.

```bash
# Set your GitLab repository details
export GITLAB_URL="https://gitlab.apps.your-cluster.com"
export GITLAB_PROJECT="myorg/my-java-app"  # Format: namespace/project
export APP_NAME="my-java-app"

# Create Pipelines as Code Repository CR
cat <<EOF | oc apply -f -
apiVersion: "pipelinesascode.tekton.dev/v1alpha1"
kind: Repository
metadata:
  name: ${APP_NAME}
  namespace: tssc-app-ci
spec:
  url: "${GITLAB_URL}/${GITLAB_PROJECT}"
  git_provider:
    type: gitlab
    url: "${GITLAB_URL}"
    secret:
      name: pipelines-as-code-secret
      key: provider.token
    webhook_secret:
      name: pipelines-as-code-secret
      key: webhook.secret
EOF

# Verify Repository CR created
oc get repository ${APP_NAME} -n tssc-app-ci

# Check Repository status
oc describe repository ${APP_NAME} -n tssc-app-ci
```

**Step 8.3: Verify Webhook in GitLab**

After creating the Repository CR, Pipelines as Code will automatically create a webhook in your GitLab project.

1. Navigate to your GitLab project
2. Go to **Settings** → **Webhooks**
3. You should see a webhook pointing to your OpenShift Pipelines as Code route
4. The webhook should be configured for **Push events** and **Merge request events**

**Step 8.4: Get Pipelines as Code Route**

```bash
# Get the Pipelines as Code webhook URL
oc get route pipelines-as-code-controller -n openshift-pipelines

# Or get full webhook URL
echo "Webhook URL: https://$(oc get route pipelines-as-code-controller -n openshift-pipelines -o jsonpath='{.spec.host}')"
```

**Note:** If the webhook wasn't created automatically, you can create it manually in GitLab:
1. Go to your GitLab project → Settings → Webhooks
2. URL: `https://<pipelines-as-code-route>/hook`
3. Secret Token: Use the value from `GITLAB_WEBHOOK_SECRET` you set earlier
4. Trigger: Enable "Push events" and "Merge request events"
5. SSL verification: Enable if using valid certificates

---

## Testing the Flow

### Test 1: Trigger a Build via GitLab Push

Now that webhooks are configured, simply push code to trigger a build automatically.

```bash
# In your application repository
cd your-app-repo

# Make a small change to trigger the build
echo "# Konflux test" >> README.md
git add README.md
git commit -m "Test Konflux flow"
git push origin main

# The webhook will automatically trigger a PipelineRun
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

Developers only need to push code. Everything else is automatic via GitLab webhooks.

```bash
# 1. Developer makes changes
cd my-java-app
vim src/main/java/com/example/MyService.java

# 2. Commit and push to GitLab
git add .
git commit -m "Add new authentication feature"
git push origin main

# That's it! GitLab webhook triggers Konflux, which handles:
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

### Issue: Build Doesn't Trigger After Git Push

**Symptoms:** Push code to GitLab but no PipelineRun created

**Solutions:**

```bash
# 1. Check Repository CR status
oc get repository -n tssc-app-ci
oc describe repository <your-app-name> -n tssc-app-ci

# 2. Verify Pipelines as Code controller is running
oc get pods -n openshift-pipelines | grep pipelines-as-code

# 3. Check Pipelines as Code controller logs
oc logs -n openshift-pipelines deployment/pipelines-as-code-controller --tail=50

# 4. Verify webhook exists in GitLab
# Go to GitLab project → Settings → Webhooks
# Verify webhook URL and secret token are correct

# 5. Test webhook manually in GitLab
# Go to GitLab project → Settings → Webhooks
# Click "Test" → "Push events"
# Check the response

# 6. Check GitLab integration secret
oc get secret pipelines-as-code-secret -n build-service
oc get secret tssc-gitlab-integration -n build-service

# 7. Verify .tekton/push.yaml exists in your repository
# Check your application repo has .tekton/push.yaml committed

# 8. Check recent events in tssc-app-ci namespace
oc get events -n tssc-app-ci --sort-by='.lastTimestamp' | tail -20
```

**Common Issues:**
- **Webhook not created**: Manually create webhook in GitLab pointing to Pipelines as Code route
- **Wrong secret**: Ensure webhook secret in GitLab matches `GITLAB_WEBHOOK_SECRET`
- **Missing .tekton/push.yaml**: Commit the Pipelines as Code configuration to your repo
- **Wrong namespace**: Ensure `.tekton/push.yaml` specifies `namespace: tssc-app-ci`
- **SSL verification failed**: If using self-signed certs, disable SSL verification in webhook settings

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

### Issue: Image Controller Pods Not Starting

**Symptoms:** `image-controller-controller-manager` stuck in `ContainerCreating` or `CreateContainerConfigError`

**Solutions:**

```bash
# 1. Check if image-controller namespace has the quaytoken secret
oc get secret quaytoken -n image-controller

# If missing, recreate from tssc-quay-integration secret (should have been created in Step 2.1)
# Get the token value (decode from base64)
QUAY_TOKEN=$(oc get secret tssc-quay-integration -n tssc -o jsonpath='{.data.token}' | base64 -d)

# Extract organization from .dockerconfigjson
# Username format: organization+robot_account (e.g., tssc+tssc_rw → tssc)
DOCKER_CONFIG=$(oc get secret tssc-quay-integration -n tssc -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d)
QUAY_USERNAME=$(echo "$DOCKER_CONFIG" | jq -r '.auths | to_entries[0].value.username')
echo "Username from Docker config: $QUAY_USERNAME"

QUAY_ORG=$(echo "$QUAY_USERNAME" | cut -d'+' -f1)
echo "Extracted organization: $QUAY_ORG (expected: tssc)"

# Create the secret with ONLY quaytoken and organization fields
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: quaytoken
  namespace: image-controller
type: Opaque
stringData:
  quaytoken: "${QUAY_TOKEN}"
  organization: "${QUAY_ORG}"
EOF

# 2. Verify the secret has ONLY quaytoken and organization keys
oc get secret quaytoken -n image-controller -o jsonpath='{.data}' | jq 'keys'
# Should show: ["organization", "quaytoken"]

# 2a. Verify the values
echo "Verifying secret values:"
echo "Quay token: $(oc get secret quaytoken -n image-controller -o jsonpath='{.data.quaytoken}' | base64 -d)"

ACTUAL_ORG=$(oc get secret quaytoken -n image-controller -o jsonpath='{.data.organization}' | base64 -d)
echo "Quay organization: $ACTUAL_ORG"

if [ "$ACTUAL_ORG" = "tssc" ]; then
  echo "✓ Organization is correct: tssc"
else
  echo "✗ WARNING: Organization is '$ACTUAL_ORG' but should be 'tssc'"
  echo "   Image-controller may fail to authenticate with Quay"
fi
echo ""

# 3. Check image-controller pods status
oc get pods -n image-controller

# 4. Check pod events for specific error
oc describe pod -n image-controller <pod-name>

# 5. Check image-controller controller manager logs
oc logs -n image-controller deployment/image-controller-controller-manager

# 6. Restart image-controller deployment if secret was missing
oc rollout restart deployment/image-controller-controller-manager -n image-controller

# 7. Watch for pods to become ready
oc get pods -n image-controller -w
```

**Common Issues:**
- **Missing quaytoken secret**: Create secret as shown above with quaytoken and organization fields
- **Incorrect secret format**: Ensure secret has ONLY two fields: quaytoken and organization
- **Wrong secret name**: Must be named `quaytoken` in image-controller namespace
- **Wrong organization value**: Must be `tssc` (extracted from username like `tssc+tssc_rw`)
- **Image pull errors**: Check if Quay registry is accessible from cluster
- **Volume mount errors**: Check if secret is properly mounted in pod spec

---

### Issue: Image Controller CSRF Token Error

**Symptoms:** `unable to register quay availability probe: CSRF token was invalid or missing`

This error occurs when image-controller tries to create a test robot account in Quay to verify connectivity.

**Solutions:**

```bash
# 1. Check image-controller logs for the full error
oc logs -n image-controller deployment/image-controller-controller-manager | grep -A 5 "CSRF"

# 2. Verify the quaytoken has correct permissions
# The token must have permissions to create robot accounts
# Log into Quay UI and check robot account permissions

# 3. Check if Quay requires CSRF tokens for API calls
# For self-hosted Quay, CSRF validation may be enabled by default

# 4. Verify the organization exists in Quay
QUAY_URL=$(oc get secret tssc-quay-integration -n tssc -o jsonpath='{.data.url}' | base64 -d)
echo "Quay URL: $QUAY_URL"
echo "Organization should be: tssc"
# Log into Quay UI and verify organization 'tssc' exists

# 5. Test Quay API access manually
QUAY_TOKEN=$(oc get secret quaytoken -n image-controller -o jsonpath='{.data.quaytoken}' | base64 -d)
QUAY_ORG=$(oc get secret quaytoken -n image-controller -o jsonpath='{.data.organization}' | base64 -d)

# Test API access (replace QUAY_HOST with your Quay hostname)
QUAY_HOST="quay-c76tb-1.apps.cluster-c76tb.dynamic.redhatworkshops.io"
curl -k -H "Authorization: Bearer ${QUAY_TOKEN}" \
  "https://${QUAY_HOST}/api/v1/organization/${QUAY_ORG}"

# 6. Check robot account permissions
# The robot account used must have "Admin" permissions on the organization
# to create other robot accounts
```

**Root Cause:**
The image-controller performs a health check by attempting to create a temporary robot account in Quay. This requires:
- Robot account token with admin permissions on the organization
- Quay API must allow robot account creation via API
- CSRF validation may interfere with API calls

**Workarounds:**

**Option 1: Disable image-controller health probe (if not critical)**
```bash
# Check if image-controller has configuration to disable the probe
oc get deployment image-controller-controller-manager -n image-controller -o yaml | grep -A 10 "env:"

# You may need to patch the deployment to disable the probe
# This is environment-specific and may require Konflux operator configuration
```

**Option 2: Ensure robot account has admin permissions**
1. Log into Quay UI
2. Navigate to Organization → `tssc`
3. Go to Robot Accounts → `tssc+tssc_rw`
4. Verify permissions include "Admin" role
5. If not, upgrade the robot account permissions

**Option 3: Check Quay CSRF settings**
For self-hosted Quay, you may need to configure CSRF token handling:
1. Check Quay configuration for `CSRF_TOKEN_ENFORCEMENT`
2. Ensure API tokens bypass CSRF validation
3. This may require Quay administrator access to modify config

**Note:** This error doesn't prevent image-controller from functioning for image management, but the health probe will continue to fail. The image-controller can still pull/push images with the provided credentials.

---

### Issue: GitLab Integration / Connectivity Issues

**Symptoms:** Webhooks not reaching OpenShift or authentication failures

**Solutions:**

```bash
# 1. Verify GitLab can reach Pipelines as Code route
oc get route pipelines-as-code-controller -n openshift-pipelines

# 2. Check if route is accessible from GitLab namespace
# If GitLab is in the same cluster, test connectivity
oc run -n gitlab test-curl --image=curlimages/curl:latest --rm -it --restart=Never \
  -- curl -k https://pipelines-as-code-controller-openshift-pipelines.apps.your-cluster.com/health

# 3. Verify GitLab secret has correct token
oc get secret tssc-gitlab-integration -n tssc -o jsonpath='{.data.token}' | base64 -d
# Test this token in GitLab (it should have api, read_repository, write_repository scopes)

# 4. Check if GitLab host and port are correct
GITLAB_HOST=$(oc get secret tssc-gitlab-integration -n tssc -o jsonpath='{.data.host}' | base64 -d)
GITLAB_PORT=$(oc get secret tssc-gitlab-integration -n tssc -o jsonpath='{.data.port}' | base64 -d)
echo "GitLab URL: https://${GITLAB_HOST}:${GITLAB_PORT}"

# 5. Verify GitLab personal access token is not expired
# Log into GitLab → User Settings → Access Tokens
# Check token expiration date

# 6. Test GitLab API access from OpenShift
oc run -n tssc-app-ci test-gitlab-api --image=curlimages/curl:latest --rm -it --restart=Never \
  -- curl -k -H "PRIVATE-TOKEN: your-gitlab-token" https://gitlab.apps.your-cluster.com/api/v4/user

# 7. Check Pipelines as Code can authenticate to GitLab
oc logs -n openshift-pipelines deployment/pipelines-as-code-controller | grep -i gitlab

# 8. Verify network policies allow traffic between namespaces
oc get networkpolicies -n gitlab
oc get networkpolicies -n openshift-pipelines
```

**Common Issues:**
- **SSL/TLS issues**: If using self-signed certificates, you may need to configure trust
- **Network policies blocking**: Ensure network policies allow traffic from openshift-pipelines to gitlab namespace
- **Token expired**: Generate a new GitLab Personal Access Token
- **Wrong GitLab URL**: Ensure the URL matches your self-hosted GitLab instance
- **Firewall/ingress**: Ensure GitLab can make outbound HTTPS connections to the Pipelines as Code route

**For self-hosted GitLab in the same cluster:**

```bash
# Allow communication between gitlab and openshift-pipelines namespaces
# This may be needed if network policies are restrictive

# Create NetworkPolicy in openshift-pipelines to allow ingress from gitlab
cat <<EOF | oc apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-gitlab
  namespace: openshift-pipelines
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: gitlab
EOF

# Verify GitLab can reach the route
# From a GitLab pod, test connectivity:
oc exec -n gitlab <gitlab-pod-name> -- curl -k https://pipelines-as-code-controller-openshift-pipelines.apps.your-cluster.com/health
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
