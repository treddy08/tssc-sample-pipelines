# Konflux Setup Guide

This guide walks you through setting up Konflux for your application step-by-step.

## Prerequisites

Before you begin, make sure you have:

- ✅ OpenShift 4.12+ cluster with admin access
- ✅ `oc` CLI installed and logged in
- ✅ Konflux operator installed (see step 1 below)
- ✅ Application name chosen (e.g., `my-java-app`)
- ✅ Git repository URL
- ✅ Container image registry (e.g., quay.io)

---

## Step 1: Install Konflux Operator

**Skip this if Konflux is already installed on your cluster.**

```bash
# Create Konflux system namespace
oc create namespace konflux-system

# Install Konflux operator
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: konflux-catalog
  namespace: openshift-marketplace
spec:
  sourceType: grpc
  image: quay.io/konflux-ci/konflux-catalog:latest
  displayName: Konflux Catalog
  publisher: Red Hat
  updateStrategy:
    registryPoll:
      interval: 10m
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: konflux-operator-group
  namespace: konflux-system
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: konflux-operator
  namespace: konflux-system
spec:
  channel: alpha
  name: konflux-operator
  source: konflux-catalog
  sourceNamespace: openshift-marketplace
  installPlanApproval: Automatic
EOF

# Wait for operator to be ready (takes 2-3 minutes)
oc wait --for=condition=Ready pod \
  -l app=konflux-operator \
  -n konflux-system \
  --timeout=300s

# Verify CRDs installed
oc get crd | grep appstudio
```

You should see:
```
applications.appstudio.redhat.com
components.appstudio.redhat.com
snapshots.appstudio.redhat.com
integrationtestscenarios.appstudio.redhat.com
releaseplans.appstudio.redhat.com
releaseplanadmissions.appstudio.redhat.com
releases.appstudio.redhat.com
```

---

## Step 2: Create Namespaces

Set your application name and create namespaces:

```bash
# CHANGE THIS to your application name
export APP_NAME="my-java-app"

# CI namespace should already exist
# If not, create it:
oc create namespace tssc-app-ci

# Create runtime namespaces
oc create namespace ${APP_NAME}-dev
oc create namespace ${APP_NAME}-stage
oc create namespace ${APP_NAME}-prod

# Label namespaces
oc label namespace tssc-app-ci environment=ci
oc label namespace ${APP_NAME}-dev environment=dev
oc label namespace ${APP_NAME}-stage environment=stage
oc label namespace ${APP_NAME}-prod environment=prod

# Verify
oc get namespaces | grep -E "tssc-app-ci|${APP_NAME}"
```

---

## Step 3: Setup RBAC (Cross-Namespace Permissions)

The release pipeline runs in `tssc-app-ci` but deploys to runtime namespaces.

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

# Link registry credentials (if you have them)
# oc secrets link release-service-account quay-credentials -n tssc-app-ci

# Verify
oc get rolebindings -n ${APP_NAME}-dev | grep release
oc get rolebindings -n ${APP_NAME}-stage | grep release
oc get rolebindings -n ${APP_NAME}-prod | grep release
```

---

## Step 4: Install Pipelines and Tasks

```bash
# Go to tssc-sample-pipelines directory
cd /path/to/tssc-sample-pipelines

# Install all tasks in tssc-app-ci
oc apply -f tasks/ -n tssc-app-ci

# Install Konflux pipelines in tssc-app-ci
oc apply -f pipelines/maven-build-ci-konflux.yaml -n tssc-app-ci
oc apply -f pipelines/release-pipeline.yaml -n tssc-app-ci
oc apply -f pipelines/integration-tests.yaml -n tssc-app-ci

# Verify
oc get tasks -n tssc-app-ci
oc get pipelines -n tssc-app-ci
```

---

## Step 5: Edit and Apply Konflux Resources

### 5.1: Edit Application

```bash
# Edit the application file
vim konflux-resources/application.yaml
```

Change these values:
```yaml
metadata:
  name: my-java-app  # ← YOUR APP NAME
