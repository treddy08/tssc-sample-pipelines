# Konflux Implementation Guide - Corrections v2

**CRITICAL UPDATE:** All pipelines run in a centralized CI namespace!

## Issue #3: Incorrect Pipeline Namespace Placement

### What I Showed (WRONG)

```
my-java-app-dev namespace:
  - Build pipeline runs here ❌
  - Release pipeline runs here ❌
  - Deployments here

my-java-app-stage namespace:
  - Release pipeline runs here ❌
  - Deployments here

my-java-app-prod namespace:
  - Release pipeline runs here ❌
  - Deployments here
```

### What's CORRECT

**All pipelines run in `tssc-app-ci` namespace!**

```
tssc-app-ci namespace:
  - ALL build pipelines run here ✓
  - ALL release pipelines run here ✓
  - Application CRs
  - Component CRs
  - Snapshots
  - ReleasePlans
  - IntegrationTestScenarios
  - Build tasks
  - Release tasks

my-java-app-dev namespace:
  - Deployments ONLY
  - ReleasePlanAdmission
  - Runtime resources (Services, Routes, ConfigMaps)

my-java-app-stage namespace:
  - Deployments ONLY
  - ReleasePlanAdmission
  - Runtime resources

my-java-app-prod namespace:
  - Deployments ONLY
  - ReleasePlanAdmission
  - Runtime resources
```

This is a **centralized CI/CD pattern** where:
- CI/CD infrastructure is isolated in one namespace
- Application runtime is in separate namespaces
- Better security and resource management

---

## Corrected Architecture (v2)

### Namespace Structure

```
┌─────────────────────────────────────────────────────────────────┐
│ tssc-app-ci (CI/CD Namespace)                                    │
│                                                                   │
│ Build Infrastructure:                                            │
│  • Application CR (my-java-app)                                  │
│  • Component CR (my-java-app-backend)                           │
│  • Build Pipeline (maven-build-ci-konflux)                      │
│  • All build tasks                                               │
│                                                                   │
│ Konflux Resources:                                              │
│  • Snapshot (created after build)                               │
│  • IntegrationTestScenario                                       │
│                                                                   │
│ Release Infrastructure:                                          │
│  • ReleasePlan → my-java-app-dev                                │
│  • ReleasePlan → my-java-app-stage                              │
│  • ReleasePlan → my-java-app-prod                               │
│  • Release Pipeline (release-pipeline)                          │
│  • All release tasks                                             │
│                                                                   │
│ Permissions:                                                     │
│  • Service accounts can deploy to dev/stage/prod namespaces     │
│  • Cross-namespace RBAC configured                              │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ my-java-app-dev (Runtime Namespace)                             │
│                                                                   │
│  • ReleasePlanAdmission (accepts from tssc-app-ci)              │
│  • Deployment                                                    │
│  • Service                                                       │
│  • Route                                                         │
│  • ConfigMaps, Secrets                                          │
│                                                                   │
│  NO pipelines run here!                                         │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ my-java-app-stage (Runtime Namespace)                           │
│                                                                   │
│  • ReleasePlanAdmission (accepts from tssc-app-ci)              │
│  • Deployment                                                    │
│  • Service                                                       │
│  • Route                                                         │
│  • ConfigMaps, Secrets                                          │
│                                                                   │
│  NO pipelines run here!                                         │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ my-java-app-prod (Runtime Namespace)                            │
│                                                                   │
│  • ReleasePlanAdmission (accepts from tssc-app-ci)              │
│  • Deployment                                                    │
│  • Service                                                       │
│  • Route                                                         │
│  • ConfigMaps, Secrets                                          │
│                                                                   │
│  NO pipelines run here!                                         │
└─────────────────────────────────────────────────────────────────┘
```

---

## Key Changes

### 1. All Pipelines in tssc-app-ci

**Build Pipeline:**
```yaml
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: maven-build-ci-konflux
  namespace: tssc-app-ci  # ← All pipelines here!
```

**Release Pipeline:**
```yaml
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: release-pipeline
  namespace: tssc-app-ci  # ← All pipelines here!
```

### 2. ReleasePlans Reference tssc-app-ci Pipeline

