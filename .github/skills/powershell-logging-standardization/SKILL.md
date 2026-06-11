# Logging Standardization Skill

## Purpose
Enforce consistent logging patterns across repository PowerShell scripts. Standardize `Write-Host`, `Write-Verbose`, `Write-Warning`, `Write-Error`, and `Write-Debug` usage to follow the routing rules below. Do not rewrite existing `Write-Information` calls.

If the input is not PowerShell script content and is not a readable file path, respond with: `Input must be PowerShell script text or a valid file path.` Do not guess.

## When to Use
- During modernization (replacing ad-hoc Write-Host calls)
- During code generation (applying correct patterns from the start)

## Stream Routing Rules

Apply these rules in strict precedence order:

1. **Keep Azure DevOps markers as `Write-Host`.** Any `Write-Host` call containing `##[section]` or `##[command]` stays unchanged.
2. **Keep explicit console banners as `Write-Host`.** Use `Write-Host` only for Azure DevOps `##[section]` markers and explicit console banners intended for interactive users; do not use `Write-Host` for normal status messages or routine progress text.
3. **Convert operational `Write-Host` to `Write-Verbose`.** Use `Write-Verbose` for messages that describe script progress, internal state, or operator diagnostics; do not use it for user-visible banners, warnings, or errors. **Prerequisite:** the containing function must have `[CmdletBinding()]`. If it does not, either add `[CmdletBinding()]` when safe to do so, or report that conversion is blocked until `[CmdletBinding()]` is added — do not silently convert.
4. **Route warnings, errors, and debug output** using the standard cmdlets below.

| Stream | Cmdlet | Use Case | Example |
|---|---|---|---|
| Verbose | `Write-Verbose` | Script progress, internal state, operator diagnostics | `Write-Verbose "Processing $root for $env"` |
| Host | `Write-Host` | Azure DevOps `##[section]` markers and interactive console banners only | `Write-Host "##[section]TERRAFORM PLAN"` |
| Warning | `Write-Warning` | Recoverable issues, deprecation notices | `Write-Warning "Backend config missing, using defaults"` |
| Error | `Write-Error` | Failures before throwing | `Write-Error "Terraform plan failed: $_"` |
| Debug | `Write-Debug` | Developer-only trace (never in pipelines) | `Write-Debug "Variable state: $config"` |

## Transformation Patterns

### Replace operational Write-Host with Write-Verbose
**Before:**
```powershell
Write-Host "Generating '$SPECIFIC' from '$TEMPLATE'"
Write-Host "Generated $import_cnt import files"
```

**After:**
```powershell
Write-Verbose "Generating '$SPECIFIC' from '$TEMPLATE'"
Write-Verbose "Generated $import_cnt import files"
```

### Preserve Azure DevOps section markers
Azure DevOps `##[section]` markers are legitimate `Write-Host` usage:
```powershell
Write-Host "##[section]RUNNING PIPELINE FOR: $runtime"  # ✅ Keep as Write-Host
```

### Ensure CmdletBinding prerequisite
`Write-Verbose` only works when `[CmdletBinding()]` is present. Before converting any `Write-Host` to `Write-Verbose`, check whether the containing function has `[CmdletBinding()]`.

- If `[CmdletBinding()]` is present, proceed with conversion.
- If `[CmdletBinding()]` is missing and can be safely added (no conflicting attributes), add it and proceed.
- If `[CmdletBinding()]` is missing and cannot be safely added, report the conversion as **blocked** and do not convert that `Write-Host` call.

```powershell
function Select-Config {
    [CmdletBinding()]  # ← Required for Write-Verbose
    param ( ... )
    Write-Verbose "Processing..."
}
```

## Inputs
- Script content or file path
- Execution context (local vs Azure DevOps pipeline)

## Outputs
- Corrected logging statements
- List of changes with stream routing justification
