# Semver Bump and Cargo Publish

A GitHub Action that automatically bumps the version of a Rust crate according to semantic versioning rules and publishes it to crates.io.

## Features

- 🔄 Automatic semantic version bumping (patch, minor, major)
- 📦 Publishes to crates.io
- 🏷️ Creates git tags automatically
- ✅ Waits for configurable status checks to pass
- 🧪 Dry run mode for testing
- 🔒 Rollback on publish failure
- 🌿 Branch-aware publishing (only publishes from main by default)
- 🛠️ Built-in Rust toolchain management
- 📋 Comprehensive validation and testing

## Limitations

- You need to manually create a GitHub release from each tag.
- No automatic release notes management.
- Only supports crates.io as the registry.
- Only supports packages with a single crate in the root directory.
- **SECURITY:** requires you to add a PAT to your repository secrets with write access to the repo.

## Usage

### Basic Usage

```yaml
name: Publish

on:
  workflow_dispatch:
    inputs:
      bump_type:
        description: "Version bump type"
        required: true
        default: "patch"
        type: choice
        options:
          - patch
          - minor
          - major
      dry_run:
        description: "Dry run mode"
        required: false
        default: true
        type: boolean

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Publish crate
        uses: your-username/semver-bump-and-cargo-publish@v1
        with:
          bump_type: ${{ github.event.inputs.bump_type }}
          dry_run: ${{ github.event.inputs.dry_run }}
          cargo_registry_token: ${{ secrets.CARGO_REGISTRY_TOKEN }}
          pat_token: ${{ secrets.PAT_TOKEN }}
```

### Advanced Usage with Status Check Dependencies

```yaml
name: Publish

on:
  workflow_dispatch:
    inputs:
      bump_type:
        description: "Version bump type"
        required: true
        default: "patch"
        type: choice
        options:
          - patch
          - minor
          - major
      dry_run:
        description: "Dry run mode"
        required: false
        default: true
        type: boolean

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Publish crate
        uses: your-username/semver-bump-and-cargo-publish@v1
        with:
          bump_type: ${{ github.event.inputs.bump_type }}
          dry_run: ${{ github.event.inputs.dry_run }}
          cargo_registry_token: ${{ secrets.CARGO_REGISTRY_TOKEN }}
          pat_token: ${{ secrets.PAT_TOKEN }}
          wait_for_checks: "CI / Test, CI / Lint, CI / Check Formatting"
          check_wait_interval: "60"
          check_timeout_count: "20"
          rust_toolchain: "nightly-2025-09-30"
          git_user_email: "bot@yourcompany.com"
          git_user_name: "Release Bot"
```

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `branch` | Branch to publish from | Yes | `main` |
| `bump_type` | Version bump type (`patch`, `minor`, `major`) | Yes | `patch` |
| `dry_run` | Skip automated commit, push, and publish | No | `true` |
| `cargo_registry_token` | Cargo registry token for crates.io | Yes | - |
| `pat_token` | Personal access token for repository access | Yes | - |
| `rust_toolchain` | Rust toolchain version to use | No | `nightly-2025-09-30` |
| `git_user_email` | Git user email for commits | No | `github-actions@github.com` |
| `git_user_name` | Git user name for commits | No | `GitHub Actions` |
| `wait_for_checks` | Comma-separated list of status check contexts to wait for | No | `""` |
| `check_wait_interval` | Time to wait between status check polling (seconds) | No | `60` |
| `check_timeout_count` | Number of polling attempts before timeout | No | `20` |

## Outputs

| Output | Description |
|--------|-------------|
| `package_name` | Name of the published package |
| `old_version` | Previous version before bump |
| `new_version` | New version after bump |
| `tag_name` | Git tag name created |
| `published` | Whether the package was published to crates.io |

## Prerequisites

### Required Secrets

You need to set up the following secrets in your repository:

