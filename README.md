# 4D Actions

A collection of reusable GitHub Actions workflows for automating the build, testing, and deployment of **4D** applications.

## Available Workflows

| Workflow | Description |
|----------|-------------|
| `get_tool4d.yml` | Downloads and caches the appropriate version of **tool4d** for the current runner. |
| `check_4d_syntax.yml` | Runs a 4D startup method using **tool4d** and returns the result to the calling workflow. |
| `get_cache_4d_binaries.yml` | Downloads and installs **4D Downloader** on macOS and Windows runners in preparation for downloading and caching 4D applications. |

---

# Quick Start

Call one of the reusable workflows from another repository:

```yaml
jobs:
  syntax:
    uses: madamov/4d_actions/.github/workflows/check_4d_syntax.yml@v1
    with:
      startup_method: checkSyntax
      runner: windows-latest  
      user_parameters: |
        {
          "errorFolderPath": "__DOCUMENTS__"
        }
```

Use the outputs in subsequent jobs:

```yaml
jobs:
  syntax:
    uses: madamov/4d_actions/.github/workflows/check_4d_syntax.yml@v1
    with:
      startup_method: checkSyntax
      runner: windows-latest
      user_parameters: |
        {
          "errorFolderPath": "__DOCUMENTS__"
        }

  verify:
    needs: syntax
    runs-on: ubuntu-latest

    steps:
      - name: Display results
        run: |
          echo "Success: ${{ needs.syntax.outputs.success }}"
          echo "Result file: ${{ needs.syntax.outputs.error_file }}"

      - name: Fail if syntax check failed
        if: needs.syntax.outputs.success != 'true'
        run: exit 1
```

---

# Workflows

## get_tool4d.yml

Downloads and caches the correct version of **tool4d**.

Typical usage:

```yaml
jobs:
  tool4d:
    uses: madamov/4d_actions/.github/workflows/get_tool4d.yml@v1
    with:
      version: "20.8"
      runner: macos-latest
```

The workflow automatically:

- Downloads tool4d if it is not already cached
- Restores cached versions when available
- Selects the correct operating system and architecture

---

## get_cache_4d_binaries.yml

Downloads a published release of **4D Downloader** and installs the appropriate package on both `macos-latest` and `windows-latest` runners.

Typical usage:

```yaml
jobs:
  cache-4d-binaries:
    uses: madamov/4d_actions/.github/workflows/get_cache_4d_binaries.yml@main
    with:
      # Optional: omit this value to use the latest published release.
      downloader_version: "1.0.0"
    secrets:
      # Required when the 4D-Downloader repository is private and the
      # caller's GITHUB_TOKEN cannot read its releases.
      DOWNLOADER_TOKEN: ${{ secrets.DOWNLOADER_TOKEN }}
```

The workflow automatically:

- Runs a macOS and Windows matrix
- Resolves the requested release tag, accepting versions with or without a leading `v`
- Uses the latest non-draft, non-prerelease release when no version is provided
- Selects the macOS DMG matching the runner architecture (Apple Silicon or Intel)
- Downloads and installs the Windows ZIP release
- Adds the resolved release, architecture, and installation paths to the workflow summary
- Supports manual execution with `workflow_dispatch`

### Inputs

| Name | Required | Description |
|------|:--------:|-------------|
| `downloader_version` | | 4D Downloader release tag. A leading `v` is optional; an empty value selects the latest published release. |

### Secrets

| Name | Required | Description |
|------|:--------:|-------------|
| `DOWNLOADER_TOKEN` | | Token with read access to the private `madamov/4D-Downloader` repository. When omitted, the workflow uses the caller's `GITHUB_TOKEN`. |

> [!NOTE]
> Despite its name, the current workflow installs 4D Downloader itself. Downloading, installing, and caching the selected 4D application binaries is the next stage of this workflow.

---

## check_4d_syntax.yml

Runs a startup method using tool4d.

The workflow automatically:

1. Checks out the caller repository.
2. Locates the `.4DProject`.
3. Reads the project's `compatibilityVersion`.
4. Determines the required tool4d version.
5. Restores or downloads tool4d.
6. Executes the specified startup method.
7. Reads the generated result JSON.
8. Returns outputs to the caller.

### Inputs

| Name | Required | Description |
|------|:--------:|-------------|
| `startup_method` | ✅ | Name of the 4D startup method to execute |
| `user_parameters` | ✅ | JSON passed to `--user-param` |
| `runner` | | Runner to execute on (default: `macos-latest`) |

### Outputs

| Name | Description |
|------|-------------|
| `success` | Value of the `success` property in the generated JSON |
| `error_file` | Full path to the generated JSON file |

---

# Supported Placeholders

String values inside `user_parameters` may contain the following placeholders:

| Placeholder | Description |
|-------------|-------------|
| `__DOCUMENTS__` | Runner's Documents folder |
| `__WORKSPACE__` | GitHub Actions workspace |

Example:

```json
{
    "errorFolderPath": "__DOCUMENTS__/Errors",
    "repository": "__WORKSPACE__"
}
```

---

# Expected Result File

The startup method must generate a JSON file named:

```
<ProjectName>_errors.json
```

For example:

```
MyApplication.4DProject
```

must generate:

```
MyApplication_errors.json
```

containing at least:

```json
{
    "success": true
}
```

Additional compiler errors or warnings may also be included.

---

# Supported 4D Versions

| Version | macOS ARM | macOS Intel | Windows |
|---------|:---------:|:-----------:|:-------:|
| 20.8 | ✅ | ✅ | ✅ |
| 20.8 HF3 | ✅ | ✅ | ✅ |
| 21.1 | ✅ | ✅ | ✅ |
| 21 R3 | ✅ | ✅ | ✅ |

---

# Requirements

The calling repository must contain:

- Exactly one `.4DProject`
- A valid `compatibilityVersion`
- The requested startup method
- A startup method that generates the expected JSON result

---

# Versioning

Examples for functionality added after `v1` use the latest code on the `main` branch:

```yaml
uses: madamov/4d_actions/.github/workflows/get_cache_4d_binaries.yml@main
```

For production workflows, prefer a release tag or commit SHA containing the workflow version you need. Existing workflows released with version 1 can use the major tag:

```yaml
uses: madamov/4d_actions/.github/workflows/check_4d_syntax.yml@v1
```

---

# Repository Structure

```
.github/
└── workflows/
    ├── get_cache_4d_binaries.yml
    ├── get_tool4d.yml
    ├── check_4d_syntax.yml
    └── update-major-version-tag.yml
```

---

# License

This repository is licensed under the MIT License. See the `LICENSE` file for details.
