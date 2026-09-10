# Pipelines

Tekton `Pipeline` definitions consumed as Konflux `IntegrationTestScenario` (ITS) pipelines via
the git resolver. Reference one with `pathInRepo: pipelines/<file>.yaml` (see the repo
[README](../README.md)).

## `mta-fbc-e2e-pipeline.yaml`

Deploys the MTA operator from an FBC (File-Based Catalog) image onto a leased OCPCTL pool
cluster and runs the Cypress UI E2E suite plus a DAST security scan.

### Flow

```
parse-metadata → extract-operator-nvr → verify-image-pullable → lease-cluster
  → cleanup-existing-mta → deploy-operator → run-dast-scan → run-e2e-tests
finally: release-cluster, slack-notification
```

The leased cluster is always returned to the pool via the `finally` tasks, whether the tests
pass or fail; a Slack notification is sent when tests actually ran.

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