```yaml
apiVersion: appstudio.redhat.com/v1alpha1
kind: ReleasePlan
metadata:
  name: release-to-dev
  namespace: tssc-app-ci  # ← ReleasePlans live here
spec:
  application: my-java-app
  target: my-java-app-dev  # ← Target runtime namespace

  # Pipeline runs in tssc-app-ci
  pipelineRef:
    name: release-pipeline
    namespace: tssc-app-ci  # ← Same namespace!
```

### 3. ReleasePlanAdmissions Reference External Origin

```yaml
apiVersion: appstudio.redhat.com/v1alpha1
kind: ReleasePlanAdmission
metadata:
  name: accept-releases
  namespace: my-java-app-dev  # ← In runtime namespace
spec:
  origin: tssc-app-ci  # ← Accept from CI namespace
  applications:
    - my-java-app
  environment: dev

  # Pipeline runs in tssc-app-ci (NOT here!)
  pipelineRef:
    name: release-pipeline
    namespace: tssc-app-ci  # ← Cross-namespace reference
```

---

## Complete Resource Placement

### tssc-app-ci Namespace

| Resource Type | Count | Names |
|--------------|-------|-------|
| Application | 1 per app | `my-java-app` |
| Component | 1 per component | `my-java-app-backend` |
| Build Pipeline | 1 per app | `maven-build-ci-konflux` |
| Release Pipeline | 1 (shared) | `release-pipeline` |
| Build Tasks | Multiple | All tasks from tssc-sample-pipelines |
| Release Tasks | Multiple | skopeo-copy, update-deployment, etc. |
| IntegrationTestScenario | 1 per app | `integration-tests` |
| Snapshot | Many | Auto-created per build |
| ReleasePlan | 3 per app | `release-to-dev`, `release-to-stage`, `release-to-prod` |
| Release | Many | Created per promotion |

### my-java-app-dev Namespace

| Resource Type | Count | Notes |
|--------------|-------|-------|
| ReleasePlanAdmission | 1 | Accepts from tssc-app-ci |
| Deployment | 1 | Runtime deployment |
| Service | 1+ | Application services |
| Route | 1+ | External access |
| ConfigMap | Multiple | App configuration |
| Secret | Multiple | App secrets |

### my-java-app-stage Namespace

| Resource Type | Count | Notes |
|--------------|-------|-------|
| ReleasePlanAdmission | 1 | Accepts from tssc-app-ci |
| Deployment | 1 | Runtime deployment |
| Service | 1+ | Application services |
| Route | 1+ | External access |
| ConfigMap | Multiple | App configuration |
| Secret | Multiple | App secrets |

### my-java-app-prod Namespace

| Resource Type | Count | Notes |
|--------------|-------|-------|
| ReleasePlanAdmission | 1 | Accepts from tssc-app-ci |
| Deployment | 1 | Runtime deployment |
| Service | 1+ | Application services |
| Route | 1+ | External access |
| ConfigMap | Multiple | App configuration |
| Secret | Multiple | App secrets |

---

## RBAC Configuration

The release pipeline in `tssc-app-ci` needs permissions to deploy to target namespaces.

### Service Account in tssc-app-ci

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: release-service-account
  namespace: tssc-app-ci
```

### RoleBindings in Target Namespaces

#### Dev Namespace
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: release-deployer
  namespace: my-java-app-dev
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: edit
subjects:
- kind: ServiceAccount
  name: release-service-account
  namespace: tssc-app-ci  # ← Cross-namespace reference
```

#### Stage Namespace
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: release-deployer
  namespace: my-java-app-stage
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: edit
subjects:
- kind: ServiceAccount
  name: release-service-account
  namespace: tssc-app-ci  # ← Cross-namespace reference
```

#### Prod Namespace
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: release-deployer
  namespace: my-java-app-prod
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: edit
subjects:
- kind: ServiceAccount
  name: release-service-account
  namespace: tssc-app-ci  # ← Cross-namespace reference
```

---

## Corrected Flow

```
Developer: git push origin main
  ↓
1. BUILD in tssc-app-ci
   maven-build-ci-konflux runs in tssc-app-ci
   Image: quay.io/myorg/myapp@sha256:abc123
   ↓
2. SNAPSHOT in tssc-app-ci
   my-java-app-snapshot-abc123
   ↓
3. INTEGRATION TESTS in tssc-app-ci
   Tests run in tssc-app-ci
   ↓
4. AUTO-RELEASE TO DEV
   ReleasePlan in tssc-app-ci triggers release
   Release pipeline runs in tssc-app-ci
   Pipeline deploys to my-java-app-dev namespace
   Image tagged: dev
   ↓
5. AUTO-RELEASE TO STAGE
   ReleasePlan in tssc-app-ci triggers release
   Release pipeline runs in tssc-app-ci
   Pipeline deploys to my-java-app-stage namespace
   Image tagged: stage
   ↓
6. MANUAL RELEASE TO PROD
   Ops creates Release CR in tssc-app-ci
   Release pipeline runs in tssc-app-ci
   Pipeline deploys to my-java-app-prod namespace
   Image tagged: prod
```