1. **`CARGO_REGISTRY_TOKEN`**: Your crates.io API token
    - Get this from [crates.io/me](https://crates.io/me)
    - Go to Account Settings → API Tokens → New Token

2.  **`PAT_TOKEN`**: GitHub Personal Access Token
    - Go to GitHub Settings → Developer settings → Personal access tokens
    - Create a fine-grained token with the following:
      - Repository access: Select your repository, only the one where this action runs.
      - Scopes:
        - `repo` scope (read + write)
        - `metadata` scope (read-only)
    - This is needed to push commits and tags back to the repository

### Required Permissions

The workflow needs the following permissions:

```yaml
permissions:
  contents: write
  pull-requests: read
```

## Status Check Integration

The action can wait for specific GitHub status checks to pass before proceeding with the publish. This is useful for ensuring CI tests pass before publishing.

### Finding Status Check Names

To find the correct status check names for the `wait_for_checks` input:

1. Go to a recent commit in your repository
2. Click on the status checks (✅ or ❌ icon)
3. The names shown are what you should use in `wait_for_checks`

For GitHub Actions workflows, the format is typically:
- `{workflow_name} / {job_name}`
- Example: `CI / Test`, `CI / Lint`

### Example with Common CI Checks

```yaml
wait_for_checks: "CI / Test, CI / Lint, CI / Check Formatting"
```

## Workflow Logic

1. **Validation**: Validates inputs and required tokens
2. **Status Checks**: Waits for specified status checks to pass (if configured)
3. **Setup**: Installs Rust toolchain and required tools
4. **Version Bump**: Bumps version in `Cargo.toml` and updates `Cargo.lock`
5. **Testing**: Runs tests, clippy, and formatting checks
6. **Git Operations**: Commits changes and creates git tag
7. **Dry Run Publish**: Tests the publish process
8. **Push**: Pushes changes to repository (if not dry run)
9. **Publish**: Publishes to crates.io (if not dry run and on main branch)
10. **Rollback**: Automatically rolls back on publish failure

## Branch Behavior

- **Main branch**: Full publish to crates.io
- **Other branches**: Version bump and tag creation only (no crates.io publish)
- **Dry run**: No commits, pushes, or publishes (testing only)

## Error Handling

- Automatic rollback if crates.io publish fails
- Clear error messages for common issues
- Validation of all inputs before starting
- Timeout handling for status checks

## Examples

### Simple Monthly Release

```yaml
name: Monthly Release
on:
  schedule:
    - cron: '0 0 1 * *'  # First day of every month
  workflow_dispatch:

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: your-username/semver-bump-and-cargo-publish@v1
        with:
          bump_type: "minor"
          dry_run: "false"
          cargo_registry_token: ${{ secrets.CARGO_REGISTRY_TOKEN }}
          pat_token: ${{ secrets.PAT_TOKEN }}
```

### Release with Full CI Integration

```yaml
name: Release
on:
  workflow_dispatch:
    inputs:
      version:
        type: choice
        options: [patch, minor, major]
        default: patch

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: your-username/semver-bump-and-cargo-publish@v1
        with:
          bump_type: ${{ github.event.inputs.version }}
          dry_run: "false"
          cargo_registry_token: ${{ secrets.CARGO_REGISTRY_TOKEN }}
          pat_token: ${{ secrets.PAT_TOKEN }}
          wait_for_checks: "CI / test, CI / lint, CI / check-formatting"
          rust_toolchain: "stable"
```

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## Troubleshooting

### Status Check Issues

If you're having trouble with the `wait_for_checks` feature:

1. **Check the exact names**: Go to a recent commit in your repository, click on the status checks, and copy the exact names shown.

2. **Format for GitHub Actions**: Use the format `workflow_name / job_name` exactly as shown in the GitHub UI.
   - Example: `CI / Test`, `CI / Lint`, `CI / Check Formatting`

3. **Case sensitivity**: Names are case-sensitive and must match exactly.

4. **External CI systems**: For non-GitHub Actions (Travis CI, CircleCI, etc.), use the context name as shown in the status API.

5. **Debugging**: If a check isn't found, the action will show helpful error messages and suggestions.

### Common Issues

- **"Status check not found"**: Verify the exact name format and spelling
- **"Timeout waiting for status check"**: Check if the workflow is actually running or if it failed
- **"Invalid bump_type"**: Must be one of: `patch`, `minor`, `major`
- **Token issues**: Ensure both `CARGO_REGISTRY_TOKEN` and `PAT_TOKEN` secrets are set

### Getting Help

If you encounter issues:
1. Check the action logs for detailed error messages
2. Verify your repository secrets are configured correctly
3. Test with `dry_run: true` first
4. Open an issue with the full error log and your workflow configuration

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
