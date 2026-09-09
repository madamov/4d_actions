# 4D Actions

A collection of reusable GitHub Actions workflows for automating the build, testing, and deployment of **4D** applications.

## Available Workflows

| Workflow | Description |
|----------|-------------|
| `get_tool4d.yml` | Downloads and caches the appropriate version of **tool4d** for the current runner. |
| `check_4d_syntax.yml` | Runs a 4D startup method using **tool4d** and returns the result to the calling workflow. |
| `get_cache_4d_binaries.yml` | Downloads, installs, caches, archives, and optionally uploads 4D build binaries for macOS and Windows. |

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

Prepares the 4D binaries required by application-build workflows without making every build download and run a full 4D installer. It downloads the selected 4D release on both macOS and Windows, installs its applications, stores them in the GitHub Actions cache, and creates ZIP archives that can optionally be uploaded to an SFTP server.

The workflow prepares three products:

- **4D (Standalone)** — used by later workflows to build standalone and client applications from a 4D Project.
- **4D Server** — used to build a standalone server application from a 4D Project.
- **4D Volume Desktop** — the Volume Desktop runtime used when producing standalone or client applications.

Publishing these archives to SFTP provides a platform-independent binary store for other workflows. A build workflow can download the exact prepared binaries from SFTP and immediately build a 4D Project, avoiding the time and complexity of downloading the original installer, mounting or executing it, and installing all three applications for every build.

Typical usage:

```yaml
jobs:
  cache-4d-binaries:
    uses: madamov/4d_actions/.github/workflows/get_cache_4d_binaries.yml@v1
    with:
      version: "20.8 HF3"
      downloader_version: "1.0.0" # Optional; defaults to the latest release.
      sftp_url: "sftp://files.example.com/4d" # Optional.
    secrets:
      PRODUCT_DOWNLOAD_USERNAME: ${{ secrets.PRODUCT_DOWNLOAD_USERNAME }}
      PRODUCT_DOWNLOAD_PASSWORD: ${{ secrets.PRODUCT_DOWNLOAD_PASSWORD }}
      DOWNLOADER_TOKEN: ${{ secrets.DOWNLOADER_TOKEN }}
      SFTP_USERNAME: ${{ secrets.SFTP_USERNAME }}
      SFTP_PASSWORD: ${{ secrets.SFTP_PASSWORD }}
      SFTP_HOST_FINGERPRINT: ${{ secrets.SFTP_HOST_FINGERPRINT }}
```

The workflow automatically:

- Runs a macOS and Windows matrix
- Resolves and installs the appropriate 4D Downloader release
- Downloads the selected 4D installer from `product-download.4d.com`
- Installs 4D, 4D Server, and 4D Volume Desktop
- Caches each installed product separately for later GitHub Actions jobs
- Creates a ZIP archive for every installed product on both platforms
- Uploads the archives to SFTP when the complete SFTP configuration is provided
- Verifies the SFTP server's SSH host key against the supplied SHA-256 fingerprint before uploading
- Uses Homebrew curl on macOS and discovers an SFTP-capable curl on Windows
- Adds downloader, installer, and installed-application details to the workflow summary
- Supports manual execution with `workflow_dispatch`

### Inputs

| Name | Required | Description |
|------|:--------:|-------------|
| `version` | ✅ | 4D version to download, install, and cache, such as `20.8 HF3` or `21 R3`. |
| `downloader_version` | | 4D Downloader release tag. A leading `v` is optional; an empty value selects the latest published release. |
| `sftp_url` | | SFTP destination directory, such as `sftp://files.example.com/4d`. Leave empty to disable SFTP upload. |

### Secrets

| Name | Required | Description |
|------|:--------:|-------------|
| `PRODUCT_DOWNLOAD_USERNAME` | ✅ | Username for `product-download.4d.com`. |
| `PRODUCT_DOWNLOAD_PASSWORD` | ✅ | Password for `product-download.4d.com`. |
| `DOWNLOADER_TOKEN` | | Token with read access to the private `madamov/4D-Downloader` repository. When omitted, the workflow uses the caller's `GITHUB_TOKEN`. |
| `SFTP_USERNAME` | | Username used to upload the ZIP archives to the SFTP server. |
| `SFTP_PASSWORD` | | Password used to upload the ZIP archives to the SFTP server. |
| `SFTP_HOST_FINGERPRINT` | | Expected SHA-256 SSH host-key fingerprint, including the `SHA256:` prefix. |

SFTP upload is optional. To enable it, provide `sftp_url` and all three SFTP secrets: `SFTP_USERNAME`, `SFTP_PASSWORD`, and `SFTP_HOST_FINGERPRINT`. If any of these four values is absent, archive upload is skipped. The product-download credentials remain required regardless of whether SFTP upload is enabled.

### Archive names

Archive names contain the normalized 4D version, platform, and product. Dots and spaces in the version are replaced by underscores:

| Product | macOS example | Windows example |
|---------|---------------|-----------------|
| 4D (Standalone) | `4d_20_8_HF3_mac.zip` | `4d_20_8_HF3_win.zip` |
| 4D Server | `4d_20_8_HF3_mac_server.zip` | `4d_20_8_HF3_win_server.zip` |
| 4D Volume Desktop | `4d_20_8_HF3_mac_vl.zip` | `4d_20_8_HF3_win_vl.zip` |

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
| 20.8 HF4 | ✅ | ✅ | ✅ |
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

# Repository Structure

```
.github/
└── workflows/
    ├── get_cache_4d_binaries.yml
    ├── get_tool4d.yml
    ├── check_4d_syntax.yml

```

---

# License

This repository is licensed under the MIT License. See the `LICENSE` file for details.
