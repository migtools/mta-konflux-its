# MTA Multicloud Tier-0 Testing via Slack

This setup allows you to trigger MTA tier-0 UI tests on multiple cloud platforms directly from Slack.

## Supported Cloud Platforms

- **ARO** - Azure Red Hat OpenShift
- **ROSA** - Red Hat OpenShift Service on AWS
- **ROSA-STS** - ROSA with STS authentication
- **OCP-AWS** - OpenShift on AWS (SNO)

## Architecture

```
Slack Workflow → EventListener → PipelineRun → Cloud Cluster → Tests → Slack Notification
```

## What It Does

1. **Resolves Snapshot:** From NVR, snapshot name, or gets latest
2. **Provisions Cluster:** Fresh cluster on selected cloud platform via OCPCTL
3. **Deploys MTA Operator:** From Konflux FBC snapshot
4. **Runs Tier-0 Tests:** Cypress UI tests with @tier0 tag
5. **Cleans Up:** Destroys cluster
6. **Notifies Slack:** Test results

## Setup Instructions

### 1. Deploy EventListener

```bash
# Apply Tekton Triggers
oc apply -f tekton-triggers/multicloud/eventlistener.yaml

# Expose EventListener as Route
oc expose svc el-mta-multicloud-tier0-trigger -n art-mta-tenant

# Get Webhook URL
WEBHOOK_URL=$(oc get route el-mta-multicloud-tier0-trigger -n art-mta-tenant -o jsonpath='{.spec.host}')
echo "Webhook URL: https://$WEBHOOK_URL"
```

**Copy this URL** - you'll need it for Slack setup.

### 2. Create Slack Workflow

1. **Open Slack** → Click workspace name → **Tools** → **Workflow Builder**

2. **Create New Workflow**
   - Name: "MTA Multicloud Tier-0 Test"
   - Trigger: **Shortcut**

3. **Add Step: Send a Form**
   
   **Question 1:**
   - Question: "Select cluster type"
   - Answer type: Select from a list
   - Options:
     - `aro` (ARO - Azure Red Hat OpenShift)
     - `rosa` (ROSA - Red Hat OpenShift Service on AWS)
     - `rosa-sts` (ROSA-STS - ROSA with STS)
     - `ocp-aws` (OCP-AWS - OpenShift on AWS)
   - Default: `aro`
   - Required: Yes
   - Variable name: `cluster_type`
   
   **Question 2:**
   - Question: "NVR, Snapshot, or leave empty for latest"
   - Answer type: Short answer (text)
   - Required: No
   - Variable name: `nvr_or_snapshot`
   
   **Question 3:**
   - Question: "Application (usually fbc-mta-8-3)"
   - Answer type: Short answer (text)
   - Default value: `fbc-mta-8-3`
   - Variable name: `application`

4. **Add Step: Send a Webhook**
   - URL: `https://<WEBHOOK_URL>` (from step 1)
   - Method: POST
   - Body:
     ```json
     {
       "cluster_type": "{{cluster_type}}",
       "nvr_or_snapshot": "{{nvr_or_snapshot}}",
       "application": "{{application}}",
       "triggered_by": "{{workflow.user.name}}"
     }
     ```

5. **Add Step: Send a Message**
   - Channel: `#mta-notifications` (or your channel)
   - Message:
     ```
     ✅ Triggered multicloud tier-0 test
     
     Cluster: {{cluster_type}}
     Snapshot/NVR: {{nvr_or_snapshot}}
     Triggered by: {{workflow.user.name}}
     
     You'll receive a notification when tests complete (~1-3 hours depending on platform).
     ```

6. **Publish Workflow**

### 3. Test It!

**In Slack:**

1. Click ⚡ (shortcuts menu) or type `/shortcuts`
2. Select "**MTA Multicloud Tier-0 Test**"
3. Fill form:
   ```
   Select cluster type:
   ┌────────────────────────────────────────────┐
   │ aro                                       │  ← dropdown
   └────────────────────────────────────────────┘
   
   NVR, Snapshot, or leave empty for latest:
   ┌────────────────────────────────────────────┐
   │ mta-operator-container-8.3.2...           │
   └────────────────────────────────────────────┘
   
   Application:
   ┌────────────────────────────────────────────┐
   │ fbc-mta-8-3                               │
   └────────────────────────────────────────────┘
   
              [Submit]
   ```

