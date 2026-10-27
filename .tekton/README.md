# MTA Tier-0 Multicloud Testing - Pipelines-as-Code Configuration

This directory contains Pipelines-as-Code (PaC) configurations for triggering MTA tier-0 UI tests on multiple cloud platforms via GitHub issue comments.

## What This Does

Enables on-demand tier-0 testing across all cloud platforms (ARO, ROSA, ROSA-STS, OCP-AWS) by commenting `/test-tier0` on any GitHub issue in this repository.

## How It Works

```
GitHub Issue Comment → PaC → 4 Parallel PipelineRuns → Provision Clusters → Deploy MTA → Run Tests → Notify
```

1. **User triggers:** Comment `/test-tier0` on any GitHub issue or PR
2. **PaC detects:** The `konflux-kflux-ocp-p01` GitHub App sends webhook to Konflux
3. **4 PipelineRuns created:** One for each cloud platform (ARO, ROSA, ROSA-STS, OCP-AWS)
4. **Each pipeline:**
   - Provisions a fresh cluster via OCPCTL (using platform-specific profile)
   - Deploys MTA operator from latest Konflux snapshot
   - Runs Cypress tier-0 UI tests (@tier0 tag)
   - Destroys the cluster
   - Sends Slack notification with results

## Files

- **tier0-aro.yaml** - PaC config for ARO (Azure Red Hat OpenShift) tests
- **tier0-rosa.yaml** - PaC config for ROSA (Red Hat OpenShift on AWS) tests
- **tier0-rosa-sts.yaml** - PaC config for ROSA-STS (ROSA with STS) tests
- **tier0-ocp-aws.yaml** - PaC config for OCP-AWS (OpenShift on AWS SNO) tests

## Usage

### Trigger Tests

Comment on any issue or PR:
```
/test-tier0
```

This will start 4 parallel test runs (one per platform) using the **latest snapshot** from `fbc-mta-8-3`.

### Monitor Progress

**In GitHub:**
- PaC bot will respond with status checks on your comment

**In Konflux:**
```bash
oc get pipelinerun -n art-mta-tenant | grep tier0
```

**In Konflux UI:**
https://konflux-ui.apps.kflux-ocp-p01.7ayg.p1.openshiftapps.com/ns/art-mta-tenant/pipelineruns

### Expected Timeline

- **OCP-AWS**: ~1.5 hours (fastest - SNO)
- **ROSA/ROSA-STS**: ~2 hours
- **ARO**: ~2.5 hours (slowest - Azure provisioning)

## Pipeline Details

**Pipeline:** `pipelines/multicloud/mta-tier0-pipeline.yaml`

**Parameters:**
- `clusterType`: Platform to test (aro, rosa, rosa-sts, ocp-aws)
- `nvrOrSnapshot`: NVR or snapshot name (currently hardcoded to "" = latest)
- `triggeredBy`: User who triggered (auto-filled by PaC)

**OCP Version:** 4.22 (hardcoded)

**Application:** Auto-derived from NVR (currently uses latest from fbc-mta-8-3)

## Prerequisites

- **GitHub App:** `konflux-kflux-ocp-p01` must be installed on this repository
- **Repository CR:** PaC Repository resource must exist in `art-mta-tenant` namespace
- **OCPCTL Profiles:** Cluster provisioning profiles (aro-standard, rosa-standard, etc.) must be available

## Future Enhancements

- [ ] Parse NVR from GitHub comment (e.g., `/test-tier0 nvr=mta-operator-container-8.4.0...`)
- [ ] Support testing specific OCP versions
- [ ] Support testing specific applications (8.2, 8.3, 8.4, etc.)
- [ ] Trigger single platform instead of all 4
- [ ] Combined status notification (one message with all 4 results)

## Troubleshooting

**No PipelineRuns created after commenting `/test-tier0`:**
1. Check GitHub App is installed: https://github.com/migtools/mta-konflux-its/settings/installations
2. Verify Repository CR exists: `oc get repository mta-konflux-its -n art-mta-tenant` (admin only)
3. Check PaC webhook events in GitHub: Settings → Webhooks → Recent Deliveries

**PipelineRun fails:**
- Check logs: `tkn pipelinerun logs <pipelinerun-name> -n art-mta-tenant -f`
- Common issues: OCPCTL capacity limits, snapshot not found, cluster provisioning timeout
