# Konflux Implementation Guide

**Complete step-by-step guide to implement real Konflux with your maven-build-ci pipeline**

This guide shows you how to convert from git tag-based promotions to Konflux's Snapshot/Release model with:
- **Stage**: Automatic promotion after successful build + tests
- **Production**: Manual promotion via Release CR

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Prerequisites](#prerequisites)
3. [Step 1: Install Konflux Components](#step-1-install-konflux-components)
4. [Step 2: Modify maven-build-ci Pipeline](#step-2-modify-maven-build-ci-pipeline)
5. [Step 3: Create Application & Component](#step-3-create-application--component)
6. [Step 4: Create Integration Tests](#step-4-create-integration-tests)
7. [Step 5: Create Release Pipeline](#step-5-create-release-pipeline)
8. [Step 6: Setup Stage (Auto-Release)](#step-6-setup-stage-auto-release)
9. [Step 7: Setup Production (Manual Release)](#step-7-setup-production-manual-release)
10. [Step 8: Remove Git Tag Triggers](#step-8-remove-git-tag-triggers)
11. [Testing the Flow](#testing-the-flow)
12. [Developer Workflow](#developer-workflow)
13. [Operations Workflow](#operations-workflow)

---

## Architecture Overview

### Current Flow (Git Tags)
```
git push → Build → Deploy to Dev
git tag stage → Promote to Stage
GitLab Release → Promote to Prod
```

### New Flow (Konflux)
```
git push → Build → Snapshot → IntegrationTests → Auto-Release to Stage
                                                 → Manual Release to Prod
```

### Key Components

| Component | Purpose | Created By |
|-----------|---------|------------|
| **Application** | Groups related components | You (once) |
| **Component** | Single buildable unit | You (once) |
| **Snapshot** | Point-in-time image set | Konflux (automatic) |
| **IntegrationTestScenario** | Tests for snapshots | You (once) |
| **ReleasePlan** | Promotion strategy | You (per environment) |
| **ReleasePlanAdmission** | Acceptance criteria | You (per environment) |
| **Release** | Promotion trigger | Konflux (auto) or You (manual) |

---

## Prerequisites

- OpenShift 4.12+ cluster
- Cluster admin access
- Multiple namespaces: `dev`, `stage`, `prod`
- Existing `maven-build-ci` pipeline working

---

## Step 1: Install Konflux Components

### 1.1: Install Konflux Operator

```bash
# Create namespace for Konflux
oc create namespace konflux-system

# Install the Konflux operator
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

# Wait for operator to be ready
oc wait --for=condition=Ready pod -l app=konflux-operator -n konflux-system --timeout=300s
```

### 1.2: Install Release Service

The Release Service handles Snapshot → Release → Promotion flow.

```bash
# Release service is part of Konflux operator
# Verify it's installed
oc get deployment -n konflux-system | grep release-service

# Should see:
# release-service-controller-manager
```

### 1.3: Install Integration Service

Handles IntegrationTestScenarios.

```bash
# Integration service is part of Konflux operator
# Verify it's installed
oc get deployment -n konflux-system | grep integration-service

# Should see:
# integration-service-controller-manager
```

### 1.4: Verify CRDs Installed

```bash
# Check that Konflux CRDs exist
oc get crd | grep appstudio

# Should see:
# applications.appstudio.redhat.com
# components.appstudio.redhat.com
# snapshots.appstudio.redhat.com
# integrationtestscenarios.appstudio.redhat.com
# releaseplans.appstudio.redhat.com
# releaseplanadmissions.appstudio.redhat.com
# releases.appstudio.redhat.com
```

---

## Step 2: Modify maven-build-ci Pipeline

The build pipeline needs to be Konflux-aware but doesn't change much.

### 2.1: Current Pipeline Issues

Your current `maven-build-ci.yaml`:
- ✅ Already produces `IMAGE_URL` and `IMAGE_DIGEST` results (good!)
- ✅ Already has Tekton Chains signing (good!)
- ❌ Updates GitOps directly to dev (remove this)
- ❌ Not labeled for Konflux discovery

### 2.2: Create Modified Pipeline

```yaml
# pipelines/maven-build-ci-konflux.yaml
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: maven-build-ci-konflux
  labels:
    # Konflux discovery labels
    appstudio.openshift.io/pipeline: build
    pipelines.openshift.io/runtime: generic
    pipelines.openshift.io/strategy: docker
    pipelines.openshift.io/used-by: build-cloud
spec:
  # CRITICAL: These results are required for Konflux to create Snapshots
  results:
    - name: IMAGE_URL
      description: The built image URL
      value: $(tasks.build-container.results.IMAGE_URL)
    - name: IMAGE_DIGEST
      description: The built image digest
      value: $(tasks.build-container.results.IMAGE_DIGEST)
    - name: CHAINS-GIT_URL
      description: The source git URL
      value: $(tasks.clone-repository.results.url)
    - name: CHAINS-GIT_COMMIT
      description: The source git commit
      value: $(tasks.clone-repository.results.commit)
    # Optional but recommended
    - name: ACS_SCAN_OUTPUT
      value: $(tasks.acs-image-scan.results.SCAN_OUTPUT)

  params:
    - description: Name of the software component
      name: component-name
      type: string
    - description: Source Repository URL
      name: git-url
      type: string
    - default: ""
      description: Revision of the Source Repository
      name: revision
      type: string
    - description: Fully Qualified Output Image
      name: output-image
      type: string
    - default: .
      name: path-context
      type: string
    - default: Dockerfile
      name: dockerfile
      type: string
    - default: "false"
      name: rebuild
      type: string
    - default: ""
      name: image-expires-after
    - default: rox-api-token
      name: stackrox-secret
      type: string
    - default: push
      name: event-type
      type: string
    - default: []
      name: build-args
      type: array
    - default: ""
      name: build-args-file
      type: string
    - name: trustification-secret-name
      default: tpa-secret
      type: string
    - name: fail-if-trustification-not-configured
      default: 'false'  # Changed to false to not fail if missing
      type: string
    - name: sboms-dir
      default: sboms
      type: string

  tasks:
    - name: init
      params:
        - name: image-url
          value: $(params.output-image)
        - name: rebuild
          value: $(params.rebuild)
      taskRef:
        name: init

    - name: clone-repository
      params:
        - name: url
          value: $(params.git-url)
        - name: revision
          value: $(params.revision)
      runAfter:
        - init
      taskRef:
        name: git-clone
      when:
        - input: $(tasks.init.results.build)
          operator: in
          values:
            - "true"
      workspaces:
        - name: output
          workspace: workspace
        - name: basic-auth
          workspace: git-auth

    - name: package
      params:
        - name: SUBDIRECTORY
          value: source
      runAfter:
        - clone-repository
      taskRef:
        name: maven
      workspaces:
        - name: source
          workspace: workspace
        - name: maven-settings
          workspace: maven-settings

    - name: build-container
      params:
        - name: IMAGE
          value: $(params.output-image)
        - name: DOCKERFILE
          value: $(params.dockerfile)
        - name: CONTEXT
          value: $(params.path-context)
        - name: IMAGE_EXPIRES_AFTER
          value: $(params.image-expires-after)
        - name: COMMIT_SHA
          value: $(tasks.clone-repository.results.commit)
        - name: BUILD_ARGS
          value:
            - $(params.build-args[*])
        - name: BUILD_ARGS_FILE
          value: $(params.build-args-file)
        - name: SBOMS_DIR
          value: $(params.sboms-dir)
      runAfter:
        - package
      taskRef:
        name: buildah-rhtap
      when:
        - input: $(tasks.init.results.build)
          operator: in
          values:
            - "true"
      workspaces:
        - name: source
          workspace: workspace

    - name: upload-sboms-to-trustification
      params:
        - name: COMPONENT_NAME
          value: $(params.component-name)
        - name: SBOMS_DIR
          value: $(params.sboms-dir)
        - name: TRUSTIFICATION_SECRET_NAME
          value: $(params.trustification-secret-name)
        - name: FAIL_IF_TRUSTIFICATION_NOT_CONFIGURED
          value: $(params.fail-if-trustification-not-configured)
      runAfter:
        - build-container
      taskRef:
        name: upload-sbom-to-trustification
      workspaces:
        - name: sboms
          workspace: workspace

    - name: acs-image-check
      retries: 3
      params:
        - name: rox-secret-name
          value: $(params.stackrox-secret)
        - name: image
          value: $(params.output-image)
        - name: insecure-skip-tls-verify
          value: "true"
        - name: image-digest
          value: $(tasks.build-container.results.IMAGE_DIGEST)
      runAfter:
        - upload-sboms-to-trustification
      taskRef:
        name: acs-image-check

    - name: acs-image-scan
      retries: 3
      params:
        - name: rox-secret-name
          value: $(params.stackrox-secret)
        - name: image
          value: $(params.output-image)
        - name: insecure-skip-tls-verify
          value: "true"
        - name: image-digest
          value: $(tasks.build-container.results.IMAGE_DIGEST)
      runAfter:
        - upload-sboms-to-trustification
      taskRef:
        name: acs-image-scan

    # REMOVED: update-deployment task
    # REMOVED: acs-deploy-check task
    # These are handled by Konflux Release flow now

  finally:
    - name: show-sbom
      params:
        - name: IMAGE_URL
          value: $(tasks.build-container.results.IMAGE_URL)
      taskRef:
        name: show-sbom-rhdh

    - name: show-summary
      params:
        - name: pipelinerun-name
          value: $(context.pipelineRun.name)
        - name: git-url
          value: $(tasks.clone-repository.results.url)?rev=$(tasks.clone-repository.results.commit)
        - name: image-url
          value: $(params.output-image)
        - name: build-task-status
          value: $(tasks.build-container.status)
      taskRef:
        name: summary
      workspaces:
        - name: workspace
          workspace: workspace

  workspaces:
    - name: workspace
    - name: maven-settings
    - name: git-auth
      optional: true
```

### 2.3: Apply the Pipeline

```bash
cd /path/to/tssc-sample-pipelines

# Apply the modified pipeline
oc apply -f pipelines/maven-build-ci-konflux.yaml -n dev

# Verify
oc get pipeline maven-build-ci-konflux -n dev
```

### 2.4: Update PipelineRun Trigger

Update your `.tekton/on-push.yaml` to use the new pipeline:

```yaml
# .tekton/on-push.yaml
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  name: maven-image-build
  annotations:
    pipelinesascode.tekton.dev/on-cel-expression: |
      body.object_kind == "push" && body.ref == "refs/heads/master"
    pipelinesascode.tekton.dev/pipeline: "https://gitlab.example.com/rhdh/tssc-sample-pipelines/-/raw/main/pipelines/maven-build-ci-konflux.yaml"
    # ... task annotations ...
  labels:
    # CRITICAL: Label for Konflux to discover this PipelineRun
    appstudio.openshift.io/component: YOUR_COMPONENT_NAME  # Set in step 3
    backstage.io/kubernetes-id: YOUR_APP_NAME
spec:
  params:
    - name: component-name
      value: YOUR_COMPONENT_NAME
    - name: git-url
      value: '{{repo_url}}'
    - name: output-image
      value: "quay.io/YOUR_ORG/YOUR_COMPONENT:{{revision}}"
    - name: revision
      value: '{{revision}}'
    - name: event-type
      value: '{{event_type}}'
  pipelineRef:
    name: maven-build-ci-konflux
  workspaces:
    - name: workspace
      volumeClaimTemplate:
        spec:
          accessModes:
          - ReadWriteOnce
          resources:
            requests:
              storage: 1Gi
    - name: maven-settings
      emptyDir: {}
    - name: git-auth
      secret:
        secretName: git-auth-secret
```

---

## Step 3: Create Application & Component

Konflux needs to know about your application and its components.

### 3.1: Create Application

```bash
# Replace values:
# - YOUR_APP_NAME: e.g., "my-java-app"
# - YOUR_DISPLAY_NAME: e.g., "My Java Application"

cat <<EOF | oc apply -f -
apiVersion: appstudio.redhat.com/v1alpha1
kind: Application
metadata:
  name: YOUR_APP_NAME
  namespace: dev
spec:
  displayName: YOUR_DISPLAY_NAME
  description: "Java application built with Maven"
EOF

# Verify
oc get application YOUR_APP_NAME -n dev
```

### 3.2: Create Component

```bash
# Replace values:
# - YOUR_COMPONENT_NAME: e.g., "backend"
# - YOUR_GIT_URL: Your source repository
# - YOUR_IMAGE_REPO: Your image repository (without tag)

cat <<EOF | oc apply -f -
apiVersion: appstudio.redhat.com/v1alpha1
kind: Component
metadata:
  name: YOUR_COMPONENT_NAME
  namespace: dev
spec:
  application: YOUR_APP_NAME
  componentName: YOUR_COMPONENT_NAME

  # Source configuration
  source:
    git:
      url: YOUR_GIT_URL
      revision: main  # or master
      context: ./

  # Build configuration
  build:
    containerImage: YOUR_IMAGE_REPO

  # Link to your build pipeline
  buildPipelineRef:
    name: maven-build-ci-konflux
EOF

# Verify
oc get component YOUR_COMPONENT_NAME -n dev
```

**What this does:**
- Tells Konflux to watch for PipelineRuns that build this component
- When a PipelineRun completes with the right labels, Konflux creates a Snapshot

---

## Step 4: Create Integration Tests

Integration tests run automatically on new Snapshots before promotion.

### 4.1: Create Integration Test Pipeline

This pipeline runs tests against the built image:

```yaml
# pipelines/integration-tests.yaml
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: integration-tests
  namespace: dev
spec:
  params:
    - name: SNAPSHOT
      description: JSON representation of the snapshot
      type: string

  tasks:
    - name: parse-snapshot
      taskSpec:
        params:
          - name: SNAPSHOT
        results:
          - name: IMAGE_URL
        steps:
          - name: parse
            image: quay.io/konflux-ci/konflux-test:latest
            script: |
              #!/bin/sh
              # Extract image URL from snapshot
              echo '$(params.SNAPSHOT)' | jq -r '.components[0].containerImage' | tee $(results.IMAGE_URL.path)
      params:
        - name: SNAPSHOT
          value: $(params.SNAPSHOT)

    - name: run-tests
      runAfter:
        - parse-snapshot
      taskSpec:
        params:
          - name: IMAGE_URL
        steps:
          - name: test
            image: $(params.IMAGE_URL)
            script: |
              #!/bin/sh
              echo "Running integration tests..."

              # Example: Run health check
              # curl http://localhost:8080/health || exit 1

              # Example: Run smoke tests
              # ./run-smoke-tests.sh || exit 1

              echo "All tests passed!"
      params:
        - name: IMAGE_URL
          value: $(tasks.parse-snapshot.results.IMAGE_URL)
```

Apply it:

```bash
oc apply -f pipelines/integration-tests.yaml -n dev
```

### 4.2: Create IntegrationTestScenario

This tells Konflux to run the test pipeline on new Snapshots:

```bash
cat <<EOF | oc apply -f -
apiVersion: appstudio.redhat.com/v1beta1
kind: IntegrationTestScenario
metadata:
  name: integration-tests
  namespace: dev
spec:
  application: YOUR_APP_NAME

  # The pipeline to run
  resolverRef:
    resolver: cluster
    params:
      - name: name
        value: integration-tests
      - name: namespace
        value: dev
      - name: kind
        value: Pipeline

  # Run on all new snapshots
  contexts:
    - name: integration
      description: Run integration tests
EOF

# Verify
oc get integrationtestscenario integration-tests -n dev
```

**What this does:**
- When Konflux creates a Snapshot, it checks for IntegrationTestScenarios
- It runs the specified pipeline, passing the Snapshot as a parameter
- If tests pass, the Snapshot is marked as validated
- Only validated Snapshots can be released

---

## Step 5: Create Release Pipeline

This pipeline handles the actual promotion to stage/prod.

### 5.1: Create Release Pipeline

```yaml
# pipelines/release-pipeline.yaml
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: release-pipeline
  labels:
    release.appstudio.openshift.io/pipeline: "true"
spec:
  params:
    - name: release
      type: string
      description: "Namespaced name of the Release"
    - name: releasePlan
      type: string
      description: "Namespaced name of the ReleasePlan"
    - name: releasePlanAdmission
      type: string
      description: "Namespaced name of the ReleasePlanAdmission"
    - name: releaseServiceConfig
      type: string
      description: "Namespaced name of the ReleaseServiceConfig"
    - name: snapshot
      type: string
      description: "JSON string of the Snapshot"
    - name: enterpriseContractPolicy
      type: string
      default: ""
      description: "Enterprise Contract policy"
    - name: enterpriseContractPublicKey
      type: string
      default: "k8s://openshift-pipelines/signing-secrets"
    - name: postCleanUp
      type: string
      default: "true"

  workspaces:
    - name: release-workspace

  tasks:
    # Task 1: Extract data from Snapshot
    - name: extract-snapshot-data
      taskSpec:
        params:
          - name: snapshot
        results:
          - name: IMAGE_URL
          - name: IMAGE_DIGEST
          - name: COMPONENT_NAME
        steps:
          - name: parse
            image: quay.io/konflux-ci/release-utils:latest
            script: |
              #!/bin/sh
              # Parse snapshot JSON
              SNAPSHOT='$(params.snapshot)'

              # Extract image URL and digest
              echo "$SNAPSHOT" | jq -r '.components[0].containerImage' | cut -d@ -f1 | tee $(results.IMAGE_URL.path)
              echo "$SNAPSHOT" | jq -r '.components[0].containerImage' | cut -d@ -f2 | tee $(results.IMAGE_DIGEST.path)
              echo "$SNAPSHOT" | jq -r '.components[0].name' | tee $(results.COMPONENT_NAME.path)
      params:
        - name: snapshot
          value: $(params.snapshot)

    # Task 2: Verify Enterprise Contract
    - name: verify-enterprise-contract
      runAfter:
        - extract-snapshot-data
      taskRef:
        name: verify-enterprise-contract
      params:
        - name: IMAGES
          value: '{"components":[{"containerImage":"$(tasks.extract-snapshot-data.results.IMAGE_URL)@$(tasks.extract-snapshot-data.results.IMAGE_DIGEST)"}]}'
        - name: POLICY_CONFIGURATION
          value: $(params.enterpriseContractPolicy)
        - name: PUBLIC_KEY
          value: $(params.enterpriseContractPublicKey)
        - name: STRICT
          value: "true"

    # Task 3: Extract target environment from ReleasePlanAdmission
    - name: get-target-environment
      runAfter:
        - verify-enterprise-contract
      taskSpec:
        params:
          - name: releasePlanAdmission
        results:
          - name: ENVIRONMENT
        steps:
          - name: get-env
            image: quay.io/openshift/origin-cli:latest
            script: |
              #!/bin/sh
              # Extract namespace from releasePlanAdmission (format: namespace/name)
              NAMESPACE=$(echo "$(params.releasePlanAdmission)" | cut -d/ -f1)

              # Get environment from ReleasePlanAdmission
              ENV=$(oc get releaseplanadmission -n $NAMESPACE \
                $(echo "$(params.releasePlanAdmission)" | cut -d/ -f2) \
                -o jsonpath='{.spec.environment}')

              echo -n "$ENV" | tee $(results.ENVIRONMENT.path)
      params:
        - name: releasePlanAdmission
          value: $(params.releasePlanAdmission)

    # Task 4: Copy image to environment-specific tag
    - name: copy-image
      runAfter:
        - get-target-environment
      taskRef:
        name: skopeo-copy
      params:
        - name: SOURCE_IMAGE_URL
          value: "docker://$(tasks.extract-snapshot-data.results.IMAGE_URL)@$(tasks.extract-snapshot-data.results.IMAGE_DIGEST)"
        - name: DESTINATION_IMAGE_URL
          value: "docker://$(tasks.extract-snapshot-data.results.IMAGE_URL):$(tasks.get-target-environment.results.ENVIRONMENT)"
        - name: SRC_TLS_VERIFY
          value: "true"
        - name: DEST_TLS_VERIFY
          value: "true"

    # Task 5: Update GitOps for target environment
    - name: update-gitops
      runAfter:
        - copy-image
      taskRef:
        name: update-deployment
      params:
        - name: gitops-repo-url
          # This should come from ReleasePlanAdmission data
          value: "https://gitlab.example.com/YOUR_ORG/YOUR_APP-gitops"
        - name: image
          value: "$(tasks.extract-snapshot-data.results.IMAGE_URL):$(tasks.get-target-environment.results.ENVIRONMENT)"
        - name: environment
          value: "$(tasks.get-target-environment.results.ENVIRONMENT)"
      workspaces:
        - name: gitops-auth
          workspace: release-workspace

  finally:
    - name: cleanup
      taskSpec:
        steps:
          - name: log
            image: registry.access.redhat.com/ubi9/ubi-minimal:latest
            script: |
              #!/bin/sh
              echo "Release pipeline completed"
              echo "Environment: $(tasks.get-target-environment.results.ENVIRONMENT)"
              echo "Image: $(tasks.extract-snapshot-data.results.IMAGE_URL):$(tasks.get-target-environment.results.ENVIRONMENT)"
```

Apply to ALL namespaces (dev, stage, prod):

```bash
oc apply -f pipelines/release-pipeline.yaml -n dev
oc apply -f pipelines/release-pipeline.yaml -n stage
oc apply -f pipelines/release-pipeline.yaml -n prod
```

---

## Step 6: Setup Stage (Auto-Release)

Stage gets automatic releases when snapshots pass integration tests.

### 6.1: Create ReleasePlan for Stage

```bash
cat <<EOF | oc apply -f -
apiVersion: appstudio.redhat.com/v1alpha1
kind: ReleasePlan
metadata:
  name: release-to-stage
  namespace: dev
  labels:
    # Enable auto-release
    release.appstudio.openshift.io/auto-release: "true"
spec:
  application: YOUR_APP_NAME
  target: stage  # Target namespace

  # Pipeline reference (in target namespace)
  pipelineRef:
    name: release-pipeline
    namespace: stage

  # Optional: Data to pass to the pipeline
  data:
    gitOpsRepo: "https://gitlab.example.com/YOUR_ORG/YOUR_APP-gitops"
EOF

# Verify
oc get releaseplan release-to-stage -n dev
```

### 6.2: Create ReleasePlanAdmission for Stage

```bash
cat <<EOF | oc apply -f -
apiVersion: appstudio.redhat.com/v1alpha1
kind: ReleasePlanAdmission
metadata:
  name: accept-dev-releases
  namespace: stage
spec:
  # Accept releases from dev namespace
  origin: dev

  # For this application
  applications:
    - YOUR_APP_NAME

  # Environment name
  environment: stage

  # Enterprise Contract policy
  policy: stage-policy

  # Pipeline to run
  pipelineRef:
    name: release-pipeline
    namespace: stage

  # Service account
  serviceAccount: release-service-account

  # Additional data
  data:
    # Enterprise Contract configuration
    enterpriseContract:
      policy: "github.com/enterprise-contract/config//slsa3"
      publicKey: "k8s://openshift-pipelines/signing-secrets"
EOF

# Verify
oc get releaseplanadmission accept-dev-releases -n stage
```

### 6.3: Create Service Account in Stage

```bash
# Create service account for releases
oc create serviceaccount release-service-account -n stage

# Grant permissions to update GitOps repo
oc create rolebinding release-edit \
  --serviceaccount=stage:release-service-account \
  --clusterrole=edit \
  -n stage

# Link registry secret
oc secrets link release-service-account quay-credentials -n stage

# Link GitOps secret
oc create secret generic gitops-auth \
  --from-literal=username=YOUR_GIT_USER \
  --from-literal=password=YOUR_GIT_TOKEN \
  -n stage

oc secrets link release-service-account gitops-auth -n stage
```

---

## Step 7: Setup Production (Manual Release)

Production requires manual Release creation for approval.

### 7.1: Create ReleasePlan for Production

```bash
cat <<EOF | oc apply -f -
apiVersion: appstudio.redhat.com/v1alpha1
kind: ReleasePlan
metadata:
  name: release-to-prod
  namespace: dev
  labels:
    # DISABLE auto-release for prod
    release.appstudio.openshift.io/auto-release: "false"
spec:
  application: YOUR_APP_NAME
  target: prod

  pipelineRef:
    name: release-pipeline
    namespace: prod

  data:
    gitOpsRepo: "https://gitlab.example.com/YOUR_ORG/YOUR_APP-gitops"
EOF

# Verify
oc get releaseplan release-to-prod -n dev
```

### 7.2: Create ReleasePlanAdmission for Production

```bash
cat <<EOF | oc apply -f -
apiVersion: appstudio.redhat.com/v1alpha1
kind: ReleasePlanAdmission
metadata:
  name: accept-dev-releases
  namespace: prod
spec:
  origin: dev
  applications:
    - YOUR_APP_NAME
  environment: prod
  policy: prod-strict-policy

  pipelineRef:
    name: release-pipeline
    namespace: prod

  serviceAccount: release-service-account

  data:
    enterpriseContract:
      # Stricter policy for production
      policy: "github.com/enterprise-contract/config//slsa3-strict"
      publicKey: "k8s://openshift-pipelines/signing-secrets"
      strict: "true"
EOF

# Verify
oc get releaseplanadmission accept-dev-releases -n prod
```

### 7.3: Create Service Account in Prod

```bash
# Same as stage
oc create serviceaccount release-service-account -n prod
oc create rolebinding release-edit \
  --serviceaccount=prod:release-service-account \
  --clusterrole=edit \
  -n prod
oc secrets link release-service-account quay-credentials -n prod

oc create secret generic gitops-auth \
  --from-literal=username=YOUR_GIT_USER \
  --from-literal=password=YOUR_GIT_TOKEN \
  -n prod
oc secrets link release-service-account gitops-auth -n prod
```

---

## Step 8: Remove Git Tag Triggers

Now that Konflux handles promotions, remove the old tag-based triggers.

### 8.1: Remove Tag Trigger

```bash
# In your repository, delete or rename .tekton/on-tag.yaml
cd your-app-repo
git mv .tekton/on-tag.yaml .tekton/on-tag.yaml.disabled
git commit -m "Disable tag-based stage promotion (using Konflux now)"
git push
```

### 8.2: Remove Release Trigger

```bash
# Delete or rename .tekton/on-release.yaml
git mv .tekton/on-release.yaml .tekton/on-release.yaml.disabled
git commit -m "Disable release-based prod promotion (using Konflux now)"
git push
```

### 8.3: Keep Build Trigger

**Keep** `.tekton/on-push.yaml` - this still triggers builds on git push.

---

## Testing the Flow

### Test 1: Automatic Stage Release

```bash
# 1. Make a code change
cd your-app-repo
echo "// test change" >> src/main/java/SomeFile.java
git add .
git commit -m "Test Konflux flow"
git push origin main

# 2. Watch build pipeline
oc get pipelineruns -n dev -w

# 3. Watch for Snapshot creation (automatically created when build succeeds)
oc get snapshots -n dev -w

# 4. Watch integration tests run
oc get pipelineruns -n dev | grep integration-tests

# 5. Watch for automatic Release creation (if tests pass)
oc get releases -n dev -w

# 6. Watch release pipeline in stage
oc get pipelineruns -n stage -w

# 7. Verify image in stage
oc get deployment -n stage -o yaml | grep image:
```

### Test 2: Manual Production Release

```bash
# 1. List available snapshots
oc get snapshots -n dev --sort-by=.metadata.creationTimestamp

# 2. Choose a validated snapshot
SNAPSHOT_NAME="YOUR_APP_NAME-snapshot-abc123"

# 3. Create Release CR
cat <<EOF | oc apply -f -
apiVersion: appstudio.redhat.com/v1alpha1
kind: Release
metadata:
  name: prod-release-v1.0.0
  namespace: dev
  annotations:
    release.appstudio.io/approved-by: "your.name@example.com"
    release.appstudio.io/change-ticket: "CHG-12345"
spec:
  snapshot: ${SNAPSHOT_NAME}
  releasePlan: release-to-prod
EOF

# 4. Watch release
oc get release prod-release-v1.0.0 -n dev -w

# 5. Watch release pipeline in prod
oc get pipelineruns -n prod -w

# 6. Verify deployment
oc get deployment -n prod -o yaml | grep image:
```

---

## Developer Workflow

### Daily Development

```bash
# 1. Make changes
git add .
git commit -m "Feature: Add new endpoint"
git push origin main

# That's it!
# → Build runs
# → Snapshot created
# → Tests run
# → If tests pass, promoted to stage automatically
```

### Check Status

```bash
# See recent snapshots
oc get snapshots -n dev

# See test results
oc get integrationtestscenario -n dev
oc get pipelineruns -n dev | grep integration

# See releases
oc get releases -n dev
```

---

## Operations Workflow

### Promote to Production

```bash
# 1. Review validated snapshots
oc get snapshots -n dev -l test-status=passed

# 2. Get details of a snapshot
oc describe snapshot YOUR_APP_NAME-snapshot-abc123 -n dev

# 3. Create release (with approvals)
cat <<EOF | oc apply -f -
apiVersion: appstudio.redhat.com/v1alpha1
kind: Release
metadata:
  name: prod-release-$(date +%Y%m%d-%H%M)
  namespace: dev
  annotations:
    release.appstudio.io/approved-by: "ops.team@example.com"
    release.appstudio.io/jira-ticket: "PROD-123"
    release.appstudio.io/change-window: "2026-02-17 14:00 UTC"
spec:
  snapshot: YOUR_APP_NAME-snapshot-abc123
  releasePlan: release-to-prod
EOF

# 4. Monitor
oc get release -n dev -w
oc get pipelineruns -n prod -w
```

### Rollback

```bash
# 1. Find previous release
oc get releases -n dev -l release.appstudio.io/target=prod \
  --sort-by=.metadata.creationTimestamp

# 2. Get the snapshot from previous release
PREV_SNAPSHOT=$(oc get release PREVIOUS_RELEASE_NAME -n dev \
  -o jsonpath='{.spec.snapshot}')

# 3. Create new release with old snapshot
cat <<EOF | oc apply -f -
apiVersion: appstudio.redhat.com/v1alpha1
kind: Release
metadata:
  name: prod-rollback-$(date +%Y%m%d-%H%M)
  namespace: dev
  annotations:
    release.appstudio.io/rollback: "true"
    release.appstudio.io/rollback-from: "CURRENT_RELEASE_NAME"
spec:
  snapshot: ${PREV_SNAPSHOT}
  releasePlan: release-to-prod
EOF
```

---

## Summary

You've now implemented full Konflux!

### What Changed

| Before | After |
|--------|-------|
| `git tag v1.0.0-stage` → promote | `git push` → automatic stage promotion |
| GitLab Release → promote to prod | Create Release CR → promote to prod |
| Tags in git history | Clean git history |
| Manual pipeline triggers | Automated Snapshot → Release flow |

### Key Benefits

1. **No Git Pollution** - Git history reflects code, not deployments
2. **Automatic Testing** - Integration tests gate promotions
3. **Atomic Releases** - Multiple components released together
4. **Audit Trail** - Complete record in Release CRs
5. **Easy Rollback** - Create Release from any previous Snapshot
6. **Policy Enforcement** - Enterprise Contract validation built-in

### What Developers Do Now

```bash
git push origin main
# Everything else happens automatically for stage
# Ops team creates Releases for production
```

**That's it!** Konflux handles the rest.