**Key Point:** All pipeline execution happens in `tssc-app-ci`, but deployments land in target namespaces.

---

## Benefits of Centralized CI Pattern

### Security
- CI/CD infrastructure isolated from runtime
- Runtime namespaces don't need pipeline execution permissions
- Easier to audit pipeline changes

### Resource Management
- Pipeline pods run in dedicated namespace
- Better resource quota management
- Shared task and pipeline definitions

### Multi-Tenancy
- Multiple apps can use same CI namespace
- Or separate CI namespaces per team
- Runtime isolation between apps

### Simplified RBAC
- Developers only need access to runtime namespaces
- Platform team manages CI namespace
- Clear separation of concerns

---

## Naming Conventions (Updated)

### Option 1: Single CI Namespace (Your Setup)

```
tssc-app-ci              ← All builds and releases
  ↓
my-java-app-dev          ← Runtime for app 1 dev
my-java-app-stage        ← Runtime for app 1 stage
my-java-app-prod         ← Runtime for app 1 prod
my-python-app-dev        ← Runtime for app 2 dev
my-python-app-stage      ← Runtime for app 2 stage
my-python-app-prod       ← Runtime for app 2 prod
```

### Option 2: Per-Team CI Namespaces

```
team-alpha-ci            ← Team Alpha's CI/CD
  ↓
team-alpha-app1-dev
team-alpha-app1-stage
team-alpha-app1-prod
team-alpha-app2-dev
team-alpha-app2-stage
team-alpha-app2-prod

team-beta-ci             ← Team Beta's CI/CD
  ↓
team-beta-app1-dev
team-beta-app1-stage
team-beta-app1-prod
```

---

## Installation Steps (Updated)

### Step 1: Create Namespaces

```bash
# CI namespace (already exists in your setup)
oc create namespace tssc-app-ci

# Runtime namespaces for your app
APP_NAME="my-java-app"
oc create namespace ${APP_NAME}-dev
oc create namespace ${APP_NAME}-stage
oc create namespace ${APP_NAME}-prod
```

### Step 2: Setup RBAC

```bash
# Create service account in CI namespace
oc create serviceaccount release-service-account -n tssc-app-ci

# Grant permissions to deploy to dev
oc create rolebinding release-deployer \
  --serviceaccount=tssc-app-ci:release-service-account \
  --clusterrole=edit \
  -n ${APP_NAME}-dev

# Grant permissions to deploy to stage
oc create rolebinding release-deployer \
  --serviceaccount=tssc-app-ci:release-service-account \
  --clusterrole=edit \
  -n ${APP_NAME}-stage

# Grant permissions to deploy to prod
oc create rolebinding release-deployer \
  --serviceaccount=tssc-app-ci:release-service-account \
  --clusterrole=edit \
  -n ${APP_NAME}-prod
```

### Step 3: Install Pipelines in tssc-app-ci

```bash
# Install build pipeline
oc apply -f pipelines/maven-build-ci-konflux.yaml -n tssc-app-ci

# Install release pipeline
oc apply -f pipelines/release-pipeline.yaml -n tssc-app-ci

# Install all tasks
oc apply -f tasks/ -n tssc-app-ci
```

### Step 4: Create Konflux Resources in tssc-app-ci

