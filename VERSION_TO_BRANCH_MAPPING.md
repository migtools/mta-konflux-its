# MTA Version to Test Branch Mapping

Mapping between MTA versions and mta-tackle2-ui test branches for upgrade testing.

## Current Mapping

| MTA Version | Test Branch   |
|-------------|---------------|
| 8.0.x       | release-0.8   |
| 8.1.x       | release-0.9   |
| 8.2.x       | release-0.10  |
| 8.3.x       | release-0.11  |

## Adding New Versions

Edit `tasks/map-version-to-test-branch.yaml` and add a new case:

```bash
8.4)
  TEST_BRANCH="release-0.12"
  ;;
```

The upgrade pipeline automatically maps versions to branches for pre-upgrade and post-upgrade tests.
