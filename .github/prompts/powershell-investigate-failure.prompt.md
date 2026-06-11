---
description: "Diagnose a PowerShell script failure and recommend fixes"
mode: "agent"
agent: "powershell-debugger"
---

# Investigate PowerShell Failure

Diagnose the root cause of a failure in a PowerShell script.

## Instructions

1. Gather the error context: script path, error message, and execution environment.
2. Read the failing script and all its dot-sourced dependencies.
3. Trace the execution path to identify where the failure occurs.
4. Apply diagnostic layers: syntax → runtime → external tools → authentication → dependencies.
5. State the root cause in one clear sentence with supporting evidence.
6. Recommend a specific fix.
7. If systemic anti-patterns are found, recommend handoff to the modernizer.

## Context

**Script:** `${input:scriptPath:Path to the failing .ps1 file}`

**Error message:** `${input:errorMessage:Paste the error message or describe the failure}`

**Environment:** `${input:environment:dev, test, snd, or prod}`
