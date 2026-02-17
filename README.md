# TSSC Sample Pipelines - Konflux Edition

This repository contains Tekton pipelines and tasks for building and deploying applications using **Red Hat Konflux** (Trusted Software Supply Chain).

## What's New: Konflux Support

This repository now includes full Konflux integration with centralized CI/CD!

### What is Konflux?

Konflux is Red Hat's trusted application pipeline that provides:
- 🔐 **Automatic image signing** (Tekton Chains)
- 📋 **SBOM generation and storage** (Trustification)
- 🛡️ **Policy enforcement** (Enterprise Contract)
- 🔄 **Automated promotions** (Snapshot/Release model)
- ✅ **Integration testing gates**
- 📊 **Complete audit trail**

---

## Repository Structure

```
tssc-sample-pipelines/
├── pipelines/
│   ├── maven-build-ci.yaml              # Original build pipeline
│   ├── promote-to-env.yaml              # Original promotion pipeline (git tag-based)
│   ├── maven-build-ci-konflux.yaml      # NEW: Konflux build pipeline
│   ├── release-pipeline.yaml            # NEW: Konflux release pipeline
│   └── integration-tests.yaml           # NEW: Konflux integration tests
├── tasks/
│   ├── acs-*.yaml                       # ACS/Stackrox security scans
│   ├── buildah-rhtap.yaml               # Container image build
│   ├── git-clone.yaml                   # Git repository clone
│   ├── maven.yaml                       # Maven package task
│   ├── skopeo-copy.yaml                 # Image copy (for promotions)
│   ├── update-deployment.yaml           # GitOps update
│   ├── verify-enterprise-contract.yaml  # Policy verification
│   ├── upload-sbom-to-trustification.yaml
│   └── ... (other tasks)
└── konflux-resources/
    ├── 00-SETUP-GUIDE.md                # Complete setup instructions
    ├── application.yaml                 # Konflux Application resource
    ├── component.yaml                   # Konflux Component resource
    ├── integration-test-scenario.yaml   # Integration test configuration
    ├── release-plans.yaml               # Release plans (dev, stage, prod)
    ├── release-plan-admission-dev.yaml  # Dev admission policy
    ├── release-plan-admission-stage.yaml # Stage admission policy
    └── release-plan-admission-prod.yaml  # Prod admission policy
```

---

## Quick Start: Konflux Setup

### Prerequisites

- OpenShift 4.12+ cluster with admin access
- Konflux operator installed
- Application name chosen (e.g., `my-java-app`)
- Git repository
- Container image registry (quay.io, etc.)

### 5-Minute Setup

```bash
# 1. Set your application name
export APP_NAME="my-java-app"

# 2. Create namespaces
oc create namespace tssc-app-ci
oc create namespace ${APP_NAME}-dev
oc create namespace ${APP_NAME}-stage
oc create namespace ${APP_NAME}-prod

# 3. Setup RBAC
oc create serviceaccount release-service-account -n tssc-app-ci
oc create rolebinding release-deployer \
  --serviceaccount=tssc-app-ci:release-service-account \
  --clusterrole=edit -n ${APP_NAME}-dev
oc create rolebinding release-deployer \
  --serviceaccount=tssc-app-ci:release-service-account \
  --clusterrole=edit -n ${APP_NAME}-stage
oc create rolebinding release-deployer \
  --serviceaccount=tssc-app-ci:release-service-account \
  --clusterrole=edit -n ${APP_NAME}-prod

# 4. Install pipelines and tasks
oc apply -f tasks/ -n tssc-app-ci
oc apply -f pipelines/maven-build-ci-konflux.yaml -n tssc-app-ci
oc apply -f pipelines/release-pipeline.yaml -n tssc-app-ci
oc apply -f pipelines/integration-tests.yaml -n tssc-app-ci

# 5. Edit and apply Konflux resources
# Edit konflux-resources/*.yaml files with your app name, git repo, and image repo
oc apply -f konflux-resources/application.yaml -n tssc-app-ci
oc apply -f konflux-resources/component.yaml -n tssc-app-ci
oc apply -f konflux-resources/integration-test-scenario.yaml -n tssc-app-ci
oc apply -f konflux-resources/release-plans.yaml -n tssc-app-ci
oc apply -f konflux-resources/release-plan-admission-dev.yaml -n ${APP_NAME}-dev
oc apply -f konflux-resources/release-plan-admission-stage.yaml -n ${APP_NAME}-stage
oc apply -f konflux-resources/release-plan-admission-prod.yaml -n ${APP_NAME}-prod
```

**For detailed instructions, see: [konflux-resources/00-SETUP-GUIDE.md](konflux-resources/00-SETUP-GUIDE.md)**

---

## Architecture: Centralized CI/CD Pattern

