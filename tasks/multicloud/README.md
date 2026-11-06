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

