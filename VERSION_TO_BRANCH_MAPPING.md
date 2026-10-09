# MTA Version to Test Branch Mapping

This document defines the mapping between MTA versions and their corresponding mta-tackle2-ui test branches used for upgrade testing.

## Current Mapping

| MTA Version | Test Branch   | Notes                          |
|-------------|---------------|--------------------------------|
| 8.0.x       | release-0.8   | MTA 8.0 family                 |
| 8.1.x       | release-0.9   | MTA 8.1 family                 |
| 8.2.x       | release-0.10  | MTA 8.2 family                 |
| 8.3.x       | release-0.11  | MTA 8.3 family                 |

## Adding Support for New Versions

When a new MTA version is released, update the mapping in:

**File:** `tasks/map-version-to-test-branch.yaml`

**Location:** In the `case` statement within the `map-version` step

**Example:** To add MTA 8.4 support:

```bash
case "$MAJOR_MINOR" in
  8.0)
    TEST_BRANCH="release-0.8"
    ;;
  8.1)
    TEST_BRANCH="release-0.9"
    ;;
  8.2)
    TEST_BRANCH="release-0.10"
    ;;
  8.3)
    TEST_BRANCH="release-0.11"
    ;;
  8.4)
    TEST_BRANCH="release-0.12"  # <-- Add this new case
    ;;
  *)
    echo "❌ No test branch mapping found for MTA version $MAJOR_MINOR"
    exit 1
    ;;
esac
```

## How It Works

The upgrade pipeline automatically:
1. Extracts the major.minor version from `gaVersion` (e.g., "8.2.2" → "8.2")
2. Extracts the major.minor version from `targetVersion` (e.g., "8.3.0" → "8.3")
3. Maps each to the corresponding test branch using the mapping table
4. Uses the mapped branches for pre-upgrade and post-upgrade tests

**Example:**
- Upgrading from 8.2.2 to 8.2.3: both use `release-0.10`
- Upgrading from 8.2 to 8.3: pre uses `release-0.10`, post uses `release-0.11`

## Troubleshooting

If you see an error like:
```
❌ No test branch mapping found for MTA version X.Y
   Update tasks/map-version-to-test-branch.yaml to add mapping for this version
```

This means the version being tested doesn't have a mapping entry. Follow the "Adding Support for New Versions" instructions above.
