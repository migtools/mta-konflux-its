# mta-konflux-its

A catalog of [Tekton](https://tekton.dev/) pipelines and tasks used as
[Konflux](https://konflux-ci.dev/) `IntegrationTestScenario` (ITS) pipelines for the MTA tenant.

## Repository structure

```
pipelines/    # Integration test pipelines referenced by IntegrationTestScenarios
tasks/        # Reusable Tekton tasks referenced by the pipelines
README.md
```

## Pipeline Overview

**mta-fbc-ui-koncur-e2e-pipeline** - Full E2E testing pipeline with sequential execution:

1. **Deploy** - MTA operator from FBC catalog to leased cluster
2. **DAST Scan** - Security testing (disables auth)
3. **Koncur Tests** - Hub API integration tests
4. **UI E2E Tests** - Cypress UI tests (full tier suite)
5. **Finally** - Cluster cleanup + Slack notification (always runs)

**Execution model**: Tasks run sequentially. If any task fails, subsequent tasks are **skipped** (not failed). Slack notification always sends with status of all tasks.

**mta-tier0-rosa-pipeline** - Tier-0 testing on ROSA clusters:

1. **Day Filter** - Checks if tests should run today (default: Monday only)
2. **OCP Version Filter** - Only runs on OCP 4.22 snapshots
3. **Provision** - Fresh ROSA cluster via OCPCTL (1 hour timeout)
4. **Deploy** - MTA operator with IDMS for stage registry mirrors
5. **UI Tests** - Tier-0 Cypress tests
6. **Finally** - Cluster cleanup + Slack notification (always runs)

**Execution model**: Day-of-week gated. By default runs only on Mondays. Can be manually triggered any day by changing the `allowedDays` parameter.

## How it's used in Konflux

An `IntegrationTestScenario` references a pipeline from this repo via the Tekton
**git resolver** — an `url` + `revision` + `pathInRepo` triple:

```yaml
apiVersion: appstudio.redhat.com/v1beta2
kind: IntegrationTestScenario
spec:
  resolverRef:
    resolver: git
    resourceKind: pipeline
    params:
      - name: url
        value: https://github.com/migtools/mta-konflux-its
      - name: revision
        value: main   # branch name, tag, or commit SHA
      - name: pathInRepo
        value: pipelines/<pipeline-file>.yaml
```

Because files are addressed by exact `pathInRepo`, the folder layout above is a convention for
readability, not a resolver requirement. Pipelines in `pipelines/` resolve their tasks from
`tasks/` in this same repository.

## OCP Version Filtering

Snapshots are filtered by target OCP version to conserve cluster resources. The pipeline only runs on allowed OCP versions (default: 4.20, 4.21, 4.22, 5.0).

Snapshots targeting unsupported OCP versions fail fast at the `filter-ocp-version` step.

## Multi-Version Testing

Pipelines support testing different MTA versions via parameters:

- **Koncur tests**: Use `koncurBranch` parameter (default: `main`, override for older versions)
- **UI E2E tests**: Use `uiTestBranch` parameter (default: `main`, override for older versions)

Example ITS for testing with specific branches:
```yaml
apiVersion: appstudio.redhat.com/v1beta2
kind: IntegrationTestScenario
spec:
  params:
    - name: uiTestBranch
      value: "release-0.10"
    - name: koncurBranch
      value: "main"
  resolverRef:
    resolver: git
    params:
      - name: revision
        value: main
      - name: pathInRepo
        value: pipelines/mta-fbc-ui-koncur-e2e-pipeline.yaml
```

## Manually Triggering ROSA Pipeline

The ROSA pipeline (`mta-tier0-rosa-pipeline`) runs only on specific days to conserve cluster resources. By default, it runs only on **Mondays**.

### To manually trigger on any day:

1. **Update the IntegrationTestScenario** to allow today:
   ```bash
   # Option 1: Allow all days
   oc patch integrationtestscenario mta-ui-tests-on-multicloud -n art-mta-tenant --type=json -p='[
     {"op": "replace", "path": "/spec/params/1/value", "value": "all"}
   ]'
   
   # Option 2: Allow today specifically (e.g., Tuesday)
   oc patch integrationtestscenario mta-ui-tests-on-multicloud -n art-mta-tenant --type=json -p='[
     {"op": "replace", "path": "/spec/params/1/value", "value": "Tuesday"}
   ]'
   ```

2. **Trigger on a specific snapshot**:
   ```bash
   # Find recent snapshots
   oc get snapshot -n art-mta-tenant | grep fbc-mta-8-3
   
   # Trigger the ITS on a snapshot
   oc patch snapshot <snapshot-name> -n art-mta-tenant --type=json -p='[
     {"op": "add", "path": "/metadata/labels/test.appstudio.openshift.io~1run", "value": "true"}
   ]'
   ```

3. **Watch the pipeline run**:
   ```bash
   oc get pipelinerun -n art-mta-tenant -w | grep mta-ui-tests-on-multicloud
   ```

4. **Reset to Monday-only after testing** (optional):
   ```bash
   oc patch integrationtestscenario mta-ui-tests-on-multicloud -n art-mta-tenant --type=json -p='[
     {"op": "replace", "path": "/spec/params/1/value", "value": "Monday"}
   ]'
   ```

### Current ITS Parameters:

```yaml
params:
  - name: clusterType
    value: "rosa"           # Cluster type: rosa
  - name: allowedDays
    value: "Monday"         # Days to run: "Monday", "all", "Monday,Friday", etc.
```

## Key Features

- **Cluster pooling**: Uses OCPCTL for instant cluster access (no 15-30min provision wait)
- **Sequential execution**: DAST → Koncur → UI tests (prevents auth conflicts)
- **Skip vs Fail**: Failed tasks cause subsequent tasks to skip (not fail)
- **Always notify**: Slack notification in `finally` block always runs
- **Infrastructure**: UBI 9.5 base images with manual oc CLI installation
