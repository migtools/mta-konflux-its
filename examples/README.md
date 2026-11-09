# IntegrationTestScenario Examples

Example `IntegrationTestScenario` (ITS) resources for MTA tier0 multicloud testing.

## Files

- **its-mta-ui-tests-on-multicloud-aro.yaml** - ITS for ARO (Azure Red Hat OpenShift)
- **its-mta-ui-tests-on-multicloud-rosa.yaml** - ITS for ROSA (Red Hat OpenShift Service on AWS)
- **its-mta-ui-tests-on-multicloud-rosa-sts.yaml** - ITS for ROSA-STS (ROSA with STS)
- **its-mta-ui-tests-on-multicloud-ocp-aws.yaml** - ITS for OCP-AWS (OpenShift on AWS SNO)

## Usage

Apply these to create/update ITS resources in Konflux:

```bash
# Delete old ITS if they exist with different names
oc delete integrationtestscenario mta-tier0-aro -n art-mta-tenant --ignore-not-found
oc delete integrationtestscenario mta-tier0-rosa -n art-mta-tenant --ignore-not-found
oc delete integrationtestscenario mta-tier0-rosa-sts -n art-mta-tenant --ignore-not-found
oc delete integrationtestscenario mta-tier0-ocp-aws -n art-mta-tenant --ignore-not-found

# Create new ITS with updated names and uiTestBranch parameter
oc apply -f examples/its-mta-ui-tests-on-multicloud-aro.yaml
oc apply -f examples/its-mta-ui-tests-on-multicloud-rosa.yaml
oc apply -f examples/its-mta-ui-tests-on-multicloud-rosa-sts.yaml
oc apply -f examples/its-mta-ui-tests-on-multicloud-ocp-aws.yaml
```

## Parameters

### uiTestBranch
- **Default**: `""` (empty = auto-detect)
- **Values**: Empty string or specific branch name

When empty, the pipeline automatically detects the test branch from the MTA version:
- MTA 8.0 → `release-0.8`
- MTA 8.1 → `release-0.9`
- MTA 8.2 → `release-0.10`
- MTA 8.3 → `release-0.11`

To override, specify a branch:
```yaml
params:
  - name: uiTestBranch
    value: "release-0.10"
```