```
┌─────────────────────────────────────────┐
│ tssc-app-ci (CI/CD Namespace)           │
│                                          │
│ ALL pipelines run here:                 │
│  • maven-build-ci-konflux               │
│  • release-pipeline                     │
│  • integration-tests                    │
│                                          │
│ Konflux Resources:                      │
│  • Application                           │
│  • Component                             │
│  • Snapshots                             │
│  • ReleasePlans (dev, stage, prod)      │
│  • IntegrationTestScenario              │
│  • Release CRs                           │
└─────────────────────────────────────────┘
                  ↓
    ┌─────────────┼─────────────┐
    ↓             ↓             ↓
┌─────────┐ ┌──────────┐ ┌──────────┐
│ *-dev   │ │ *-stage  │ │ *-prod   │
│ Runtime │ │ Runtime  │ │ Runtime  │
│ ONLY    │ │ ONLY     │ │ ONLY     │
└─────────┘ └──────────┘ └──────────┘
```

### Key Benefits

1. **Centralized CI/CD** - All pipelines in one namespace
2. **Three Environments** - Dev → Stage → Prod
3. **Automatic Promotions** - Dev and Stage auto-release
4. **Manual Production** - Requires approval
5. **Policy Enforcement** - Different policies per environment
6. **Complete Audit Trail** - Every promotion tracked

---

## How It Works

### Build Flow

```
Developer pushes code
  ↓
Build pipeline runs in tssc-app-ci
  ↓
Image tagged with commit SHA
  ↓
Konflux creates Snapshot (immutable)
  ↓
Integration tests run
  ↓
If tests pass → Snapshot validated
```

### Release Flow

```
Validated Snapshot
  ↓
Auto-Release to Dev
  - Enterprise Contract: relaxed policy
  - Image tagged: dev
  - Deployed to my-java-app-dev
  ↓
Auto-Release to Stage
  - Enterprise Contract: standard policy
  - Image tagged: stage
  - Deployed to my-java-app-stage
  ↓
Manual Release to Prod (when approved)
  - Enterprise Contract: STRICT policy
  - Image tagged: prod
  - Deployed to my-java-app-prod
```

---

## Differences: Old vs New Pipelines

### Old Approach (Git Tag-Based)

```bash
# Developer workflow
git push                    # Build and deploy to dev
git tag v1.0.0-stage       # Promote to stage
git push --tags
# Create GitLab Release     # Promote to prod

# Issues:
# - Git history polluted with deployment tags
# - No policy enforcement
# - Manual promotion required for everything
# - No integration test gates
# - Limited audit trail
```

### New Approach (Konflux)

```bash
# Developer workflow
git push                    # Build → Test → Dev → Stage (all automatic!)

# Ops workflow (for prod only)
oc apply -f release-to-prod.yaml

# Benefits:
# - Clean git history
# - Automatic promotions
# - Policy enforcement per environment
# - Integration tests gate releases
# - Complete audit trail
# - Image signing and verification
```

---

## Pipelines

### 1. maven-build-ci-konflux.yaml

**Purpose:** Build container images from Maven projects

**Runs in:** `tssc-app-ci` namespace

**Triggered by:** Git push to main branch

**What it does:**
1. Clone repository
2. Build Maven project (`mvn package`)
3. Build container image (Buildah)
4. Upload SBOMs to Trustification
5. Run ACS security scans
6. Produce results (IMAGE_URL, IMAGE_DIGEST)

**Results:**
- Image pushed to registry with commit SHA tag
- Tekton Chains automatically signs image
- SBOMs stored in Trustification
- Results feed into Snapshot creation

**Key Change from Original:**
- ❌ Removed `update-deployment` task (Konflux handles this)
- ❌ Removed `acs-deploy-check` task
- ✅ Added Konflux discovery labels

---

### 2. release-pipeline.yaml

**Purpose:** Promote images to environments

**Runs in:** `tssc-app-ci` namespace

**Triggered by:** Konflux (automatic or manual Release CR)

**What it does:**
1. Extract image info from Snapshot
2. Verify Enterprise Contract policies
3. Determine target environment
4. Copy image to environment-specific tag
5. Update GitOps repository

**Example:**
```
Snapshot: quay.io/myorg/myapp@sha256:abc123...
Target: dev
Action: Copy to quay.io/myorg/myapp:dev
```

---

### 3. integration-tests.yaml

**Purpose:** Test newly built images

**Runs in:** `tssc-app-ci` namespace

**Triggered by:** Konflux after Snapshot creation

**What it does:**
1. Parse Snapshot to get image URL
2. Run health checks
3. Run integration tests
4. Report results

**Customize this pipeline for your application!**

---

## Enterprise Contract Policies

Different environments have different policies:

### Dev Environment
```yaml
policy: "slsa3"
strict: "false"  # Warn only, don't block
```
- ⚠️ CVEs logged as warnings
- ⚠️ Policy violations logged
- ✅ Promotion continues

### Stage Environment
```yaml
policy: "slsa3"
strict: "true"  # Block on violations
```
- ❌ High/Critical CVEs block promotion
- ❌ Policy violations block promotion
- ✅ Must pass to promote

