# Azure Authentication Validation Skill

## Purpose
Validate Azure authentication patterns in PowerShell scripts. This validation covers service principal, managed identity, and subscription-context patterns. If the script uses a different authentication method (for example, interactive `az login` or device-code sign-in), mark the result as `not-applicable` rather than guessing.

If the script does not contain Azure authentication or subscription-selection logic, return `status=not-applicable` and state that no authentication pattern could be validated.

If the input is empty, the file cannot be read, or the content is not PowerShell, return one result with `status=error` and explain that validation could not be performed.

## When to Use
- During debugging (authentication failures are the root cause)
- During security review (validating auth flow compliance)

## Evaluation Procedure

Apply checks in this strict order. Report results per check.

1. **Input validity** — verify the input is readable PowerShell. If not, return `status=error` and stop.
2. **Subscription context** (Check 1) — verify explicit subscription selection.
3. **Credential source** (Check 2) — verify SP credentials come from Key Vault.
4. **Environment variable lifecycle** (Check 3) — verify ARM_* set/use/clean pattern.
5. **Managed identity preference** (Check 4) — apply only when the script runs in an Azure-hosted environment. If the script is intended for local development or CI/CD, service principal authentication is allowed and must not be flagged.
6. **Settings-driven configuration** (Check 5) — verify no hardcoded GUIDs.

## Validation Checks

### 1. Subscription Context
Require an explicit subscription selection before any Azure command that targets a subscription. Accept either `az account set -s $project_subscription_id`, `Set-AzContext -Subscription $project_subscription_id`, or `--subscription $project_subscription_id` on the command itself.
```powershell
# ✅ Correct — explicit subscription
az account set -s $project_subscription_id

# 🟠 Incorrect — assumes default subscription
az storage blob upload --account-name $name
```

### 2. Service Principal Authentication
Treat a service principal credential as compliant only when it is retrieved from Key Vault at runtime (for example, `az keyvault secret show --query value --output tsv`). Do not treat environment variables, pipeline secret variables, or other secret stores as compliant unless they are explicitly listed here.
```powershell
# ✅ Correct — from Key Vault
$pipeline_appid = (az keyvault secret show --name $secretName --vault-name $vaultName --query value --output tsv)

# 🔴 Incorrect — hardcoded or from config file
$pipeline_appid = "12345678-1234-1234-1234-123456789012"
```

### 3. Environment Variable Lifecycle
For each `ARM_*` variable assigned in the script, require at least one subsequent read of that variable before the script ends. Require cleanup (e.g., `Remove-Item Env:\ARM_*`) only for variables that are created for the current step and are not needed by later steps in the same script.
```powershell
# Check for: set → use → clean pattern
$env:ARM_CLIENT_ID = $appId          # SET
terraform -chdir="$root" plan        # USE
Remove-Item Env:\ARM_CLIENT_ID       # CLEAN ← must exist
```

### 4. Managed Identity Preference
Flag scripts as managed-identity candidates only when the script runs in an Azure-hosted execution environment (for example, Azure VM, App Service, Functions, Container Apps, AKS) and the script uses a service principal instead of `Connect-AzAccount -Identity`. Do not flag scripts intended for local development or CI/CD pipelines.
```powershell
# Candidate for managed identity (Azure-hosted execution)
Connect-AzAccount -ServicePrincipal -Credential $cred
# → Consider: Connect-AzAccount -Identity
```

### 5. Settings-Driven Configuration
Treat `tenant_id` and `subscription_id` as compliant only when they are loaded from an explicit settings source such as `project.tf` locals (via `hcl2json`), environment variables, or a named config file. Do not treat literal GUID strings as compliant.
```powershell
# ✅ Correct — from settings via hcl2json
$project_config = (hcl2json "$settings_root/settings/$env_shortname/global/project.tf" | ConvertFrom-Json).locals
$tenant_id = $project_config.project.tenant_id

# 🔴 Incorrect — hardcoded GUID
$tenant_id = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
```

## Common Failure Scenarios in This Repository

| Symptom | Root Cause | Fix |
|---|---|---|
| "AADSTS700016: Application not found" | Wrong tenant context | Verify `$tenant_id` from settings |
| "AuthorizationFailed" | SP missing RBAC role | Check role assignments in settings |
| "The subscription was not found" | Wrong `az account set` | Use `$project_subscription_id` from settings |
| "Access denied" on Key Vault | Missing Key Vault RBAC | Verify AD group membership |

## Inputs
- Script content or file path. If the file path is invalid or the file cannot be opened, return `status=error` and do not attempt partial validation.
- Error message (for debugging context, optional)

## Outputs
- Validation results per check category (pass, fail, not-applicable, or error)
- Specific remediation steps for failures