4. **Pipeline runs!** Monitor in Konflux:
   ```
   https://konflux-ui.apps.kflux-ocp-p01.7ayg.p1.openshiftapps.com/ns/art-mta-tenant/pipelineruns
   ```

5. **Get notification** when complete (~2-3 hours)

## Usage Examples

### Example 1: Test Latest Snapshot on ARO

In Slack form:
- **Cluster Type:** `aro`
- **NVR/Snapshot:** `<leave empty>`
- **Application:** `fbc-mta-8-3`

### Example 2: Test Specific NVR on ROSA

In Slack form:
- **Cluster Type:** `rosa`
- **NVR/Snapshot:** `mta-operator-container-8.4.0.202610191234.p2.g14f9d24.assembly.stream.el9`
- **Application:** `fbc-mta-8-4`

### Example 3: Test Specific Snapshot on ROSA-STS

In Slack form:
- **Cluster Type:** `rosa-sts`
- **NVR/Snapshot:** `fbc-mta-8-3-20261019-123456-000`
- **Application:** `fbc-mta-8-3`

### Example 4: Test on OCP-AWS

In Slack form:
- **Cluster Type:** `ocp-aws`
- **NVR/Snapshot:** `<leave empty>`
- **Application:** `fbc-mta-8-3`

## Manual Trigger (Without Slack)

If you need to trigger manually:

```bash
# Get latest snapshot
SNAPSHOT=$(oc get snapshot -n art-mta-tenant \
  -l appstudio.openshift.io/application=fbc-mta-8-3 \
  --sort-by=.metadata.creationTimestamp \
  -o jsonpath='{.items[-1].metadata.name}')

# Trigger pipeline (specify cluster type)
tkn pipeline start mta-multicloud-tier0-pipeline \
  -n art-mta-tenant \
  --param clusterType=aro \
  --param nvrOrSnapshot="$SNAPSHOT" \
  --param application=fbc-mta-8-3 \
  --param triggeredBy="manual-cli" \
  --use-param-defaults \
  --showlog

# For ROSA:
# --param clusterType=rosa

# For ROSA-STS:
# --param clusterType=rosa-sts

# For OCP-AWS:
# --param clusterType=ocp-aws
```

## Pipeline Timeline

| Phase | Duration | What Happens |
|-------|----------|--------------|
| Resolve Snapshot | 30s | Finds snapshot from NVR/name/latest |
| Map Cluster Config | 5s | Maps cluster type to OCPCTL profile |
| Provision Cluster | 60-120 min | OCPCTL creates cluster (varies by platform) |
| Deploy Operator | 10-15 min | FBC catalog + operator install |
| Tier-0 Tests | 20-30 min | Cypress @tier0 tests |
| Cleanup | 5-10 min | Destroy cluster |
| **Total** | **~1.5-3 hours** | Varies by cluster type |

**Platform-specific provisioning times:**
- **OCP-AWS**: ~60 min (fastest, SNO)
- **ROSA/ROSA-STS**: ~90 min
- **ARO**: ~120 min (slowest, Azure provisioning)

## Troubleshooting

### Pipeline Failed at Provision

Check cluster provisioning logs:
```bash
oc logs <pipeline-run-pod> -c step-wait-for-ready
```

Common causes:
- ARO: Azure capacity issues in region
- ROSA/ROSA-STS: AWS quotas or STS permissions
- OCP-AWS: AWS instance limits

### Pipeline Failed at Tests

Check test logs:
```bash
oc logs <pipeline-run-pod> -c step-run-tests
```

### Slack Not Receiving Notification

Check webhook secret exists:
```bash
oc get secret slack-webhook-upgrade -n art-mta-tenant
```

## Future Enhancements

Possible additions:
- Multi-cloud matrix triggering (one Slack click → test on all platforms in parallel)
- FIPS-enabled cluster variants
- Custom OCP versions (currently fixed to 4.22)
- Different test tags beyond @tier0

## Files Created

```
tasks/multicloud/
  resolve-snapshot-from-nvr.yaml      # NVR → snapshot resolution
  map-cluster-profile.yaml            # Cluster type → OCPCTL profile mapping
  slack-notification-tier0.yaml       # Slack results notification

pipelines/multicloud/
  mta-tier0-pipeline.yaml             # Unified multicloud test pipeline

tekton-triggers/multicloud/
  eventlistener.yaml                  # Slack webhook → pipeline trigger
```
