# MTA Multicloud Tier-0 Testing via Slack

This setup allows you to trigger MTA tier-0 UI tests on ALL cloud platforms with a single Slack command.

## Supported Cloud Platforms

One webhook triggers parallel tests on:
- **ARO** - Azure Red Hat OpenShift
- **ROSA** - Red Hat OpenShift Service on AWS
- **ROSA-STS** - ROSA with STS authentication
- **OCP-AWS** - OpenShift on AWS (SNO)

## Architecture

```
Slack Workflow → EventListener → 4 Parallel PipelineRuns → 4 Clusters → Tests → Slack Notifications
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
   
   **Question:**
   - Question: "NVR or leave empty for latest (tests run on ALL platforms)"
   - Answer type: Short answer (text)
   - Required: No
   - Variable name: `nvr_or_snapshot`

4. **Add Step: Send a Webhook**
   - URL: `https://<WEBHOOK_URL>` (from step 1)
   - Method: POST
   - Body:
     ```json
     {
       "nvr_or_snapshot": "{{nvr_or_snapshot}}",
       "triggered_by": "{{workflow.user.name}}"
     }
     ```

5. **Add Step: Send a Message**
   - Channel: `#mta-notifications` (or your channel)
   - Message:
     ```
     ✅ Triggered tier-0 tests on ALL platforms (ARO, ROSA, ROSA-STS, OCP-AWS)
     
     NVR: {{nvr_or_snapshot}}
     Triggered by: {{workflow.user.name}}
     
     4 parallel test runs started. You'll receive notifications as each completes (~1-3 hours per platform).
     ```

6. **Publish Workflow**

### 3. Test It!

**In Slack:**

1. Click ⚡ (shortcuts menu) or type `/shortcuts`
2. Select "**MTA Multicloud Tier-0 Test**"
3. Fill form:
   ```
   NVR or leave empty for latest:
   ┌────────────────────────────────────────────┐
   │ mta-operator-container-8.4.0...           │
   └────────────────────────────────────────────┘
   
              [Submit]
   ```

4. **4 pipelines run in parallel!** Monitor in Konflux:
   ```
   https://konflux-ui.apps.kflux-ocp-p01.7ayg.p1.openshiftapps.com/ns/art-mta-tenant/pipelineruns
   ```
   You'll see:
   - `mta-tier0-aro-xxxxx`
   - `mta-tier0-rosa-xxxxx`
   - `mta-tier0-rosa-sts-xxxxx`
   - `mta-tier0-ocp-aws-xxxxx`

5. **Get 4 separate notifications** as each platform completes (one per platform)

## Usage Examples

### Example 1: Test Latest Snapshot on All Platforms

In Slack form:
- **NVR:** `<leave empty>`

→ Triggers 4 parallel tests using latest snapshot from default app `fbc-mta-8-3`

**Result:** 4 Slack notifications (one per platform):
```
✅ ARO (Azure Red Hat OpenShift) - PASSED
✅ ROSA (Red Hat OpenShift Service on AWS) - PASSED  
✅ ROSA-STS (ROSA with STS) - PASSED
✅ OCP-AWS (OpenShift on AWS) - PASSED
```

### Example 2: Test Specific NVR on All Platforms

In Slack form:
- **NVR:** `mta-operator-container-8.4.0.202610191234.p2.g14f9d24.assembly.stream.el9`

→ Triggers 4 parallel tests, auto-derives app `fbc-mta-8-4` from NVR

**Result:** 4 Slack notifications showing status for each platform

## Manual Trigger (Without Slack)

If you need to trigger tests on a **single** platform manually:

```bash
# Test specific NVR on ARO only
tkn pipeline start mta-multicloud-tier0-pipeline \
  -n art-mta-tenant \
  --param clusterType=aro \
  --param nvrOrSnapshot="mta-operator-container-8.4.0.202610191234.p2.g14f9d24.assembly.stream.el9" \
  --param triggeredBy="manual-cli" \
  --use-param-defaults \
  --showlog

# Test on different platform
tkn pipeline start mta-multicloud-tier0-pipeline \
  -n art-mta-tenant \
  --param clusterType=rosa \
  --param nvrOrSnapshot="" \
  --param triggeredBy="manual-cli" \
  --use-param-defaults \
  --showlog
```

**Note:** Manual trigger runs one platform at a time. Use Slack workflow to trigger all 4 platforms with one command.

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
