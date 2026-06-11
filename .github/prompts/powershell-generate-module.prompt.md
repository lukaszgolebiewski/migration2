---
description: "Generate a new PowerShell function, module, or helper script"
mode: "agent"
agent: "powershell-generator"
---

# Generate PowerShell Module

Generate a new PowerShell artifact for the repository.

## Instructions

1. Confirm the artifact type, purpose, and target location.
2. Study adjacent files in the target directory for pattern matching.
3. Present the proposed file structure and function signatures.
4. Wait for my approval before creating files.
5. Generate the code with full compliance to repository standards.
6. Generate a companion Pester test file.

## Requirements

**Artifact type:** `${input:artifactType:function, module, script, helper, or pipeline-stage}`

**Purpose:** `${input:purpose:Describe what the code should do}`

**Target location:** `${input:location:e.g., blueprint/helpers/ or blueprint/local-pipeline/}`