### Prod Environment
```yaml
policy: "slsa3-strict"
strict: "true"  # Strictest enforcement
```
- ❌ Any CVEs block promotion
- ❌ All compliance checks must pass
- ❌ All tests must pass
- ✅ Zero tolerance

---

## Image Tagging Strategy

### Build Phase
```
quay.io/myorg/myapp:a1b2c3d4  ← Git commit SHA
                    @sha256:abc123...  ← Immutable digest
```

### Release Phase
```
quay.io/myorg/myapp:dev    → sha256:abc123...
quay.io/myorg/myapp:stage  → sha256:abc123...
quay.io/myorg/myapp:prod   → sha256:def456...
```

**Konflux tracks by digest, deploys by environment tag.**

---

## Developer Workflow

### Daily Development

```bash
# 1. Make changes
cd my-java-app
vim src/main/java/MyService.java

# 2. Commit and push
git add .
git commit -m "Add new feature"
git push origin main

# That's it! Everything else is automatic:
# → Build runs
# → Image created with commit SHA tag
# → Snapshot created
# → Integration tests run
# → Auto-release to dev
# → Auto-release to stage (if dev succeeds)
```

### Check Status

```bash
# See builds
oc get pipelineruns -n tssc-app-ci

# See snapshots
oc get snapshots -n tssc-app-ci

# See releases
oc get releases -n tssc-app-ci

# Check what's deployed
oc get pods -n my-java-app-dev
oc get pods -n my-java-app-stage
oc get pods -n my-java-app-prod
```

---

## Operations Workflow

### Promote to Production

```bash
# 1. List validated snapshots
oc get snapshots -n tssc-app-ci

# 2. Choose snapshot currently in stage
SNAPSHOT="my-java-app-snapshot-xyz123"

# 3. Create production release
cat <<EOF | oc apply -f -
apiVersion: appstudio.redhat.com/v1alpha1
kind: Release
metadata:
  name: prod-release-$(date +%Y%m%d-%H%M)
  namespace: tssc-app-ci
  annotations:
    release.appstudio.io/approved-by: "ops@example.com"
    release.appstudio.io/jira-ticket: "PROD-12345"
spec:
  snapshot: ${SNAPSHOT}
  releasePlan: release-to-prod
EOF

# 4. Monitor
oc get releases -n tssc-app-ci -w
oc get pipelineruns -n tssc-app-ci -w
```

---

## Customization

### Integration Tests

Edit `pipelines/integration-tests.yaml` to add your tests:

```yaml
- name: run-tests
  taskSpec:
    steps:
      - name: health-check
        script: |
          curl -f http://localhost:8080/health

      - name: api-tests
        script: |
          ./run-api-tests.sh

      - name: database-check
        script: |
          psql -c "SELECT 1"
```

### Enterprise Contract Policies

Customize policies in ReleasePlanAdmissions:

```yaml
# More relaxed dev
data:
  enterpriseContract:
    policy: "slsa2"  # Lighter policy
    strict: "false"

# Custom policy URL
data:
  enterpriseContract:
    policy: "oci://quay.io/myorg/my-policy:latest"
    strict: "true"
```

---

## Troubleshooting

### Common Issues

**Build doesn't create Snapshot**
- Check build pipeline has IMAGE_URL and IMAGE_DIGEST results
- Verify PipelineRun has correct labels
- Check Component exists with matching name

**Integration tests don't run**
- Verify IntegrationTestScenario exists
- Check it references correct application
- Look at integration-service logs

**Release fails with Enterprise Contract error**
- Review pipeline logs for policy violations
- Check CVE scan results: `oc get pipelinerun <name> -o yaml`
- Verify image signatures exist

**Cross-namespace permissions error**
- Verify RoleBindings exist in target namespaces
- Check service account has correct permissions
- Review release-service-account secrets

---

## Additional Resources

- **Konflux Documentation:** https://konflux-ci.dev/
- **Enterprise Contract:** https://enterprisecontract.dev/
- **Tekton Pipelines:** https://tekton.dev/
- **SLSA Framework:** https://slsa.dev/
- **Tekton Chains:** https://tekton.dev/docs/chains/

---

## Migration from Old Pipelines

If you're currently using the git tag-based pipelines:

1. ✅ Keep old pipelines running during transition
2. ✅ Set up Konflux in parallel (new namespaces)
3. ✅ Test thoroughly with non-production builds
4. ✅ Gradually migrate applications
5. ✅ Disable old tag-based triggers once stable
6. ✅ Archive old pipelines for reference

**The old pipelines are still in this repo** (`maven-build-ci.yaml`, `promote-to-env.yaml`) for reference.

---

## Contributing

Improvements welcome! Please:
1. Test changes thoroughly
2. Update documentation
3. Follow existing patterns
4. Include examples

---

## License

[Include your license here]

---

## Support

For issues or questions:
- Open an issue in this repository
- Check Konflux documentation
- Review Enterprise Contract docs

---

**Happy building with Konflux! 🚀**