```bash
# Application
cat <<EOF | oc apply -f -
apiVersion: appstudio.redhat.com/v1alpha1
kind: Application
metadata:
  name: ${APP_NAME}
  namespace: tssc-app-ci
spec:
  displayName: "My Java Application"
EOF

# Component
cat <<EOF | oc apply -f -
apiVersion: appstudio.redhat.com/v1alpha1
kind: Component
metadata:
  name: ${APP_NAME}-backend
  namespace: tssc-app-ci
spec:
  application: ${APP_NAME}
  componentName: ${APP_NAME}-backend
  source:
    git:
      url: https://gitlab.example.com/myorg/my-java-app
      revision: main
  build:
    containerImage: quay.io/myorg/my-java-app
  buildPipelineRef:
    name: maven-build-ci-konflux
EOF

# IntegrationTestScenario
cat <<EOF | oc apply -f -
apiVersion: appstudio.redhat.com/v1beta1
kind: IntegrationTestScenario
metadata:
  name: integration-tests
  namespace: tssc-app-ci
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
EOF

# ReleasePlans (3 total)
cat <<EOF | oc apply -f -
apiVersion: appstudio.redhat.com/v1alpha1
kind: ReleasePlan
metadata:
  name: release-to-dev
  namespace: tssc-app-ci
  labels:
    release.appstudio.openshift.io/auto-release: "true"
spec:
  application: ${APP_NAME}
  target: ${APP_NAME}-dev
  pipelineRef:
    name: release-pipeline
    namespace: tssc-app-ci
---
apiVersion: appstudio.redhat.com/v1alpha1
kind: ReleasePlan
metadata:
  name: release-to-stage
  namespace: tssc-app-ci
  labels:
    release.appstudio.openshift.io/auto-release: "true"
spec:
  application: ${APP_NAME}
  target: ${APP_NAME}-stage
  pipelineRef:
    name: release-pipeline
    namespace: tssc-app-ci
---
apiVersion: appstudio.redhat.com/v1alpha1
kind: ReleasePlan
metadata:
  name: release-to-prod
  namespace: tssc-app-ci
  labels:
    release.appstudio.openshift.io/auto-release: "false"
spec:
  application: ${APP_NAME}
  target: ${APP_NAME}-prod
  pipelineRef:
    name: release-pipeline
    namespace: tssc-app-ci
EOF
```

### Step 5: Create ReleasePlanAdmissions in Runtime Namespaces

```bash
# Dev
cat <<EOF | oc apply -f -
apiVersion: appstudio.redhat.com/v1alpha1
kind: ReleasePlanAdmission
metadata:
  name: accept-releases
  namespace: ${APP_NAME}-dev
spec:
  origin: tssc-app-ci
  applications:
    - ${APP_NAME}
  environment: dev
  pipelineRef:
    name: release-pipeline
    namespace: tssc-app-ci
  serviceAccount: release-service-account
EOF

# Stage
cat <<EOF | oc apply -f -
apiVersion: appstudio.redhat.com/v1alpha1
kind: ReleasePlanAdmission
metadata:
  name: accept-releases
  namespace: ${APP_NAME}-stage
spec:
  origin: tssc-app-ci
  applications:
    - ${APP_NAME}
  environment: stage
  pipelineRef:
    name: release-pipeline
    namespace: tssc-app-ci
  serviceAccount: release-service-account
EOF

# Prod
cat <<EOF | oc apply -f -
apiVersion: appstudio.redhat.com/v1alpha1
kind: ReleasePlanAdmission
metadata:
  name: accept-releases
  namespace: ${APP_NAME}-prod
spec:
  origin: tssc-app-ci
  applications:
    - ${APP_NAME}
  environment: prod
  pipelineRef:
    name: release-pipeline
    namespace: tssc-app-ci
  serviceAccount: release-service-account
EOF
```

---

## Summary of Corrections

1. **Missing Dev Environment** - Added dev as first deployment target
2. **Poor Namespace Naming** - Use `<app-name>-<env>` pattern
3. **Centralized CI** - All pipelines run in `tssc-app-ci` ✓

### Corrected Resource Placement

| Resource | Namespace | Count |
|----------|-----------|-------|
| Application | tssc-app-ci | 1 per app |
| Component | tssc-app-ci | 1+ per app |
| Build Pipeline | tssc-app-ci | 1 per app |
| Release Pipeline | tssc-app-ci | 1 (shared) |
| IntegrationTestScenario | tssc-app-ci | 1 per app |
| Snapshot | tssc-app-ci | Many |
| ReleasePlan | tssc-app-ci | 3 per app |
| Release | tssc-app-ci | Many |
| ReleasePlanAdmission | `<app>-dev` | 1 |
| ReleasePlanAdmission | `<app>-stage` | 1 |
| ReleasePlanAdmission | `<app>-prod` | 1 |
| Deployments | `<app>-dev/stage/prod` | 1 per env |

This is the **correct enterprise pattern** for Konflux!
