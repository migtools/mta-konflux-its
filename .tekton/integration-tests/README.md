# Pipelines

Tekton `Pipeline` definitions consumed as Konflux `IntegrationTestScenario` (ITS) pipelines via
the git resolver. Reference one with `pathInRepo: .tekton/integration-tests/<file>.yaml` (see the repo
[README](../README.md)).

## `mta-fbc-e2e-pipeline.yaml`

Deploys the MTA operator from an FBC (File-Based Catalog) image onto a leased OCPCTL pool
cluster and runs the Cypress UI E2E suite plus a DAST security scan.

**Features:**
- **OCP version filtering**: Only tests on OCP 4.20, 4.21, 4.22 to conserve cluster resources
- **NVR extraction & validation**: Extracts operator NVR from snapshot and validates FBC catalog consistency
- **Controlled notifications**: Slack notifications only sent when tests actually run (not for filtered snapshots)

### Flow

```
parse-metadata → filter-ocp-version → extract-operator-nvr → verify-image-pullable
  → lease-cluster → cleanup-existing-mta → deploy-operator → run-dast-scan → run-e2e-tests
finally: release-cluster, slack-notification
```

The pipeline fails fast if the snapshot targets an unsupported OCP version (before leasing clusters).
The leased cluster is always returned to the pool via the `finally` tasks, whether the tests
pass or fail. Slack notifications are sent only when DAST scan runs (meaning the OCP filter passed).

### Parameters

| Name       | Default         | Description                                     |
| ---------- | --------------- | ----------------------------------------------- |
| `SNAPSHOT` | _(required)_    | Application Snapshot JSON (provided by Konflux) |
| `poolName` | `ci-sno-pool-1` | OCPCTL cluster pool to lease from               |

## `mta-fbc-koncur-e2e-pipeline.yaml`

Deploys the MTA operator from an FBC (File-Based Catalog) image onto a leased OCPCTL pool
cluster and runs koncur tackle-hub E2E tests.

### Flow

```
parse-metadata → lease-cluster → cleanup-existing-mta → deploy-operator
  → run-koncur-hub-tests
finally: release-cluster
```

The leased cluster is always returned to the pool via the `finally` task, whether the tests
pass or fail.

### Parameters

| Name           | Default          | Description                                              |
| -------------- | ---------------- | -------------------------------------------------------- |
| `SNAPSHOT`     | _(required)_      | Application Snapshot JSON (provided by Konflux)          |
| `poolName`     | `ci-koncur-pool`  | OCPCTL cluster pool to lease from                        |
| `koncurBranch` | `main`            | Branch of `konveyor/koncur` to build and test from       |
