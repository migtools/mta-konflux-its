# Pipelines

Tekton `Pipeline` definitions consumed as Konflux `IntegrationTestScenario` (ITS) pipelines via
the git resolver. Reference one with `pathInRepo: pipelines/<file>.yaml` (see the repo
[README](../README.md)).

## `mta-fbc-ui-koncur-e2e-pipeline.yaml`

Deploys the MTA operator from an FBC (File-Based Catalog) image onto a leased OCPCTL pool
cluster and runs UI Cypress tests, DAST scan, and koncur hub tests.
UI and DAST run in parallel after deployment. Koncur tests run in the `finally` section,
ensuring they execute even if UI or DAST tests fail.

**Benefits:**
- **Single cluster lease**: Both test suites run on the same deployment
- **Reuse operator deployment**: No duplicate setup overhead
- **Combined reporting**: Slack notification includes all test results (UI + DAST + koncur)
- **Resource efficiency**: Lower cluster usage compared to separate pipelines

### Flow

```
parse-metadata → filter-ocp-version → extract-operator-nvr → verify-image-pullable
  → lease-cluster → cleanup-existing-mta → deploy-operator
  → ┌─ run-dast-scan
    └─ run-e2e-tests (UI Cypress)
finally: run-koncur-hub-tests, release-cluster, slack-notification
```

DAST and UI tests run **in parallel** after deployment for faster execution. Koncur tests run in the `finally` section, ensuring they execute even if UI or DAST tests fail. This prevents UI/koncur test interference (koncur waits until UI completes) while guaranteeing complete test coverage and Slack notification regardless of prior test results.

### Parameters

| Name           | Default         | Description                                              |
| -------------- | --------------- | -------------------------------------------------------- |
| `SNAPSHOT`     | _(required)_    | Application Snapshot JSON (provided by Konflux)          |
| `poolName`     | `ci-sno-pool-1` | OCPCTL cluster pool to lease from                        |
| `uiTestBranch` | `main`          | Branch of mta-tackle2-ui to use for UI E2E tests         |
| `koncurBranch` | `main`          | Branch of konveyor/koncur to build and test from         |

### Multi-Version Testing

Supports testing different MTA versions via parameters:

```yaml
spec:
  params:
    - name: uiTestBranch
      value: "release-0.10"
    - name: koncurBranch
      value: "release-0.10"
  resolverRef:
    params:
      - name: revision
        value: main
      - name: pathInRepo
        value: pipelines/mta-fbc-ui-koncur-e2e-pipeline.yaml
```
