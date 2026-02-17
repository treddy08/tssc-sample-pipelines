# Konflux Implementation Guide - Corrections

**IMPORTANT:** The current KONFLUX-IMPLEMENTATION-GUIDE.md has two critical issues that need correction.

## Issue 1: Missing Dev Environment

### What's Wrong
The guide shows:
- ✅ Build pipeline in `dev` namespace
- ✅ Snapshot creation
- ❌ **NO deployment to dev environment!**
- ✅ Auto-release to `stage`
- ✅ Manual release to `prod`

### What's Missing
There's no dev environment deployment! In the current setup:
- Builds happen in `dev`
- But nothing gets deployed to `dev` for testing
- First deployment is to `stage`

This is wrong - you need a dev environment where you can test before promoting to stage.

### Correct Flow
```
Build → Snapshot → Integration Tests → Dev → Stage → Prod
                                        ↑      ↑       ↑
                                      Auto   Auto   Manual
```

## Issue 2: Poor Namespace Naming

### What's Wrong
The guide uses:
- `dev` - Too generic
- `stage` - Too generic
- `prod` - Too generic

Problems:
- Conflicts with other apps in same cluster
- Not clear which app they belong to
- Doesn't follow Konflux best practices

### Better Naming

**Pattern: `<app-name>-<environment>`**

For an app called "my-java-app":
- `my-java-app-dev` (or `myjavaapp-dev`)
- `my-java-app-stage` (or `myjavaapp-stage`)
- `my-java-app-prod` (or `myjavaapp-prod`)

For multi-tenant scenarios:
- `tenant-team1-dev`
- `tenant-team1-stage`
- `tenant-team1-prod`

---

## Corrected Architecture

### Namespace Structure

```
┌─────────────────────────────────────────────────────────────┐
│ <app-name>-dev                                               │
│                                                               │
│ - Application CR                                             │
│ - Component CR                                               │
│ - Build Pipeline (maven-build-ci-konflux)                   │
│ - Snapshot (created after build)                            │
│ - IntegrationTestScenario                                    │
│ - ReleasePlan → dev (auto-release: true)                    │
│ - ReleasePlan → stage (auto-release: true)                  │
│ - ReleasePlan → prod (auto-release: false)                  │
│                                                               │
│ ALSO:                                                         │
│ - Deployment resources (Deployment, Service, Route)         │
│ - Dev environment runs here                                  │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ <app-name>-stage                                             │
│                                                               │
│ - ReleasePlanAdmission (accepts from dev)                   │
│ - Release Pipeline                                           │
│ - Deployment resources                                       │
│ - Stage environment runs here                                │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ <app-name>-prod                                              │
│                                                               │
│ - ReleasePlanAdmission (accepts from dev)                   │
│ - Release Pipeline                                           │
│ - Deployment resources                                       │
│ - Production environment runs here                           │
└─────────────────────────────────────────────────────────────┘
```

### Complete Flow

```
Developer: git push origin main
   ↓
1. Build Pipeline runs in <app-name>-dev
   - Clone source
   - Build Maven project
   - Build container image
   - Push to registry with SHA tag
   - Run ACS scans
   ↓
2. Konflux creates Snapshot automatically
   - Snapshot contains: image@sha256:abc123
   ↓
3. Integration Tests run (in <app-name>-dev)
   - Tests run against the snapshot
   - Mark snapshot as passed/failed
   ↓
4. Auto-Release to Dev (if tests pass)
   - ReleasePlan: release-to-dev
   - autoRelease: true
   - Target: <app-name>-dev
   - Release pipeline runs:
     * Verify Enterprise Contract
     * Copy image to "dev" tag
     * Update GitOps for dev
   - Application deploys to <app-name>-dev namespace
   ↓
5. Auto-Release to Stage (if dev is stable)
   - ReleasePlan: release-to-stage
   - autoRelease: true
   - Target: <app-name>-stage
   - Release pipeline runs in <app-name>-stage:
     * Verify Enterprise Contract
     * Copy image to "stage" tag
     * Update GitOps for stage
   - Application deploys to <app-name>-stage namespace
   ↓
6. Manual Release to Prod (when approved)
   - Ops creates Release CR manually
   - ReleasePlan: release-to-prod
   - autoRelease: false
   - Target: <app-name>-prod
   - Release pipeline runs in <app-name>-prod:
     * Verify Enterprise Contract (stricter policy)
     * Copy image to "prod" tag
     * Update GitOps for prod
   - Application deploys to <app-name>-prod namespace
```

