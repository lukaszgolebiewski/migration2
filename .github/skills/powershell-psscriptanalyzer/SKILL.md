# PSScriptAnalyzer Skill

## Purpose
Run and interpret PSScriptAnalyzer findings against repository PowerShell standards. Maps analyzer rules to project conventions and produces actionable recommendations.

If Invoke-ScriptAnalyzer returns no findings, respond with: `No PSScriptAnalyzer findings were found. Summary: 0 errors, 0 warnings, 0 information.` Do not produce the structured findings list for empty results.

## When to Use
- Before modernizing a script (pre-analysis baseline)
- After generating new code (compliance validation)
- During debugging (identifying static issues)

## Procedure

### 1. Check Availability
```powershell
if (-not (Get-Module -ListAvailable PSScriptAnalyzer)) {
    Install-Module -Name PSScriptAnalyzer -Scope CurrentUser -Force
}
```
If `Install-Module` fails or `Invoke-ScriptAnalyzer` throws an exception, stop immediately, report the exact error text, and do not fabricate findings.

### 2. Run Analysis
If `$TargetPath` is a file, run without `-Recurse`. If `$TargetPath` is a directory, run with `-Recurse`. Use `$Severity` to filter; when no severity filter is provided, include all levels.
```powershell
# Single file
$results = Invoke-ScriptAnalyzer -Path $TargetPath -Severity $Severity

# Directory
$results = Invoke-ScriptAnalyzer -Path $TargetPath -Recurse -Severity $Severity
```

### 3. Map Findings to Repository Standards

| PSScriptAnalyzer Rule | Repository Standard | Action |
|---|---|---|
| `PSAvoidUsingWriteHost` | `powershell.instructions.md` — Write-Verbose preference | Replace with Write-Verbose |
| `PSUseCmdletCorrectly` | `powershell.instructions.md` — approved verbs | Rename function |
| `PSAvoidUsingPlainTextForPassword` | `powershell-security.instructions.md` — credential handling | Use SecureString |
| `PSUseShouldProcessForStateChangingFunctions` | `powershell.instructions.md` — CmdletBinding | Add SupportsShouldProcess |
| `PSAvoidUsingInvokeExpression` | `powershell-security.instructions.md` — injection prevention | Refactor to direct invocation |
| `PSAvoidGlobalVars` | `powershell.instructions.md` — scope management | Use script or local scope |
| `PSUseDeclaredVarsMoreThanAssignments` | `copilot-instructions.md` — no dead code | Remove unused variables |

### 4. Output Format
Return results as a structured list:
```
Rule: <RuleName>
Severity: Error|Warning|Information
File: <path>
Line: <number>
Message: <description>
Repository Standard: <exact file name and section heading, e.g. powershell.instructions.md — Scope Management>
Recommended Fix: <one-line action>
```

## Inputs
- `$TargetPath` — path to a single file or to a directory. If `$TargetPath` is a file, run `Invoke-ScriptAnalyzer -Path $TargetPath`. If `$TargetPath` is a directory, run `Invoke-ScriptAnalyzer -Path $TargetPath -Recurse`.
- `$Severity` — one or more of `Error`, `Warning`, `Information`. Default: `@('Error', 'Warning', 'Information')` (all levels).

## Outputs
- Structured findings list mapped to repository standards
- Summary counts by severity and category
