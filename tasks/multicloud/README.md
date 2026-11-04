# MTA Multicloud Deployment Tasks

This directory contains tasks for deploying MTA operator on managed multicloud platforms (ROSA, ARO) with stage builds from Konflux.

## Overview

Managed OpenShift platforms like ROSA (Red Hat OpenShift Service on AWS) and ARO (Azure Red Hat OpenShift) require special handling when deploying operators from **stage registry builds**. This is due to registry mirroring requirements that don't exist on standard OCP clusters.

## The Problem

### Why Standard Deployment Fails on ROSA/ARO

When deploying MTA operator from Konflux stage builds on ROSA/ARO clusters, OLM fails during the **bundle unpacking phase** with the error:

```
bundle unpacking failed. Reason: DeadlineExceeded
Message: Job was active longer than specified deadline
```

**Root Cause:**

1. **FBC catalog references production registry**
   - Konflux builds generate FBC catalogs that reference `registry.redhat.io` (production)
   - Example: `registry.redhat.io/mta/mta-operator-bundle@sha256:...`

2. **Stage images are in stage registry**
   - Actual operator images are published to `registry.stage.redhat.io`
   - Production registry doesn't have these images (they're pre-release)

3. **OLM can't pull bundle images**
   - OLM tries to pull `registry.redhat.io/mta/mta-operator-bundle`
   - Image doesn't exist at that location
   - Pull fails → bundle unpack job times out
   - OLM caches this failure → subscription stuck

4. **Pull secret alone is insufficient**
   - Adding `registry.stage.redhat.io` credentials to the global pull secret allows authentication
   - But doesn't redirect image pulls from `registry.redhat.io` → `registry.stage.redhat.io`
   - FBC catalog still references `registry.redhat.io`, so OLM pulls from the wrong registry

### Why This Only Affects Managed Clusters

**Standard OCP clusters:** Can use `ImageDigestMirrorSet` (IDMS) freely

**ROSA/ARO clusters:** Have validation webhooks that block IDMS creation:
- `sre-imagecontentpolicies-validation` - Blocks ImageContentSourcePolicy/ImageDigestMirrorSet
- `sre-regular-user-validation` - Additional SRE controls

These webhooks must be deleted before IDMS can be created.

## The Solution

### ImageDigestMirrorSet (IDMS)

**IDMS redirects registry traffic at the node level:**

```yaml
apiVersion: config.openshift.io/v1
kind: ImageDigestMirrorSet
metadata:
  name: stage-images-mirror-set
spec:
  imageDigestMirrors:
  - mirrors:
    - registry.stage.redhat.io/mta/mta-operator-bundle
    source: registry.redhat.io/mta/mta-operator-bundle
  - mirrors:
    - registry.stage.redhat.io/mta/mta-rhel9-operator
    source: registry.redhat.io/mta/mta-rhel9-operator
  # ... (7 total MTA image mappings)
```

**How it works:**
1. When OLM requests `registry.redhat.io/mta/mta-operator-bundle`
2. CRI-O checks IDMS configuration on the node
3. Redirects pull to `registry.stage.redhat.io/mta/mta-operator-bundle`
4. Image pull succeeds (using stage registry credentials from pull secret)
5. OLM unpacks bundle successfully
6. InstallPlan is created

### Critical Timing Requirements

**IDMS must be created BEFORE CatalogSource**, otherwise:

1. ❌ CatalogSource created → OLM starts bundle unpack
2. ❌ Bundle images pull from `registry.redhat.io` (wrong registry)
3. ❌ Pull fails → bundle unpack job times out
4. ❌ OLM caches this failure in subscription status
5. ⚠️ Even if IDMS is created later, OLM won't retry automatically
6. ⚠️ Requires CatalogSource deletion + catalog-operator restart to clear cache

**Correct order:**

1. ✅ Delete ROSA validation webhooks
2. ✅ Pause MachineConfigPools
3. ✅ Update global pull secret (add stage registry credentials)
4. ✅ Create ImageDigestMirrorSet
5. ✅ Resume MachineConfigPools
6. ✅ **Wait for MCPs to finish updating** (15-20 minutes)
   - MCPs must apply new machine-config with IDMS rules to all nodes
   - `oc wait mcp master worker --for condition=updated --timeout=60m`
7. ✅ Create CatalogSource (now bundle pulls will succeed)
8. ✅ Create Subscription → OLM unpacks bundles → InstallPlan created → Operator installs

## MachineConfigPool Update Process

When IDMS is created, it triggers a MachineConfigPool update:

**What happens:**
1. Machine Config Operator generates new `rendered-master-*` and `rendered-worker-*` configs
2. MCP enters `UPDATING` state
3. Nodes are updated **one by one** (rolling update):
   - Node is cordoned
   - Workloads are drained
   - New machine-config is applied (includes IDMS registry mirrors)
   - CRI-O is restarted with new registry configuration
   - Node is uncordoned
4. MCP enters `UPDATED` state when all nodes complete

**Timing:**
- **ROSA**: ~15-20 minutes for 3 masters + 5 workers
- **ARO**: ~20-25 minutes (similar node counts)
- **Larger clusters**: Longer (scales with node count)

