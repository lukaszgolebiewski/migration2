---
applyTo: "**/*.ps1"
---

# PowerShell Conventions

## Script Architecture

This repository uses PowerShell for two purposes:

1. **Helpers** (`blueprint/helpers/`) — Reusable utility functions sourced by pipeline scripts.
2. **Local pipeline** (`blueprint/local-pipeline/`) — Numbered execution stages (`1-azure-login.ps1` → `5-terraform-apply.ps1`) that drive Terraform workflows from a developer workstation.

Pipeline scripts are also invoked from Azure DevOps YAML templates via `AzureCLI@2` inline scripts.

## Function Design

- Use **approved PowerShell verbs** (`Get-`, `Set-`, `New-`, `Remove-`, `Update-`, `Select-`, `Test-`). Run `Get-Verb` if unsure.
- Use **PascalCase** for function names and parameters.
- Every function must accept `[CmdletBinding()]` and support `-Verbose` by default:
  ```powershell
  function Select-Config {
      [CmdletBinding()]
      param (
          [Parameter(Mandatory = $true)][string]$EnvShortname,
          [Parameter(Mandatory = $true)][string]$Root
      )
      Write-Verbose "Processing $Root for $EnvShortname"
      # ...
  }
  ```
- Prefer `Write-Verbose` over `Write-Host` for operational messages. Reserve `Write-Host` for user-facing progress banners only.
- Use `Write-Error` / `throw` for failures — never silently continue.

## Parameter Patterns

- Mark required parameters with `[Parameter(Mandatory = $true)]`.
- Provide defaults for optional parameters directly in the `param()` block.
- Standard parameters used across the repo:
  - `$env_shortname` — Environment identifier (`dev`, `test`, `snd`, `prod`).
  - `$subpipeline` — Blueprint module name (e.g., `storage`, `core`, `bootstrap`).
  - `$blueprint_root` — Relative path to the blueprint directory.
  - `$settings_root` — Relative path to the settings root.
- Use `[ValidateSet()]` for parameters with known valid values:
  ```powershell
  [ValidateSet("dev", "test", "snd", "prod")]
  [string]$EnvShortname
  ```

## Formatting & Style

- Use **splatting** for commands with more than three parameters:
  ```powershell
  $params = @{
      AccountName   = $storageAccountName
      ContainerName = "tfstate"
      Name          = $blobName
      AuthMode      = "key"
  }
  az storage blob upload @params
  ```
- Indent with 4 spaces (PowerShell community standard).
- Brace style: opening brace on same line (`K&R`), matching existing files.
- Avoid aliases in scripts (`Get-ChildItem` not `ls`, `ForEach-Object` not `%`).

## File Operations

- Use `Get-Content -Path` / `Set-Content -Path` for text manipulation.
- Use `-Raw` flag when reading entire files for regex replacement.
- Template processing pattern (used by `select-config`):
  ```powershell
  ((Get-Content -Path $template -Raw) -replace '###{env_shortname}###', $envShortname) |
      Set-Content -Path $specific
  ```

## External Tools

- The repo depends on `hcl2json` for parsing `.tf` settings files. Check with `ensure_tool_is_downloaded` before use.
- Azure CLI (`az`) calls should always specify `--auth-mode key` or use the service principal context.
- Terraform is invoked via `terraform -chdir="$code_root"` — never `cd` into the module directory.

## Error Handling

- Wrap critical sections in `try/catch`. Log the error and re-throw:
  ```powershell
  try {
      terraform -chdir="$root" plan -out "$planFile"
  } catch {
      Write-Error "Terraform plan failed: $_"
      throw
  }
  ```
- Use `$ErrorActionPreference = 'Stop'` at the top of scripts that should fail fast.
- Always check `$LASTEXITCODE` after external tool invocations (`terraform`, `az`, `hcl2json`).

## Security

- Never print credentials, tokens, or secrets to the console.
- Use `$env:AZDO_PERSONAL_ACCESS_TOKEN` for PAT injection — never pass as a visible parameter.
- When calling `az`, prefer `--output none` for mutating operations to avoid leaking response data.

## Naming Conventions

- Helper function files: lowercase kebab-case (`scheduling-functions.ps1`, `miscellaneous.ps1`).
- Pipeline stage scripts: numbered prefix with verb (`1-azure-login.ps1`, `2-terraform-int.ps1`).
- Functions inside helpers: PascalCase with descriptive verbs (`SwitchBootstrapToStorageBackend`, `Select-Config`).