spec:
  displayName: "My Java Application"  # ← YOUR DISPLAY NAME
```

Apply:
```bash
oc apply -f konflux-resources/application.yaml -n tssc-app-ci
oc get application ${APP_NAME} -n tssc-app-ci
```

### 5.2: Edit Component

```bash
# Edit the component file
vim konflux-resources/component.yaml
```

Change these values:
```yaml
metadata:
  name: my-java-app-backend  # ← YOUR COMPONENT NAME
spec:
  application: my-java-app  # ← MUST MATCH application.yaml
  source:
    git:
      url: https://gitlab.example.com/myorg/my-java-app  # ← YOUR GIT REPO
  build:
    containerImage: quay.io/myorg/my-java-app-backend  # ← YOUR IMAGE REPO
```

Apply:
```bash
oc apply -f konflux-resources/component.yaml -n tssc-app-ci
oc get component ${APP_NAME}-backend -n tssc-app-ci
```

### 5.3: Apply Integration Test Scenario

```bash
# Edit if needed
vim konflux-resources/integration-test-scenario.yaml

# Apply
oc apply -f konflux-resources/integration-test-scenario.yaml -n tssc-app-ci
oc get integrationtestscenario integration-tests -n tssc-app-ci
```

### 5.4: Edit and Apply ReleasePlans

```bash
# Edit the release plans
vim konflux-resources/release-plans.yaml
```

Change the `target` fields to match your app name:
```yaml
target: my-java-app-dev    # ← YOUR APP NAME + "-dev"
target: my-java-app-stage  # ← YOUR APP NAME + "-stage"
target: my-java-app-prod   # ← YOUR APP NAME + "-prod"
```

Apply:
```bash
oc apply -f konflux-resources/release-plans.yaml -n tssc-app-ci

# Verify all three
oc get releaseplans -n tssc-app-ci
```

### 5.5: Edit and Apply ReleasePlanAdmissions

```bash
# Edit dev admission
vim konflux-resources/release-plan-admission-dev.yaml
```

Change:
```yaml
metadata:
  namespace: my-java-app-dev  # ← YOUR APP NAME + "-dev"
spec:
  applications:
    - my-java-app  # ← YOUR APP NAME
```

Apply to dev namespace:
```bash
oc apply -f konflux-resources/release-plan-admission-dev.yaml -n ${APP_NAME}-dev
```

Repeat for stage and prod:
```bash
# Edit and apply stage
vim konflux-resources/release-plan-admission-stage.yaml
oc apply -f konflux-resources/release-plan-admission-stage.yaml -n ${APP_NAME}-stage

# Edit and apply prod
vim konflux-resources/release-plan-admission-prod.yaml
oc apply -f konflux-resources/release-plan-admission-prod.yaml -n ${APP_NAME}-prod

# Verify
oc get releaseplanadmission -n ${APP_NAME}-dev
oc get releaseplanadmission -n ${APP_NAME}-stage
oc get releaseplanadmission -n ${APP_NAME}-prod
```

---

## Step 6: Update Your Git Repository Trigger

In your application repository, update `.tekton/on-push.yaml`:

```yaml
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  generateName: my-java-app-build-
  namespace: tssc-app-ci  # ← Pipeline runs in CI namespace
  annotations:
    pipelinesascode.tekton.dev/on-cel-expression: |
      body.object_kind == "push" && body.ref == "refs/heads/main"
  labels:
    # CRITICAL: Konflux uses these labels
    appstudio.openshift.io/application: my-java-app  # ← YOUR APP NAME
    appstudio.openshift.io/component: my-java-app-backend  # ← YOUR COMPONENT NAME
    backstage.io/kubernetes-id: my-java-app
spec:
  params:
    - name: component-name
      value: my-java-app-backend  # ← YOUR COMPONENT NAME
    - name: git-url
      value: '{{repo_url}}'
    - name: output-image
      value: "quay.io/myorg/my-java-app-backend:{{revision}}"  # ← YOUR IMAGE REPO:{{revision}}
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

