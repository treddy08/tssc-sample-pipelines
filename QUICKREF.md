# Konflux Quick Reference

## Common Commands

### Check Build Status
```bash
# Watch builds
oc get pipelineruns -n tssc-app-ci -w

# Get latest build
oc get pipelineruns -n tssc-app-ci --sort-by=.metadata.creationTimestamp | tail -n 1

# View build logs
oc logs -f pipelinerun/<name> -n tssc-app-ci
```

### Check Snapshots
```bash
# List snapshots
oc get snapshots -n tssc-app-ci

# Get latest snapshot
oc get snapshots -n tssc-app-ci --sort-by=.metadata.creationTimestamp | tail -n 1

# View snapshot details
oc describe snapshot <name> -n tssc-app-ci
```

### Check Releases
```bash
# List all releases
oc get releases -n tssc-app-ci

# Filter by environment
oc get releases -n tssc-app-ci | grep dev
oc get releases -n tssc-app-ci | grep stage
oc get releases -n tssc-app-ci | grep prod

# Watch releases
oc get releases -n tssc-app-ci -w
```

### Check Deployments
```bash
# Dev environment
oc get pods -n my-java-app-dev
oc get deployment -n my-java-app-dev

# Stage environment
oc get pods -n my-java-app-stage

# Prod environment
oc get pods -n my-java-app-prod
```

### Manual Production Release
```bash
# List snapshots
oc get snapshots -n tssc-app-ci

# Create release
SNAPSHOT="my-java-app-snapshot-xyz123"
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
  snapshot: ${SNAPSHOT}
  releasePlan: release-to-prod
EOF
```

---

## Resource Locations

| Resource | Namespace | Count |
|----------|-----------|-------|
| Application | tssc-app-ci | 1 |
| Component | tssc-app-ci | 1+ |
| Build Pipeline | tssc-app-ci | 1 |
| Release Pipeline | tssc-app-ci | 1 |
| Integration Pipeline | tssc-app-ci | 1 |
| Snapshot | tssc-app-ci | Many |
| ReleasePlan | tssc-app-ci | 3 |
| Release | tssc-app-ci | Many |
| ReleasePlanAdmission | my-app-dev | 1 |
| ReleasePlanAdmission | my-app-stage | 1 |
| ReleasePlanAdmission | my-app-prod | 1 |

---

## Troubleshooting

### Build Fails
```bash
# View build logs
oc logs -f pipelinerun/<name> -n tssc-app-ci

# Check specific task
oc logs -f pipelinerun/<name> -c step-<taskname> -n tssc-app-ci

# View all events
oc get events -n tssc-app-ci --sort-by='.lastTimestamp'
```

### No Snapshot Created
```bash
# Check if build completed
oc get pipelinerun <name> -n tssc-app-ci -o yaml | grep "status:"

# Check build results
oc get pipelinerun <name> -n tssc-app-ci -o yaml | grep -A 10 "pipelineResults:"

# Check Component exists
oc get component -n tssc-app-ci

# Check Component matches PipelineRun labels
oc get pipelinerun <name> -n tssc-app-ci -o yaml | grep "appstudio.openshift.io"
```

### Integration Tests Don't Run
```bash
# Check IntegrationTestScenario
oc get integrationtestscenario -n tssc-app-ci

# View scenario details
oc describe integrationtestscenario integration-tests -n tssc-app-ci

# Check integration-service logs
oc logs -n konflux-system deployment/integration-service-controller-manager
```

### Release Fails
```bash
# View release details
oc describe release <name> -n tssc-app-ci

# Check release pipeline logs
oc logs -f pipelinerun/<release-pipelinerun> -n tssc-app-ci

# Check ReleasePlanAdmission in target namespace
oc get releaseplanadmission -n my-app-dev

# Check RBAC permissions
oc get rolebindings -n my-app-dev | grep release
```

### Enterprise Contract Fails
```bash
# View EC task logs
oc logs pipelinerun/<name> -c step-verify-enterprise-contract -n tssc-app-ci

# Common failures:
# - CVEs found: Update base image
# - Signature verification failed: Check Tekton Chains
# - SBOM missing: Check buildah-rhtap task
# - Policy violation: Review policy in ReleasePlanAdmission
```

