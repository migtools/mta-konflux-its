# Pipelines

Tekton `Pipeline` definitions consumed as Konflux `IntegrationTestScenario` (ITS) pipelines via
the git resolver. Reference one with `pathInRepo: pipelines/<file>.yaml` (see the repo
[README](../README.md)).

## `mta-fbc-ui-koncur-e2e-pipeline.yaml`

Deploys the MTA operator from an FBC (File-Based Catalog) image onto a leased OCPCTL pool
cluster and runs both UI Cypress tests and koncur hub tests sequentially on the same cluster.
Koncur tests run even if UI tests fail (failure-tolerant execution).

**Benefits:**
- **Single cluster lease**: Both test suites run on the same deployment
- **Reuse operator deployment**: No duplicate setup overhead
- **Combined reporting**: Slack notification includes all test results (UI + DAST + koncur)
- **Resource efficiency**: Lower cluster usage compared to separate pipelines

### Flow

```
parse-metadata → filter-ocp-version → extract-operator-nvr → verify-image-pullable
  → lease-cluster → cleanup-existing-mta → deploy-operator
  → run-dast-scan → run-e2e-tests (UI Cypress) → run-koncur-hub-tests
finally: release-cluster, slack-notification
```

Tests run **sequentially** after deployment. Koncur tests run even if UI tests fail (failure-tolerant execution) to ensure complete test coverage and Slack notification regardless of UI test status.

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
