---
name: 'upgrade-packmind-chart'
description: "Upgrade Packmind Helm Chart following a new Packmind app release"
---

# Upgrade Packmind Chart

This skill upgrades the Packmind Helm chart to a new version.

## Usage

```
/upgrade-packmind-chart [version]
```

Where `[version]` is the new Packmind application version in semver format (e.g., `1.8.0`).

## Instructions

When this skill is invoked:

1. **Check if version is provided**:
   - If NO version argument is provided, use the AskUserQuestion tool to prompt the user: "What is the new Packmind version? (format: x.y.z, e.g., 1.8.0)"
   - Wait for the user's response before proceeding

2. **Validate the version format**: Ensure the provided version matches the semver pattern `x.y.z` (e.g., `1.8.0`). If invalid, inform the user and stop.

3. **Read the current Chart.yaml** at `packmind/Chart.yaml` to get the current chart version.

4. **Calculate the new chart version**:
   - Parse the current chart `version` (e.g., `0.21.0`)
   - Increment the minor version by 1 (e.g., `0.21.0` → `0.22.0`)
   - Reset patch to 0 if incrementing minor

5. **Update `packmind/Chart.yaml`**:
   - Update `appVersion` to the provided version
   - Update `version` to the new incremented chart version

6. **Update `packmind/values.yaml`**:
   - Update `api.image.tag` to the provided version
   - Update `frontend.image.tag` to the provided version
   - Update `mcpServer.image.tag` to the provided version

7. **Report the changes**:
   - Show the old and new chart version
   - Show the old and new app version
   - List all updated image tags

8. **Create a git commit**:
   - Stage the modified files: `packmind/Chart.yaml` and `packmind/values.yaml`
   - Create a commit with message: `Release <new_chart_version> for Packmind <new_app_version>`
   - Example: `Release 0.22.0 for Packmind 1.8.0`

9. **Tag the commit**:
   - Create a git tag with format: `release/<new_chart_version>`
   - Example: `release/0.22.0`

10. **Push to remote**:
    - Push the commit to the remote repository
    - Push the tag to the remote repository

## Example

```
/upgrade-packmind-chart 1.8.0
```

This will:
- Update `Chart.yaml` version from `0.21.0` to `0.22.0`
- Update `Chart.yaml` appVersion from `1.7.0` to `1.8.0`
- Update all image tags in `values.yaml` from `1.7.0` to `1.8.0`
- Create a commit: `Release 0.22.0 for Packmind 1.8.0`
- Tag the commit: `release/0.22.0`
- Push commit and tag to remote
