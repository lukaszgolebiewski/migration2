# Secret Handling Review Skill

## Purpose
Flag only likely credentials or secrets in PowerShell scripts, including passwords, API keys, tokens, access keys, client secrets, PATs, connection strings, and certificates; do not flag ordinary variable names or non-secret identifiers.

Do not flag sample, test, or documentation values that are clearly illustrative; only analyze executable PowerShell statements and real variable assignments, not comments, examples, or placeholder values.

Also flag assignments to hashtable entries, object properties, and splatted parameter tables when the key or property name matches secret-like patterns or when the assigned literal is a credential-like value.

Also flag any command that writes a secret-bearing value to a file, log, transcript, JSON/XML output, or other output stream, not only `Write-Host` and `Write-Output`. This includes `Out-File`, `Set-Content`, `Add-Content`, `Export-CliXml`, `ConvertTo-Json`, `Start-Transcript`, `Write-Verbose`, or command output piped to `Format-List`.

Also inspect other CLI and PowerShell commands for secret-bearing parameters such as `-Password`, `-Token`, `-ApiKey`, `-ClientSecret`, and `-Credential`, not only `az`.

## When to Use
- During security review (primary detection tool)
- After modernization that touches authentication code

## Detection Patterns

### 1. Plaintext Credentials in Variables
```powershell
# 🔴 CRITICAL — plaintext password in variable
$password = "mySecret123"
$securePassword = ConvertTo-SecureString -String "$($env:servicePrincipalKey)" -AsPlainText -Force
```
**Remediation:** Read from Key Vault at runtime; never materialize as plaintext.

### 2. Secrets in Output Streams
```powershell
# 🔴 CRITICAL — secret written to console
Write-Host "SP Password: $pipeline_password"

# 🟠 HIGH — full JSON response may contain keys
$result = az keyvault secret show --name $secretName
Write-Host $result
```
**Remediation:** Use `--query value --output tsv` and never log the value.

### 3. Credentials in Parameters
```powershell
# 🟠 HIGH — visible in process list
az login --service-principal -u $appId -p $password --tenant $tenantId
```
**Remediation:** Use `$env:AZURE_CLIENT_SECRET` or certificate-based auth.

### 4. Uncleared Environment Variables
```powershell
# 🟡 MEDIUM — secrets persist after execution
$env:ARM_CLIENT_SECRET = $pipeline_password
# ... terraform runs ...
# (no cleanup)
```
**Remediation:** Clear in a `finally` block:
```powershell
try { terraform apply $plan }
finally { Remove-Item Env:\ARM_CLIENT_SECRET -ErrorAction SilentlyContinue }
```

### 5. Unsafe Download Patterns
```powershell
# 🟡 MEDIUM — no integrity verification
Invoke-WebRequest -Uri $url -OutFile $path
```
**Remediation:** Verify file hash after download.

## Scanning Approach

### Two-Step Detection Algorithm

**Step 1:** Detect every matching rule for each statement.  
**Step 2:** Choose severity = highest severity among those matches; evidence = all matched rule names.

### Rule Reference (Evaluation Order)

1. **Direct secret materialization** — Flag a variable name when its identifier contains, case-insensitively, one of these substrings: `password`, `secret`, `token`, `credential`, `clientsecret`, `accesskey`, or `pat`. Do not flag generic names such as `key`, `value`, `name`, or `id` unless the assigned literal is clearly sensitive.
2. **Secret output/logging** — Check all `Write-Host` / `Write-Output` calls for variable interpolation containing secret-like values.
3. **Secret arguments** — Check all `az` calls for `-p` or `--password` parameters (do not flag `--query` or `--output` parameters).
4. **Uncleaned environment variables** — Flag any assignment to an environment variable whose name contains `secret`, `password`, `token`, `key`, or `credential` when there is no later `Remove-Item Env:<VariableName>` for that same variable in the same function or in a `try/finally` block that covers the assignment.
5. **Unsafe download patterns** — Check for `Invoke-WebRequest` without hash verification.
6. **Safe secret retrieval** — Do NOT flag secret retrieval commands that read from a secure store and do not write the secret to output, such as `az keyvault secret show --query value --output tsv`, unless the retrieved value is written to the console, a log file, a transcript, or returned to the caller in the same statement or in a later statement.

## Error Handling

- If the input is empty, unreadable, or not valid PowerShell source, return one error finding with severity `error`, location `input`, evidence `Invalid or unreadable input`, and remediation `Provide valid PowerShell script content or a readable file path.`
- If the input path cannot be read, or the input is not valid PowerShell source, return one error finding and do not attempt a partial scan.
- If PowerShell parsing fails or the script contains syntax errors, return one error finding with severity `error`, location `input`, evidence `Invalid or unreadable PowerShell source`, and do not attempt a partial scan.
- If no findings are present, return an empty findings list and state `No credential exposure patterns found.`

## Inputs
- Script content or file path

## Outputs
- Findings list with severity, location, evidence, and remediation
