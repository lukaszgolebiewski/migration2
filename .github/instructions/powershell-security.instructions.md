---
applyTo: "**/*.ps1"
---

# PowerShell Security Conventions

These rules supplement the constitution's security section with PowerShell-specific enforcement patterns. For general security principles, refer to `copilot-instructions.md`. For Azure resource security, refer to `terraform-security.instructions.md`.

## Credential Handling

- Never store credentials in variables longer than necessary. Clear sensitive variables immediately after use.
- Never pass secrets as visible parameters. Use `[SecureString]` or environment variables (`$env:*`).
- Never use `ConvertTo-SecureString -AsPlainText` with literal strings. Accept secure input or read from Key Vault.
- When retrieving secrets from Key Vault via `az keyvault secret show`, pipe through `--query value` and trim — never store the full JSON response.

## Output Safety

- Use `--output none` on mutating Azure CLI calls to prevent leaking response payloads.
- Never write tokens, keys, connection strings, or passwords to `Write-Host`, `Write-Verbose`, or `Write-Output`.
- Sanitize error messages before logging — `$_.Exception.Message` may contain embedded credentials from HTTP responses.
- Avoid `Write-Debug` with sensitive context in shared pipelines — debug streams may be captured.

## External Command Execution

- Never use `Invoke-Expression` (`iex`) with user-supplied or variable-interpolated strings.
- Prefer splatting and named parameters over string concatenation for command arguments.
- Validate all file paths before passing to `Get-Content`, `Set-Content`, or `Remove-Item` — prevent path traversal.
- When downloading tools (e.g., `Invoke-WebRequest`), validate the source URL and consider hash verification.

## Authentication Flows

- Prefer managed identity authentication over service principal secrets.
- When service principal auth is required, retrieve credentials from Key Vault at runtime — never from config files.
- Clear `$env:ARM_CLIENT_SECRET` and similar environment variables after Terraform execution completes.
- Use `Connect-AzAccount -Identity` where Azure-hosted execution supports it.

## Pipeline Security

- Set `$ErrorActionPreference = 'Stop'` to prevent silent continuation past security-relevant failures.
- Never suppress errors from authentication or authorization operations with `-ErrorAction SilentlyContinue`.
- Validate `$LASTEXITCODE` after every external tool call — treat non-zero as a security-relevant event.
- Do not hardcode pipeline PAT tokens. Use `$env:SYSTEM_ACCESSTOKEN` or `$env:AZDO_PERSONAL_ACCESS_TOKEN`.
