---
description: "Modernize a PowerShell script to enterprise standards"
mode: "agent"
agent: "powershell-modernizer"
---

# Modernize PowerShell Script

Analyze and modernize the specified PowerShell script.

## Instructions

1. Read the target script completely.
2. Run PSScriptAnalyzer against it (if available).
3. Identify all anti-patterns against repository standards.
4. Present a numbered modernization plan with estimated changes per function.
5. Wait for my approval before making any changes.
6. After changes, generate a migration report.

## Target

Modernize: `${input:scriptPath:Path to the .ps1 file to modernize}`

## Focus Areas
- CmdletBinding and parameter validation
- Approved verbs and naming conventions
- Write-Verbose over Write-Host
- Error handling with try/catch and $LASTEXITCODE
- Splatting for multi-parameter commands
- No aliases in scripts
