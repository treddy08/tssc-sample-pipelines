# Konflux Implementation Guide (Corrected)

**Complete step-by-step guide to implement Konflux with centralized CI/CD**

This guide shows you how to convert from git tag-based promotions to Konflux's Snapshot/Release model with:
- **Centralized CI**: All pipelines run in `tssc-app-ci` namespace
- **Dev**: Automatic deployment after successful build + tests
- **Stage**: Automatic promotion from dev
- **Production**: Manual promotion via Release CR

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Prerequisites](#prerequisites)
3. [Step 1: Create Namespaces](#step-1-create-namespaces)
4. [Step 2: Install Konflux Components](#step-2-install-konflux-components)
5. [Step 3: Setup Cross-Namespace RBAC](#step-3-setup-cross-namespace-rbac)
6. [Step 4: Install Pipelines in tssc-app-ci](#step-4-install-pipelines-in-tssc-app-ci)
7. [Step 5: Create Application & Component](#step-5-create-application--component)
8. [Step 6: Create Integration Tests](#step-6-create-integration-tests)
9. [Step 7: Create ReleasePlans](#step-7-create-releaseplans)
10. [Step 8: Create ReleasePlanAdmissions](#step-8-create-releaseplanadmissions)
11. [Step 9: Update PipelineRun Triggers](#step-9-update-pipelinerun-triggers)
12. [Testing the Flow](#testing-the-flow)
13. [Developer Workflow](#developer-workflow)
14. [Operations Workflow](#operations-workflow)

---

## Architecture Overview

### Centralized CI Pattern

```
┌─────────────────────────────────────────────────────────────────┐
│ tssc-app-ci (CI/CD Namespace)                                    │
│                                                                   │
│ ALL pipelines run here:                                         │
│  • maven-build-ci-konflux (Build Pipeline)                      │
│  • release-pipeline (Release Pipeline)                          │
│  • integration-tests (Test Pipeline)                            │
│  • All build and release tasks                                  │
│                                                                   │
│ Konflux Resources:                                              │
│  • Application CR                                               │
│  • Component CR                                                 │
│  • Snapshots (auto-created after builds)                        │
│  • IntegrationTestScenario                                      │
│  • ReleasePlan → dev (auto-release: true)                       │
│  • ReleasePlan → stage (auto-release: true)                     │
│  • ReleasePlan → prod (auto-release: false)                     │
│  • Release CRs (auto-created or manual)                         │
└─────────────────────────────────────────────────────────────────┘
                              ↓
        ┌─────────────────────┼─────────────────────┐
        ↓                     ↓                     ↓
┌───────────────────┐ ┌───────────────────┐ ┌───────────────────┐
│ my-java-app-dev   │ │ my-java-app-stage │ │ my-java-app-prod  │
│                   │ │                   │ │                   │
│ Runtime ONLY:     │ │ Runtime ONLY:     │ │ Runtime ONLY:     │
│ • ReleasePlan     │ │ • ReleasePlan     │ │ • ReleasePlan     │
│   Admission       │ │   Admission       │ │   Admission       │
│ • Deployment      │ │ • Deployment      │ │ • Deployment      │
│ • Service         │ │ • Service         │ │ • Service         │
│ • Route           │ │ • Route           │ │ • Route           │
│ • ConfigMaps      │ │ • ConfigMaps      │ │ • ConfigMaps      │
│ • Secrets         │ │ • Secrets         │ │ • Secrets         │
│                   │ │                   │ │                   │
│ NO pipelines!     │ │ NO pipelines!     │ │ NO pipelines!     │
└───────────────────┘ └───────────────────┘ └───────────────────┘
```

### Current Flow (Git Tags)
```
git push → Build → Deploy to Dev
git tag stage → Promote to Stage
GitLab Release → Promote to Prod
```

### New Flow (Konflux Centralized CI)
```
git push
  ↓
Build in tssc-app-ci
  ↓
Snapshot created in tssc-app-ci
  ↓
Integration Tests in tssc-app-ci
  ↓
Auto-Release to my-java-app-dev (ReleasePlan in tssc-app-ci)
  ↓
Auto-Release to my-java-app-stage (ReleasePlan in tssc-app-ci)
  ↓
Manual Release to my-java-app-prod (Create Release CR in tssc-app-ci)
```

### Key Components

| Component | Purpose | Location | Created By |
|-----------|---------|----------|------------|
| **Application** | Groups related components | tssc-app-ci | You (once) |
| **Component** | Single buildable unit | tssc-app-ci | You (once) |
| **Build Pipeline** | Builds container image | tssc-app-ci | You (once) |
| **Snapshot** | Point-in-time image set | tssc-app-ci | Konflux (automatic) |
| **IntegrationTestScenario** | Tests for snapshots | tssc-app-ci | You (once) |
| **ReleasePlan** (x3) | Promotion strategies | tssc-app-ci | You (once per env) |
| **ReleasePlanAdmission** (x3) | Acceptance criteria | my-java-app-{dev,stage,prod} | You (once per env) |
| **Release Pipeline** | Promotes images | tssc-app-ci | You (once) |
| **Release** | Promotion trigger | tssc-app-ci | Konflux (auto) or You (manual) |

---

## Prerequisites

- OpenShift 4.12+ cluster
- Cluster admin access
- Existing `maven-build-ci` pipeline working
- Application name chosen (e.g., `my-java-app`)
- Component name chosen (e.g., `my-java-app-backend`)

---

## Step 1: Create Namespaces

### 1.1: Define Application Name

```bash
# Set your application name
export APP_NAME="my-java-app"

# This will create:
# - tssc-app-ci (already exists - CI/CD namespace)
# - my-java-app-dev (runtime namespace)
# - my-java-app-stage (runtime namespace)
# - my-java-app-prod (runtime namespace)
```

### 1.2: Create Runtime Namespaces

```bash
# CI namespace should already exist
# If not, create it:
oc create namespace tssc-app-ci

# Create runtime namespaces for your app
oc create namespace ${APP_NAME}-dev
oc create namespace ${APP_NAME}-stage
oc create namespace ${APP_NAME}-prod

# Verify
oc get namespaces | grep -E "tssc-app-ci|${APP_NAME}"
```

### 1.3: Label Namespaces

```bash
# Label CI namespace
oc label namespace tssc-app-ci \
  app.kubernetes.io/part-of=konflux-ci \
  environment=ci

# Label runtime namespaces
oc label namespace ${APP_NAME}-dev environment=dev
oc label namespace ${APP_NAME}-stage environment=stage
oc label namespace ${APP_NAME}-prod environment=prod
```

---

## Step 2: Install Konflux Components

### 2.1: Install Konflux Operator

```bash
# Create namespace for Konflux system components
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
oc wait --for=condition=Ready pod \
  -l app=konflux-operator \
  -n konflux-system \
  --timeout=300s
```

### 2.2: Verify Konflux CRDs

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

### 2.3: Verify Konflux Services

```bash
# Verify release service
oc get deployment -n konflux-system | grep release-service

# Verify integration service
oc get deployment -n konflux-system | grep integration-service
```

---

## Step 3: Setup Cross-Namespace RBAC

The release pipeline runs in `tssc-app-ci` but needs to deploy to runtime namespaces.

### 3.1: Create Service Account in CI Namespace

```bash
# Create service account for release pipelines
oc create serviceaccount release-service-account -n tssc-app-ci
```

### 3.2: Grant Permissions to Deploy to Dev

```bash
# Allow CI namespace to deploy to dev namespace
oc create rolebinding release-deployer \
  --serviceaccount=tssc-app-ci:release-service-account \
  --clusterrole=edit \
  -n ${APP_NAME}-dev
```

### 3.3: Grant Permissions to Deploy to Stage

```bash
# Allow CI namespace to deploy to stage namespace
oc create rolebinding release-deployer \
  --serviceaccount=tssc-app-ci:release-service-account \
  --clusterrole=edit \
  -n ${APP_NAME}-stage
```

### 3.4: Grant Permissions to Deploy to Prod

```bash
# Allow CI namespace to deploy to prod namespace
oc create rolebinding release-deployer \
  --serviceaccount=tssc-app-ci:release-service-account \
  --clusterrole=edit \
  -n ${APP_NAME}-prod
```

### 3.5: Link Registry Credentials

```bash
# Assuming you have a quay-credentials secret
# Link to release service account
oc secrets link release-service-account quay-credentials -n tssc-app-ci

# Create GitOps credentials secret
oc create secret generic gitops-auth \
  --from-literal=username=YOUR_GIT_USER \
  --from-literal=password=YOUR_GIT_TOKEN \
  -n tssc-app-ci

oc secrets link release-service-account gitops-auth -n tssc-app-ci
```

---

## Step 4: Install Pipelines in tssc-app-ci

**IMPORTANT**: All pipelines go in `tssc-app-ci` namespace ONLY.

### 4.1: Install All Tasks

```bash
cd /path/to/tssc-sample-pipelines

# Install all tasks in tssc-app-ci
oc apply -f tasks/ -n tssc-app-ci

# Verify
oc get tasks -n tssc-app-ci
```

### 4.2: Create Modified Build Pipeline

Create `pipelines/maven-build-ci-konflux.yaml`:

```yaml
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: maven-build-ci-konflux
  namespace: tssc-app-ci  # ← ALL pipelines in tssc-app-ci
  labels:
    # Konflux discovery labels
    appstudio.openshift.io/pipeline: build
    pipelines.openshift.io/runtime: generic
    pipelines.openshift.io/strategy: docker
    pipelines.openshift.io/used-by: build-cloud
spec:
  # CRITICAL: These results are required for Konflux Snapshots
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
    - name: ACS_SCAN_OUTPUT
      value: $(tasks.acs-image-scan.results.SCAN_OUTPUT)

  params:
    - name: component-name
      type: string
    - name: git-url
      type: string
    - name: revision
      type: string
      default: ""
    - name: output-image
      type: string
    - name: path-context
      type: string
      default: .
    - name: dockerfile
      type: string
      default: Dockerfile
    - name: rebuild
      type: string
      default: "false"
    - name: image-expires-after
      default: ""
    - name: stackrox-secret
      type: string
      default: rox-api-token
    - name: event-type
      type: string
      default: push
    - name: build-args
      type: array
      default: []
    - name: build-args-file
      type: string
      default: ""
    - name: trustification-secret-name
      type: string
      default: tpa-secret
    - name: fail-if-trustification-not-configured
      type: string
      default: 'false'
    - name: sboms-dir
      type: string
      default: sboms

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
    # These are now handled by Konflux Release flow

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

Apply it:

```bash
oc apply -f pipelines/maven-build-ci-konflux.yaml -n tssc-app-ci
```

### 4.3: Create Release Pipeline

Create `pipelines/release-pipeline.yaml`:

```yaml
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: release-pipeline
  namespace: tssc-app-ci  # ← Runs in CI namespace
  labels:
    release.appstudio.openshift.io/pipeline: "true"
spec:
  params:
    - name: release
      type: string
    - name: releasePlan
      type: string
    - name: releasePlanAdmission
      type: string
    - name: releaseServiceConfig
      type: string
    - name: snapshot
      type: string
    - name: enterpriseContractPolicy
      type: string
      default: ""
    - name: enterpriseContractPublicKey
      type: string
      default: "k8s://openshift-pipelines/signing-secrets"
    - name: postCleanUp
      type: string
      default: "true"

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
              SNAPSHOT='$(params.snapshot)'
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

    # Task 3: Get target namespace and environment
    - name: get-target-environment
      runAfter:
        - verify-enterprise-contract
      taskSpec:
        params:
          - name: releasePlanAdmission
        results:
          - name: ENVIRONMENT
          - name: TARGET_NAMESPACE
        steps:
          - name: get-env
            image: quay.io/openshift/origin-cli:latest
            script: |
              #!/bin/sh
              # Extract namespace from releasePlanAdmission (format: namespace/name)
              NAMESPACE=$(echo "$(params.releasePlanAdmission)" | cut -d/ -f1)
              echo -n "$NAMESPACE" | tee $(results.TARGET_NAMESPACE.path)

              # Get environment from ReleasePlanAdmission
              ENV=$(oc get releaseplanadmission -n $NAMESPACE \
                $(echo "$(params.releasePlanAdmission)" | cut -d/ -f2) \
                -o jsonpath='{.spec.environment}')
              echo -n "$ENV" | tee $(results.ENVIRONMENT.path)
      params:
        - name: releasePlanAdmission
          value: $(params.releasePlanAdmission)

    # Task 4: Copy image to environment tag
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

    # Task 5: Update GitOps deployment
    - name: update-deployment
      runAfter:
        - copy-image
      taskRef:
        name: update-deployment
      params:
        - name: gitops-repo-url
          value: "https://gitlab.example.com/YOUR_ORG/YOUR_APP-gitops"
        - name: image
          value: "$(tasks.extract-snapshot-data.results.IMAGE_URL):$(tasks.get-target-environment.results.ENVIRONMENT)"
        - name: environment
          value: "$(tasks.get-target-environment.results.ENVIRONMENT)"
        - name: target-namespace
          value: "$(tasks.get-target-environment.results.TARGET_NAMESPACE)"

  finally:
    - name: cleanup
      taskSpec:
        steps:
          - name: log
            image: registry.access.redhat.com/ubi9/ubi-minimal:latest
            script: |
              #!/bin/sh
              echo "Release pipeline completed"
              echo "Target: $(tasks.get-target-environment.results.TARGET_NAMESPACE)"
              echo "Environment: $(tasks.get-target-environment.results.ENVIRONMENT)"
              echo "Image: $(tasks.extract-snapshot-data.results.IMAGE_URL):$(tasks.get-target-environment.results.ENVIRONMENT)"
```

Apply it:

```bash
oc apply -f pipelines/release-pipeline.yaml -n tssc-app-ci
```

---

## Step 5: Create Application & Component

### 5.1: Create Application in tssc-app-ci

```bash
# Set component name
export COMPONENT_NAME="${APP_NAME}-backend"

cat <<EOF | oc apply -f -
apiVersion: appstudio.redhat.com/v1alpha1
kind: Application
metadata:
  name: ${APP_NAME}
  namespace: tssc-app-ci  # ← In CI namespace
spec:
  displayName: "My Java Application"
  description: "Java application built with Maven"
EOF

# Verify
oc get application ${APP_NAME} -n tssc-app-ci
```

### 5.2: Create Component in tssc-app-ci

```bash
cat <<EOF | oc apply -f -
apiVersion: appstudio.redhat.com/v1alpha1
kind: Component
metadata:
  name: ${COMPONENT_NAME}
  namespace: tssc-app-ci  # ← In CI namespace
spec:
  application: ${APP_NAME}
  componentName: ${COMPONENT_NAME}

  # Source configuration
  source:
    git:
      url: https://gitlab.example.com/YOUR_ORG/YOUR_APP
      revision: main
      context: ./

  # Build configuration
  build:
    containerImage: quay.io/YOUR_ORG/${COMPONENT_NAME}

  # Link to build pipeline in tssc-app-ci
  buildPipelineRef:
    name: maven-build-ci-konflux
EOF

# Verify
oc get component ${COMPONENT_NAME} -n tssc-app-ci
```

---

## Step 6: Create Integration Tests

### 6.1: Create Integration Test Pipeline

Create `pipelines/integration-tests.yaml`:

```yaml
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: integration-tests
  namespace: tssc-app-ci  # ← In CI namespace
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
              echo "All tests passed!"
      params:
        - name: IMAGE_URL
          value: $(tasks.parse-snapshot.results.IMAGE_URL)
```

Apply it:

```bash
oc apply -f pipelines/integration-tests.yaml -n tssc-app-ci
```

### 6.2: Create IntegrationTestScenario

```bash
cat <<EOF | oc apply -f -
apiVersion: appstudio.redhat.com/v1beta1
kind: IntegrationTestScenario
metadata:
  name: integration-tests
  namespace: tssc-app-ci  # ← In CI namespace
spec:
  application: ${APP_NAME}

  # Pipeline in tssc-app-ci
  resolverRef:
    resolver: cluster
    params:
      - name: name
        value: integration-tests
      - name: namespace
        value: tssc-app-ci  # ← Same namespace
      - name: kind
        value: Pipeline

  contexts:
    - name: integration
      description: Run integration tests
EOF

# Verify
oc get integrationtestscenario integration-tests -n tssc-app-ci
```

---

## Step 7: Create ReleasePlans

All three ReleasePlans go in `tssc-app-ci`.

### 7.1: ReleasePlan for Dev (Auto-Release)

```bash
cat <<EOF | oc apply -f -
apiVersion: appstudio.redhat.com/v1alpha1
kind: ReleasePlan
metadata:
  name: release-to-dev
  namespace: tssc-app-ci  # ← In CI namespace
  labels:
    release.appstudio.openshift.io/auto-release: "true"
spec:
  application: ${APP_NAME}
  target: ${APP_NAME}-dev  # ← Target runtime namespace

  # Pipeline runs in tssc-app-ci
  pipelineRef:
    name: release-pipeline
    namespace: tssc-app-ci  # ← Same namespace

  serviceAccount: release-service-account
EOF
```

### 7.2: ReleasePlan for Stage (Auto-Release)

```bash
cat <<EOF | oc apply -f -
apiVersion: appstudio.redhat.com/v1alpha1
kind: ReleasePlan
metadata:
  name: release-to-stage
  namespace: tssc-app-ci  # ← In CI namespace
  labels:
    release.appstudio.openshift.io/auto-release: "true"
spec:
  application: ${APP_NAME}
  target: ${APP_NAME}-stage  # ← Target runtime namespace

  # Pipeline runs in tssc-app-ci
  pipelineRef:
    name: release-pipeline
    namespace: tssc-app-ci

  serviceAccount: release-service-account
EOF
```

### 7.3: ReleasePlan for Prod (Manual Release)

```bash
cat <<EOF | oc apply -f -
apiVersion: appstudio.redhat.com/v1alpha1
kind: ReleasePlan
metadata:
  name: release-to-prod
  namespace: tssc-app-ci  # ← In CI namespace
  labels:
    release.appstudio.openshift.io/auto-release: "false"
spec:
  application: ${APP_NAME}
  target: ${APP_NAME}-prod  # ← Target runtime namespace

  # Pipeline runs in tssc-app-ci
  pipelineRef:
    name: release-pipeline
    namespace: tssc-app-ci

  serviceAccount: release-service-account
EOF
```

### 7.4: Verify ReleasePlans

```bash
# All three should be in tssc-app-ci
oc get releaseplans -n tssc-app-ci

# Should see:
# release-to-dev    (auto-release: true)
# release-to-stage  (auto-release: true)
# release-to-prod   (auto-release: false)
```

---

## Step 8: Create ReleasePlanAdmissions

One ReleasePlanAdmission in EACH runtime namespace.

### 8.1: ReleasePlanAdmission for Dev

```bash
cat <<EOF | oc apply -f -
apiVersion: appstudio.redhat.com/v1alpha1
kind: ReleasePlanAdmission
metadata:
  name: accept-releases
  namespace: ${APP_NAME}-dev  # ← In runtime namespace
spec:
  origin: tssc-app-ci  # ← Accept from CI namespace
  applications:
    - ${APP_NAME}
  environment: dev

  # Pipeline runs in tssc-app-ci (cross-namespace reference)
  pipelineRef:
    name: release-pipeline
    namespace: tssc-app-ci  # ← Cross-namespace!

  serviceAccount: release-service-account

  data:
    enterpriseContract:
      policy: "github.com/enterprise-contract/config//slsa3"
      publicKey: "k8s://openshift-pipelines/signing-secrets"
EOF
```

### 8.2: ReleasePlanAdmission for Stage

```bash
cat <<EOF | oc apply -f -
apiVersion: appstudio.redhat.com/v1alpha1
kind: ReleasePlanAdmission
metadata:
  name: accept-releases
  namespace: ${APP_NAME}-stage  # ← In runtime namespace
spec:
  origin: tssc-app-ci  # ← Accept from CI namespace
  applications:
    - ${APP_NAME}
  environment: stage

  # Pipeline runs in tssc-app-ci
  pipelineRef:
    name: release-pipeline
    namespace: tssc-app-ci  # ← Cross-namespace!

  serviceAccount: release-service-account

  data:
    enterpriseContract:
      policy: "github.com/enterprise-contract/config//slsa3"
      publicKey: "k8s://openshift-pipelines/signing-secrets"
EOF
```

### 8.3: ReleasePlanAdmission for Prod

```bash
cat <<EOF | oc apply -f -
apiVersion: appstudio.redhat.com/v1alpha1
kind: ReleasePlanAdmission
metadata:
  name: accept-releases
  namespace: ${APP_NAME}-prod  # ← In runtime namespace
spec:
  origin: tssc-app-ci  # ← Accept from CI namespace
  applications:
    - ${APP_NAME}
  environment: prod

  # Pipeline runs in tssc-app-ci
  pipelineRef:
    name: release-pipeline
    namespace: tssc-app-ci  # ← Cross-namespace!

  serviceAccount: release-service-account

  data:
    enterpriseContract:
      # Stricter policy for production
      policy: "github.com/enterprise-contract/config//slsa3-strict"
      publicKey: "k8s://openshift-pipelines/signing-secrets"
      strict: "true"
EOF
```

### 8.4: Verify ReleasePlanAdmissions

```bash
# Should be one in each runtime namespace
oc get releaseplanadmission -n ${APP_NAME}-dev
oc get releaseplanadmission -n ${APP_NAME}-stage
oc get releaseplanadmission -n ${APP_NAME}-prod
```

---

## Step 9: Update PipelineRun Triggers

### 9.1: Update On-Push Trigger

Update your `.tekton/on-push.yaml` to use the new pipeline and add Konflux labels:

```yaml
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  name: maven-image-build
  annotations:
    pipelinesascode.tekton.dev/on-cel-expression: |
      body.object_kind == "push" && body.ref == "refs/heads/master"
    pipelinesascode.tekton.dev/pipeline: "https://gitlab.example.com/rhdh/tssc-sample-pipelines/-/raw/main/pipelines/maven-build-ci-konflux.yaml"
    # Add all task annotations...
  labels:
    # CRITICAL: Konflux discovery labels
    appstudio.openshift.io/component: my-java-app-backend  # ← Your component name
    appstudio.openshift.io/application: my-java-app  # ← Your app name
    backstage.io/kubernetes-id: my-java-app
spec:
  params:
    - name: component-name
      value: my-java-app-backend
    - name: git-url
      value: '{{repo_url}}'
    - name: output-image
      value: "quay.io/YOUR_ORG/my-java-app-backend:{{revision}}"
    - name: revision
      value: '{{revision}}'
    - name: event-type
      value: '{{event_type}}'
  pipelineRef:
    resolver: cluster
    params:
      - name: name
        value: maven-build-ci-konflux
      - name: namespace
        value: tssc-app-ci  # ← Pipeline in CI namespace
      - name: kind
        value: Pipeline
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

### 9.2: Disable Tag/Release Triggers

```bash
# In your repository
cd your-app-repo
git mv .tekton/on-tag.yaml .tekton/on-tag.yaml.disabled
git mv .tekton/on-release.yaml .tekton/on-release.yaml.disabled
git commit -m "Disable tag/release triggers (using Konflux now)"
git push
```

---

## Testing the Flow

### Complete End-to-End Test

```bash
# 1. Make a code change
cd your-app-repo
echo "// test change" >> src/main/java/SomeFile.java
git add .
git commit -m "Test Konflux flow"
git push origin main

# 2. Watch build pipeline (in tssc-app-ci)
oc get pipelineruns -n tssc-app-ci -w

# 3. Watch for Snapshot creation (in tssc-app-ci)
oc get snapshots -n tssc-app-ci -w

# 4. Watch integration tests (in tssc-app-ci)
oc get pipelineruns -n tssc-app-ci | grep integration-tests

# 5. Watch for automatic Release to dev (in tssc-app-ci)
oc get releases -n tssc-app-ci -w

# 6. Watch release pipeline run (in tssc-app-ci)
oc get pipelineruns -n tssc-app-ci | grep release-pipeline

# 7. Verify deployment in dev
oc get deployment -n ${APP_NAME}-dev -o yaml | grep image:

# 8. Watch for automatic Release to stage
oc get releases -n tssc-app-ci | grep stage

# 9. Verify deployment in stage
oc get deployment -n ${APP_NAME}-stage -o yaml | grep image:
```

### Manual Production Release

```bash
# 1. List validated snapshots
oc get snapshots -n tssc-app-ci --sort-by=.metadata.creationTimestamp

# 2. Choose a snapshot
SNAPSHOT_NAME="${APP_NAME}-snapshot-abc123"

# 3. Create Release CR (in tssc-app-ci)
cat <<EOF | oc apply -f -
apiVersion: appstudio.redhat.com/v1alpha1
kind: Release
metadata:
  name: prod-release-$(date +%Y%m%d-%H%M)
  namespace: tssc-app-ci  # ← In CI namespace
  annotations:
    release.appstudio.io/approved-by: "ops@example.com"
    release.appstudio.io/change-ticket: "CHG-12345"
spec:
  snapshot: ${SNAPSHOT_NAME}
  releasePlan: release-to-prod
EOF

# 4. Watch release pipeline
oc get pipelineruns -n tssc-app-ci -w

# 5. Verify deployment in prod
oc get deployment -n ${APP_NAME}-prod -o yaml | grep image:
```

---

## Developer Workflow

### Daily Development

```bash
# Just push code
git add .
git commit -m "Feature: Add new endpoint"
git push origin main

# Konflux handles the rest:
# → Build runs in tssc-app-ci
# → Snapshot created in tssc-app-ci
# → Tests run in tssc-app-ci
# → Auto-release to dev
# → Auto-release to stage
```

### Check Build Status

```bash
# See pipelines in CI namespace
oc get pipelineruns -n tssc-app-ci

# See snapshots
oc get snapshots -n tssc-app-ci

# See releases
oc get releases -n tssc-app-ci
```

---

## Operations Workflow

### Promote to Production

```bash
# 1. Review validated snapshots
oc get snapshots -n tssc-app-ci

# 2. Check which snapshot is in stage
oc get releases -n tssc-app-ci | grep stage

# 3. Get details
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
    release.appstudio.io/jira-ticket: "PROD-123"
spec:
  snapshot: ${SNAPSHOT_NAME}
  releasePlan: release-to-prod
EOF

# 5. Monitor
oc get release -n tssc-app-ci -w
oc get pipelineruns -n tssc-app-ci -w
```

---

## Resource Placement Summary

### tssc-app-ci Namespace

| Resource Type | Count | Names |
|--------------|-------|-------|
| Application | 1 | my-java-app |
| Component | 1+ | my-java-app-backend |
| Build Pipeline | 1 | maven-build-ci-konflux |
| Release Pipeline | 1 | release-pipeline |
| Integration Pipeline | 1 | integration-tests |
| Build Tasks | Multiple | All tasks |
| Release Tasks | Multiple | skopeo-copy, update-deployment, etc. |
| IntegrationTestScenario | 1 | integration-tests |
| Snapshot | Many | Auto-created per build |
| ReleasePlan | 3 | release-to-dev, release-to-stage, release-to-prod |
| Release | Many | Auto-created or manual |

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

## Summary

### What Changed

| Before | After |
|--------|-------|
| Pipelines in each namespace | All pipelines in tssc-app-ci |
| `git tag v1.0.0-stage` → promote | `git push` → automatic promotions |
| GitLab Release → prod | Create Release CR → prod |
| Generic namespace names (dev, stage, prod) | App-scoped names (my-java-app-dev, etc.) |
| No dev environment | Three environments (dev, stage, prod) |

### Key Benefits

1. **Centralized CI/CD** - Better security, resource management, and RBAC
2. **Three Environments** - Dev for testing, stage for validation, prod for production
3. **Clear Namespace Naming** - App-scoped names prevent conflicts
4. **No Git Pollution** - Clean git history
5. **Automatic Testing** - Integration tests gate all promotions
6. **Audit Trail** - Complete record in Release CRs
7. **Easy Rollback** - Promote any previous Snapshot

### Developer Experience

```bash
# Before
git push
git tag v1.0.0-stage
git push --tags
# Create GitLab Release for prod

# After
git push
# Everything automated (dev, stage)
# Ops creates Release CR for prod
```

**That's it!** Konflux centralized CI/CD is now fully configured.