Commit and push to your repo.

---

## Step 7: Test the Complete Flow

### Trigger a Build

```bash
# Make a change and push
cd your-app-repo
echo "# test" >> README.md
git add .
git commit -m "Test Konflux flow"
git push origin main
```

### Watch the Build

```bash
# Watch build pipeline
oc get pipelineruns -n tssc-app-ci -w

# You should see:
# my-java-app-build-xxxxx  (Running → Succeeded)
```

### Watch Snapshot Creation

```bash
# Konflux creates Snapshot automatically
oc get snapshots -n tssc-app-ci -w

# You should see:
# my-java-app-snapshot-xxxxx
```

### Watch Integration Tests

```bash
# Integration tests run automatically
oc get pipelineruns -n tssc-app-ci | grep integration-tests
```

### Watch Auto-Release to Dev

```bash
# After tests pass, auto-release to dev
oc get releases -n tssc-app-ci -w

# You should see:
# my-java-app-snapshot-xxxxx-dev-auto  (Running → Succeeded)
```

### Watch Auto-Release to Stage

```bash
# After dev succeeds, auto-release to stage
oc get releases -n tssc-app-ci | grep stage
```

### Verify Deployments

```bash
# Check dev deployment
oc get pods -n ${APP_NAME}-dev

# Check stage deployment
oc get pods -n ${APP_NAME}-stage
```

---

## Step 8: Manual Release to Production

When ready to promote to production:

```bash
# List validated snapshots
oc get snapshots -n tssc-app-ci

# Choose a snapshot that's in stage
SNAPSHOT_NAME="my-java-app-snapshot-xxxxx"

# Create manual release to prod
cat <<EOF | oc apply -f -
apiVersion: appstudio.redhat.com/v1alpha1
kind: Release
metadata:
  name: prod-release-$(date +%Y%m%d-%H%M)
  namespace: tssc-app-ci
  annotations:
    release.appstudio.io/approved-by: "your.name@example.com"
    release.appstudio.io/jira-ticket: "PROD-12345"
spec:
  snapshot: ${SNAPSHOT_NAME}
  releasePlan: release-to-prod
EOF

# Watch release
oc get releases -n tssc-app-ci -w

# Verify prod deployment
oc get pods -n ${APP_NAME}-prod
```

---

## Troubleshooting

### Build doesn't trigger
- Check PipelineRun labels match Component
- Verify namespace is tssc-app-ci
- Check git webhook is configured

### Snapshot not created
- Check build pipeline has IMAGE_URL and IMAGE_DIGEST results
- Verify Component exists with matching labels
- Check Konflux operator logs

### Integration tests don't run
- Verify IntegrationTestScenario exists
- Check it references correct application
- Review integration test pipeline logs

### Release fails
- Check ReleasePlanAdmission exists in target namespace
- Verify origin matches (tssc-app-ci)
- Review Enterprise Contract policy errors
- Check RBAC permissions

### Enterprise Contract fails
- Review policy violations in pipeline logs
- Check CVE scan results
- Verify image signatures
- Review SBOM content

---

## Next Steps

1. ✅ Customize integration tests in `pipelines/integration-tests.yaml`
2. ✅ Configure GitOps repository for deployments
3. ✅ Set up monitoring and alerts
4. ✅ Configure Enterprise Contract policies per environment
5. ✅ Add more components if you have a multi-component app

---

## Resources

- Konflux Documentation: https://konflux-ci.dev/
- Enterprise Contract: https://enterprisecontract.dev/
- SLSA Framework: https://slsa.dev/
- Tekton Pipelines: https://tekton.dev/

---

## Summary

You now have:
- ✅ Centralized CI/CD in tssc-app-ci namespace
- ✅ Three runtime environments (dev, stage, prod)
- ✅ Automatic promotions (dev, stage)
- ✅ Manual production releases
- ✅ Integration testing
- ✅ Policy enforcement (Enterprise Contract)
- ✅ Image signing and verification
- ✅ Complete audit trail

Congratulations! Your Konflux setup is complete.