**Why we can't skip the wait:**
- If CatalogSource is created before MCPs finish updating:
  - Some nodes still have old CRI-O config (no IDMS mirrors)
  - CatalogSource pod might land on a node without IDMS
  - Bundle pulls fail → cached failure state

## Debugging Failed Deployments

### Symptom: Subscription stuck, no InstallPlan

**Check OLM catalog-operator logs:**
```bash
oc logs -n openshift-operator-lifecycle-manager deployment/catalog-operator | grep -i "mta\|unpacking"
```

**Look for:**
```
bundle unpacking failed. Reason: DeadlineExceeded
```

### Symptom: CatalogSource flipping READY/TRANSIENT_FAILURE

**Check CatalogSource pod:**
```bash
POD=$(oc get pods -n openshift-marketplace -l olm.catalogSource=mta-konflux-catalog -o jsonpath='{.items[0].metadata.name}')
oc logs -n openshift-marketplace $POD
```

**Check image pull status:**
```bash
oc get pod $POD -n openshift-marketplace -o jsonpath='{.status.containerStatuses[0].state}'
```

### Symptom: MCPs stuck in UPDATING state

**Check MCP status:**
```bash
oc get mcp -o wide
```

**Check node status:**
```bash
oc get nodes
oc describe node <node-name> | grep -A 20 "Conditions"
```

**Check machine-config-daemon logs:**
```bash
oc logs -n openshift-machine-config-operator ds/machine-config-daemon --tail=100
```

### Recovery: Clear OLM Cached State

If deployment failed before IDMS was ready:

```bash
# Delete failed resources
oc delete subscription mta-operator -n openshift-mta
oc delete catalogsource mta-konflux-catalog -n openshift-marketplace

# Wait for cleanup
sleep 10

# Recreate CatalogSource (IDMS should be ready now)
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: mta-konflux-catalog
  namespace: openshift-marketplace
spec:
  sourceType: grpc
  image: <FBC_IMAGE>
  displayName: MTA Konflux Catalog
  publisher: MTA Engineering
EOF

# Recreate Subscription
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: mta-operator
  namespace: openshift-mta
spec:
  channel: stable-v8.3
  installPlanApproval: Automatic
  name: mta-operator
  source: mta-konflux-catalog
  sourceNamespace: openshift-marketplace
EOF
```

## Verification Checklist

After deployment completes:

**1. Check IDMS is applied:**
```bash
oc get imagedigestmirrorset stage-images-mirror-set
oc get imagedigestmirrorset stage-images-mirror-set -o yaml | grep -A 20 "spec:"
```

**2. Check MCPs are updated:**
```bash
oc get mcp
# Should show: UPDATED=True, UPDATING=False, DEGRADED=False
```

**3. Check CatalogSource is READY:**
```bash
oc get catalogsource mta-konflux-catalog -n openshift-marketplace
oc get catalogsource mta-konflux-catalog -n openshift-marketplace -o jsonpath='{.status.connectionState.lastObservedState}'
# Should output: READY
```

**4. Check InstallPlan was created:**
```bash
oc get installplan -n openshift-mta
# Should show phase: Complete
```

**5. Check CSV installed:**
```bash
oc get csv -n openshift-mta
# Should show phase: Succeeded
```

**6. Check operator running:**
```bash
oc get deployment mta-operator -n openshift-mta
# Should show: READY=1/1, AVAILABLE=1
```

**7. Check Tackle CR deployed:**
```bash
oc get tackle mta -n openshift-mta
oc get pods -n openshift-mta
# Should show: mta-hub, mta-ui pods running
```

## Related Files

- **Task:** `tasks/multicloud/deploy-mta-operator-multicloud.yaml`
- **Pipeline:** `pipelines/multicloud/mta-tier0-pipeline.yaml`
- **Cluster Provisioning:** `tasks/ocpctl-provision.yaml`
- **Cluster Profiles:** `tasks/multicloud/map-cluster-profile.yaml`

## Jenkins Comparison

This implementation is based on the Jenkins deployment workflow from `migrationqe-automation`:

**Jenkins source:** `/home/sshveta/Work/mta/mta-qe-automation/mta/deploy_mta.sh` (lines 69-96)

**Key differences:**
- Jenkins: Bash script, sequential execution
- Tekton: Declarative pipeline, parallel task execution
- Jenkins: Manual IDMS template in `templates/idms.sh`
- Tekton: IDMS embedded in task YAML (more portable)
- Both: Same webhook deletion + MCP management approach

## References

- [OpenShift IDMS Documentation](https://docs.openshift.com/container-platform/latest/openshift_images/image-configuration.html#images-configuration-registry-mirror_image-configuration)
- [OLM Bundle Unpacking](https://olm.operatorframework.io/docs/advanced-tasks/adding-admission-and-conversion-webhooks/)
- [ROSA Cluster Configuration](https://docs.openshift.com/rosa/rosa_architecture/rosa_policy_service_definition/rosa-policy-responsibility-matrix.html)
- [MachineConfig Overview](https://docs.openshift.com/container-platform/latest/post_installation_configuration/machine-configuration-tasks.html)
