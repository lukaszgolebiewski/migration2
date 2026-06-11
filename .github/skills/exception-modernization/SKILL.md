# Exception Modernization Skill

## Purpose
Convert legacy PowerShell error handling patterns to modern, structured exception handling using this explicit standard: wrap native executable calls in `try/catch`, throw on non-zero `$LASTEXITCODE`, preserve existing command arguments and formatting, and avoid adding duplicate error-handling wrappers when one already exists.

If the input already uses `try/catch`, `$LASTEXITCODE` checks, and explicit error handling throughout, return the code unchanged and state that no modernization was required.

If the input is not readable PowerShell script content or a valid script path, respond with an error message and do not guess at the transformation.

## When to Use
- During script modernization (anti-pattern remediation)
- During debugging (when silent failures are the root cause)

## Transformation Rules

Apply these rules in strict order. Do not over-apply — skip any rule whose condition is already satisfied in the existing code.

1. Detect whether the command is a native executable (e.g., `terraform`, `az`, `hcl2json`, or any command run via `&` or `Start-Process`). Do not add `$LASTEXITCODE` checks for PowerShell cmdlets or functions.
2. Add `$ErrorActionPreference = 'Stop'` only when the current script or scope lacks it.
3. Wrap the native command in `try/catch` only when the current code lacks a wrapper for that command.
4. Add a `$LASTEXITCODE` check immediately after the native command, only when one does not already exist.
5. Replace `$ErrorActionPreference = 'SilentlyContinue'` only when the code uses it to suppress an error that should be handled explicitly.

If the target command already has a `try/catch` wrapper or an explicit `$LASTEXITCODE` check, preserve the existing handling and do not add duplicate logic.

### 1. Add ErrorActionPreference
**Before:**
```powershell
# (no error preference set)
az storage blob upload --account-name $name
```

**After:**
```powershell
$ErrorActionPreference = 'Stop'
az storage blob upload --account-name $name
if ($LASTEXITCODE -ne 0) { throw "Blob upload failed with exit code $LASTEXITCODE" }
```

### 2. Wrap External Calls in try/catch
**Before:**
```powershell
terraform -chdir="$root" plan -out "$planFile"
```

**After:**
```powershell
try {
    terraform -chdir="$root" plan -out "$planFile"
    if ($LASTEXITCODE -ne 0) { throw "Terraform plan failed with exit code $LASTEXITCODE" }
}
catch {
    Write-Error "Terraform plan failed: $_"
    throw
}
```

### 3. Replace SilentlyContinue with Explicit Handling
**Before:**
```powershell
$ErrorActionPreference = "SilentlyContinue"
Remove-Item Env:\ARM_CLIENT_SECRET
$ErrorActionPreference = "Continue"
```

**After:**
```powershell
if (Test-Path Env:\ARM_CLIENT_SECRET) { Remove-Item Env:\ARM_CLIENT_SECRET }
```

### 4. Add LASTEXITCODE Checks
For every external command invocation in the script (including `terraform`, `az`, `hcl2json`, and any command run via `&` or `Start-Process`), add an explicit `$LASTEXITCODE` check immediately after the command. Do not add `$LASTEXITCODE` checks for PowerShell cmdlets or functions; only add them for native executables.
```powershell
if ($LASTEXITCODE -ne 0) {
    throw "$ToolName failed with exit code $LASTEXITCODE"
}
```

## Inputs
- Script content or file path
- Function names to modernize (optional). If function names are provided, update only those functions. If no function names are provided, update the entire script. If a named function is not found in the provided script, report that the function was not found and do not modify any other part of the script.

## Outputs
- Modernized error handling code
- List of changes with before/after snippets
