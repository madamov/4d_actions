# 4D Actions

A collection of reusable GitHub Actions workflows for automating the build, testing, and deployment of **4D** applications.

## Available Workflows

| Workflow | Description |
|----------|-------------|
| `get_tool4d.yml` | Downloads and caches the appropriate version of **tool4d** for the current runner. |
| `check_4d_syntax.yml` | Runs a 4D startup method using **tool4d** and returns the result to the calling workflow. |

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

# Repository Structure

```
.github/
└── workflows/
    ├── get_tool4d.yml
    └── check_4d_syntax.yml
```

---

# License

This repository is licensed under the MIT License. See the `LICENSE` file for details.