---

## Corrected ReleasePlans

You need **THREE** ReleasePlans in `<app-name>-dev`:

### 1. ReleasePlan for Dev

```yaml
apiVersion: appstudio.redhat.com/v1alpha1
kind: ReleasePlan
metadata:
  name: release-to-dev
  namespace: my-java-app-dev
  labels:
    release.appstudio.openshift.io/auto-release: "true"
spec:
  application: my-java-app
  target: my-java-app-dev  # Same namespace!

  pipelineRef:
    name: release-pipeline
    namespace: my-java-app-dev
```

**Note:** Target is the SAME namespace! This deploys to dev after build.

### 2. ReleasePlan for Stage

```yaml
apiVersion: appstudio.redhat.com/v1alpha1
kind: ReleasePlan
metadata:
  name: release-to-stage
  namespace: my-java-app-dev
  labels:
    release.appstudio.openshift.io/auto-release: "true"
spec:
  application: my-java-app
  target: my-java-app-stage

  pipelineRef:
    name: release-pipeline
    namespace: my-java-app-stage
```

### 3. ReleasePlan for Prod

```yaml
apiVersion: appstudio.redhat.com/v1alpha1
kind: ReleasePlan
metadata:
  name: release-to-prod
  namespace: my-java-app-dev
  labels:
    release.appstudio.openshift.io/auto-release: "false"
spec:
  application: my-java-app
  target: my-java-app-prod

  pipelineRef:
    name: release-pipeline
    namespace: my-java-app-prod
```

---

## Corrected ReleasePlanAdmissions

### Dev (in <app-name>-dev)

```yaml
apiVersion: appstudio.redhat.com/v1alpha1
kind: ReleasePlanAdmission
metadata:
  name: accept-dev-releases
  namespace: my-java-app-dev
spec:
  origin: my-java-app-dev
  applications:
    - my-java-app
  environment: dev

  pipelineRef:
    name: release-pipeline
    namespace: my-java-app-dev

  serviceAccount: release-service-account
```

### Stage (in <app-name>-stage)

```yaml
apiVersion: appstudio.redhat.com/v1alpha1
kind: ReleasePlanAdmission
metadata:
  name: accept-dev-releases
  namespace: my-java-app-stage
spec:
  origin: my-java-app-dev
  applications:
    - my-java-app
  environment: stage

  pipelineRef:
    name: release-pipeline
    namespace: my-java-app-stage

  serviceAccount: release-service-account
```

### Prod (in <app-name>-prod)

```yaml
apiVersion: appstudio.redhat.com/v1alpha1
kind: ReleasePlanAdmission
metadata:
  name: accept-dev-releases
  namespace: my-java-app-prod
spec:
  origin: my-java-app-dev
  applications:
    - my-java-app
  environment: prod

  pipelineRef:
    name: release-pipeline
    namespace: my-java-app-prod

  serviceAccount: release-service-account
```

---

## How Auto-Progression Works

### Option A: Separate Auto-Releases (Recommended)

Each environment has its own auto-release setting:

```
Snapshot created
  ↓
Tests pass
  ↓
Auto-Release to Dev (ReleasePlan with autoRelease: true)
  ↓
Dev deployment succeeds
  ↓
Auto-Release to Stage (ReleasePlan with autoRelease: true)
  ↓
Stage deployment succeeds
  ↓
Wait for manual prod release
```

### Option B: Chain Releases

Only auto-release to dev, then manually promote through environments:

```
Snapshot created
  ↓
Tests pass
  ↓
Auto-Release to Dev (ReleasePlan with autoRelease: true)
  ↓
Dev validated (manual or automated checks)
  ↓
Create Release to Stage (manual)
  ↓
Stage validated
  ↓
Create Release to Prod (manual)
```

**Recommendation:** Use Option A for dev and stage, manual for prod.

---

## Developer Workflow (Corrected)

