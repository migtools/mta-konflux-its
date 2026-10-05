# Pipelines

Tekton `Pipeline` definitions consumed as Konflux `IntegrationTestScenario` (ITS) pipelines via
the git resolver. Reference one with `pathInRepo: pipelines/<file>.yaml` (see the repo
[README](../README.md)).

## `mta-fbc-ui-koncur-e2e-pipeline.yaml`

Deploys the MTA operator from an FBC (File-Based Catalog) image onto a leased OCPCTL pool
cluster and runs DAST scan, koncur hub tests, and UI Cypress tests **sequentially**.

**Sequential execution prevents auth conflicts**: DAST disables authentication for scanning,
which would break UI tests if they ran in parallel. Tests run in order: DAST → Koncur → UI.

**Benefits:**
- **Single cluster lease**: All test suites run on the same deployment
- **Reuse operator deployment**: No duplicate setup overhead  
- **Skip vs Fail**: If any test fails, subsequent tests are **skipped** (not failed)
- **Combined reporting**: Slack notification includes all test results (DAST + Koncur + UI)
- **Resource efficiency**: Lower cluster usage compared to separate pipelines

### Flow

```
parse-metadata → filter-ocp-version → extract-operator-nvr → verify-image-pullable
  → lease-cluster → cleanup-existing-mta → deploy-operator
  → run-dast-scan → run-koncur-tests → run-e2e-tests (UI Cypress)
finally: cleanup-notify
  (releases cluster → sends Slack notification)
```

Tests run **sequentially** to avoid auth conflicts. If DAST fails, Koncur and UI are **skipped**.
If Koncur fails, UI is **skipped**. The finally task always runs (if deployment succeeded) to
release the cluster and send Slack notification with all test results.

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

## `mta-fbc-upgrade-e2e-pipeline.yaml`

Tests upgrade path from GA MTA version to FBC version being built.

**Upgrade Flow:**
1. **Deploy GA version** from certified-operators catalog (e.g., MTA 8.2)
2. **Run pre-upgrade tests** to validate GA version functionality
3. **Upgrade to FBC version** being tested (e.g., MTA 8.3)
4. **Run post-upgrade tests** to verify upgrade success and functionality

**Upgrade Types:**
- **Z-stream**: Same major.minor version (8.2.0 → 8.2.1) - only approves install plan
- **Y-stream**: Different major.minor (8.2 → 8.3) - updates subscription channel and catalog source

### Flow

```
parse-metadata → filter-ocp-version → extract-operator-nvr → verify-image-pullable
  → lease-cluster → cleanup-existing-mta
  → deploy-ga-operator → run-pre-upgrade-tests
  → upgrade-operator → run-post-upgrade-tests
finally: cleanup-notify
  (releases cluster → sends Slack notification)
```

### Parameters

| Name                      | Default         | Description                                           |
| ------------------------- | --------------- | ----------------------------------------------------- |
| `SNAPSHOT`                | _(required)_    | Application Snapshot JSON (provided by Konflux)       |
| `poolName`                | `ci-sno-pool-1` | OCPCTL cluster pool to lease from                     |
| `gaVersion`               | `8.2`           | GA MTA version to start with (e.g., "8.2", "8.3")     |
| `targetVersion`           | `8.3.0`         | Target MTA version after upgrade (e.g., "8.3.0")      |
| `preUpgradeTestBranch`    | `main`          | Branch of mta-tackle2-ui for pre-upgrade tests        |
| `postUpgradeTestBranch`   | `main`          | Branch of mta-tackle2-ui for post-upgrade tests       |
| `skipPreUpgradeTests`     | `false`         | Skip pre-upgrade tests                                |
| `skipPostUpgradeTests`    | `false`         | Skip post-upgrade tests                               |

### Example Usage

Test upgrade from MTA 8.2 to 8.3:

```yaml
apiVersion: appstudio.redhat.com/v1beta1
kind: IntegrationTestScenario
metadata:
  name: mta-fbc-upgrade-8-2-to-8-3
  namespace: your-namespace
spec:
  application: fbc-mta-8-3
  resolverRef:
    resolver: git
    params:
      - name: url
        value: https://github.com/migtools/mta-konflux-its
      - name: revision
        value: main
      - name: pathInRepo
        value: pipelines/mta-fbc-upgrade-e2e-pipeline.yaml
  params:
    - name: gaVersion
      value: "8.2"
    - name: targetVersion
      value: "8.3.0"
    - name: poolName
      value: "ci-sno-pool-1"
```
