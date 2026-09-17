# mta-konflux-its

A catalog of [Tekton](https://tekton.dev/) pipelines and tasks used as
[Konflux](https://konflux-ci.dev/) `IntegrationTestScenario` (ITS) pipelines for the MTA tenant.

## Repository structure

```
pipelines/    # integration-test pipelines referenced by IntegrationTestScenarios
tasks/        # reusable Tekton tasks referenced by the pipelines
README.md
```

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

## Multi-Version Testing

Pipelines support testing different MTA versions via parameters rather than separate branches:

- **koncur tests**: Use `koncurBranch` parameter (default: `main`, override for older versions)
- **UI E2E tests**: Use `uiTestBranch` parameter (default: `main`, override for older versions)

Example ITS for MTA 8.2 (tests from release-0.10 branch):
```yaml
apiVersion: appstudio.redhat.com/v1beta2
kind: IntegrationTestScenario
spec:
  params:
    - name: uiTestBranch
      value: "release-0.10"
  resolverRef:
    resolver: git
    params:
      - name: revision
        value: main  # Always use main - version controlled via parameters
      - name: pathInRepo
        value: pipelines/mta-fbc-e2e-pipeline.yaml
```