```bash
# Developer makes changes
cd my-java-app
git add .
git commit -m "Add new feature"
git push origin main

# What happens automatically:
# 1. Build pipeline runs in my-java-app-dev
# 2. Snapshot created: my-java-app-snapshot-abc123
# 3. Integration tests run
# 4. If tests pass:
#    - Release to dev created automatically
#    - Image tagged as "dev"
#    - Deployed to my-java-app-dev namespace
# 5. If dev stable (or immediate):
#    - Release to stage created automatically
#    - Image tagged as "stage"
#    - Deployed to my-java-app-stage namespace

# Developer can test in dev:
oc get pods -n my-java-app-dev

# Developer can test in stage:
oc get pods -n my-java-app-stage
```

---

## Operations Workflow (Corrected)

### Promote to Production

```bash
# 1. List validated snapshots
oc get snapshots -n my-java-app-dev --sort-by=.metadata.creationTimestamp

# 2. Check which ones are deployed to stage
oc get releases -n my-java-app-dev | grep stage

# 3. Pick a snapshot that's validated in stage
SNAPSHOT="my-java-app-snapshot-abc123"

# 4. Create production release
cat <<EOF | oc apply -f -
apiVersion: appstudio.redhat.com/v1alpha1
kind: Release
metadata:
  name: prod-release-$(date +%Y%m%d-%H%M)
  namespace: my-java-app-dev
  annotations:
    release.appstudio.io/approved-by: "ops@example.com"
    release.appstudio.io/change-ticket: "CHG-12345"
    release.appstudio.io/validated-in: "stage"
spec:
  snapshot: ${SNAPSHOT}
  releasePlan: release-to-prod
EOF

# 5. Monitor
oc get release -n my-java-app-dev -w
oc get pipelineruns -n my-java-app-prod -w
oc get pods -n my-java-app-prod
```

---

## Summary of Changes Needed

### In the Guide

1. **Replace all namespace references:**
   - `dev` → `my-java-app-dev` (or your app name)
   - `stage` → `my-java-app-stage`
   - `prod` → `my-java-app-prod`

2. **Add dev environment setup:**
   - Step 6a: Create ReleasePlan for dev (autoRelease: true)
   - Step 6b: Create ReleasePlanAdmission for dev
   - Step 6c: Setup dev GitOps

3. **Update the flow diagram:**
   - Show three environments (dev, stage, prod)
   - Show dev auto-deployment after build
   - Show stage auto-promotion after dev
   - Show prod manual promotion

4. **Update all examples:**
   - Use proper namespace names throughout
   - Show three ReleasePlans
   - Show three ReleasePlanAdmissions

---

## Quick Reference

### Correct Namespace Names

```bash
# Create namespaces
APP_NAME="my-java-app"
oc create namespace ${APP_NAME}-dev
oc create namespace ${APP_NAME}-stage
oc create namespace ${APP_NAME}-prod
```

### Correct Flow

```
git push
  ↓
Build in ${APP_NAME}-dev
  ↓
Snapshot
  ↓
Integration Tests
  ↓
Auto-Deploy to ${APP_NAME}-dev (tag: dev)
  ↓
Auto-Promote to ${APP_NAME}-stage (tag: stage)
  ↓
Manual Promote to ${APP_NAME}-prod (tag: prod)
```

### Where Things Live

| Resource | Namespace |
|----------|-----------|
| Application | `${APP_NAME}-dev` |
| Component | `${APP_NAME}-dev` |
| Build Pipeline | `${APP_NAME}-dev` |
| Snapshot | `${APP_NAME}-dev` |
| IntegrationTestScenario | `${APP_NAME}-dev` |
| ReleasePlan (all 3) | `${APP_NAME}-dev` |
| ReleasePlanAdmission (dev) | `${APP_NAME}-dev` |
| ReleasePlanAdmission (stage) | `${APP_NAME}-stage` |
| ReleasePlanAdmission (prod) | `${APP_NAME}-prod` |
| Release Pipeline (dev) | `${APP_NAME}-dev` |
| Release Pipeline (stage) | `${APP_NAME}-stage` |
| Release Pipeline (prod) | `${APP_NAME}-prod` |
| Dev Deployment | `${APP_NAME}-dev` |
| Stage Deployment | `${APP_NAME}-stage` |
| Prod Deployment | `${APP_NAME}-prod` |

---

## Next Steps

The main KONFLUX-IMPLEMENTATION-GUIDE.md needs to be regenerated with these corrections. Would you like me to:

1. Create a completely new corrected guide?
2. Create a patch/diff showing what needs to change?
3. Create separate guides for each environment setup?

The key takeaway: **You need THREE environments (dev, stage, prod), not two!**