---

## Enterprise Contract Policies

### Dev (Relaxed)
```yaml
policy: "github.com/enterprise-contract/config//slsa3"
strict: "false"
```
- Warnings only
- CVEs logged but allowed
- Fast iteration

### Stage (Standard)
```yaml
policy: "github.com/enterprise-contract/config//slsa3"
strict: "true"
```
- Blocks high/critical CVEs
- Enforces policy
- Pre-production gate

### Prod (Strict)
```yaml
policy: "github.com/enterprise-contract/config//slsa3-strict"
strict: "true"
```
- Zero CVEs
- All checks must pass
- Production safety

---

## Image Tags

### After Build
```
quay.io/myorg/myapp:a1b2c3d4    ← Git commit SHA
                   @sha256:abc123...  ← Digest (immutable)
```

### After Releases
```
quay.io/myorg/myapp:dev    → sha256:abc123...
quay.io/myorg/myapp:stage  → sha256:abc123...
quay.io/myorg/myapp:prod   → sha256:def456...
```

---

## Environment Variables

### For Setup Scripts
```bash
export APP_NAME="my-java-app"
export COMPONENT_NAME="${APP_NAME}-backend"
export GIT_URL="https://gitlab.example.com/myorg/my-java-app"
export IMAGE_REPO="quay.io/myorg/my-java-app-backend"
```

---

## Useful Queries

### What's deployed where?
```bash
# Dev
oc get deployment -n my-java-app-dev -o jsonpath='{.items[0].spec.template.spec.containers[0].image}'

# Stage
oc get deployment -n my-java-app-stage -o jsonpath='{.items[0].spec.template.spec.containers[0].image}'

# Prod
oc get deployment -n my-java-app-prod -o jsonpath='{.items[0].spec.template.spec.containers[0].image}'
```

### Latest successful build
```bash
oc get pipelineruns -n tssc-app-ci \
  -l appstudio.openshift.io/component=my-java-app-backend \
  -o jsonpath='{range .items[?(@.status.conditions[0].status=="True")]}{.metadata.name}{"\t"}{.status.completionTime}{"\n"}{end}' \
  | sort -k2 | tail -1
```

### Snapshots by status
```bash
# Passed
oc get snapshots -n tssc-app-ci -l test.appstudio.openshift.io/status=Passed

# Failed
oc get snapshots -n tssc-app-ci -l test.appstudio.openshift.io/status=Failed

# Pending
oc get snapshots -n tssc-app-ci -l test.appstudio.openshift.io/status=Pending
```

---

## Delete/Cleanup

### Delete a Release
```bash
oc delete release <name> -n tssc-app-ci
```

### Delete a Snapshot
```bash
oc delete snapshot <name> -n tssc-app-ci
```

### Clean Old PipelineRuns
```bash
# Delete completed runs older than 7 days
oc get pipelineruns -n tssc-app-ci \
  -o json | jq -r '.items[] | select(.status.completionTime < (now - 604800 | strftime("%Y-%m-%dT%H:%M:%SZ"))) | .metadata.name' \
  | xargs oc delete pipelinerun -n tssc-app-ci
```

---

## Quick Setup Checklist

- [ ] Konflux operator installed
- [ ] Namespaces created (tssc-app-ci, *-dev, *-stage, *-prod)
- [ ] RBAC configured (service account + rolebindings)
- [ ] Pipelines installed (build, release, integration)
- [ ] Tasks installed
- [ ] Application CR created
- [ ] Component CR created
- [ ] IntegrationTestScenario created
- [ ] ReleasePlans created (3)
- [ ] ReleasePlanAdmissions created (3)
- [ ] Git trigger updated

---

## URLs to Bookmark

- Konflux Docs: https://konflux-ci.dev/
- Enterprise Contract: https://enterprisecontract.dev/
- SLSA: https://slsa.dev/
- Tekton: https://tekton.dev/
- Tekton Chains: https://tekton.dev/docs/chains/
