---
applyTo: "**/*.tf"
---

# Terraform Security Conventions

These rules supplement the constitution's security section with Terraform-specific enforcement patterns.

## Diagnostic Settings

- **Every deployable resource** must have an `azurerm_monitor_diagnostic_setting` wired to Log Analytics.
- Reference the workspace via data source: `data.azurerm_log_analytics_workspace.main`.
- Standard naming: `{resource_prefix}-{project_shortname}-{key}-{env_shortname}-diagnostic-law`.
- Enable at minimum: `category_group = "allLogs"`, `category_group = "audit"`, and `Transaction` metrics.
- Never remove, skip, or conditionally disable diagnostic settings.

## Network Access

- Default posture: `public_network_access_enabled = false`.
- The `var.public_access_enabled` toggle exists only for initial provisioning. It must gate network rules:
  ```hcl
  default_action = var.public_access_enabled ? "Allow" : each.value.network_rules.default_action
  ```
- Never set `public_network_access_enabled = true` in settings files without explicit user justification.
- Apply network rules with `default_action = "Deny"` and explicit `ip_rules` / `virtual_network_subnet_ids`.
- `bypass = ["AzureServices"]` when the resource needs internal Azure connectivity.

## Private Endpoints

- Private Endpoints are the **default** connectivity method for all PaaS resources.
- PE resource naming: follows the resource's naming pattern with `-pe` suffix.
- PE resource group: `rg-{project_shortname}-{module}-pe-{env_shortname}`.
- Each module's `private-endpoints.tf` manages PE creation and DNS zone linking.
- Never remove a private endpoint without adding a replacement connectivity path.

## RBAC & Access Control

- **AD group-based RBAC** is the access model. Use `azuread_group` data sources + `azurerm_role_assignment`.
- Never assign roles directly to individual user or service principal object IDs — always go through groups.
- Use the flatten + map pattern for role assignments:
  ```hcl
  locals {
    role_list = flatten([for k, v in var.ad_groups : [for role, groups in v["account_roles"] : ...]])
    role_map  = { for item in local.role_list : keys(item)[0] => values(item)[0] }
  }
  ```
- Role definition names use human-readable strings (`"Storage Blob Data Reader"`, not role IDs) unless a custom role is required.
- Managed identity role assignments follow the same pattern via `var.user_assigned_identities`.

## Managed Identities

- Prefer **user-assigned managed identities** over system-assigned for cross-resource access.
- Identity naming: `id-{project_shortname}-{purpose}-{env_shortname}`.
- Identities are defined in settings and provisioned in the `core/` module, then referenced by other modules.
- Never create service principal secrets for workload authentication — use managed identity federation.

## Key Vault

- `purge_soft_delete_on_destroy = false` — always.
- `recover_soft_deleted_key_vaults = true` on the default and project provider aliases.
- Store all generated credentials (SP passwords, connection strings, keys) in Key Vault immediately.
- Reference secrets via `azurerm_key_vault_secret` data sources — never pass secrets through outputs or variables in plaintext.
- RBAC mode (`enable_rbac_authorization = true`) is preferred over access policies.

## TLS & Encryption

- `min_tls_version = "TLS1_2"` on all resources that support it.
- Enable infrastructure encryption where available (`infrastructure_encryption_enabled = true`).
- Never disable HTTPS-only settings.

## Tagging

- Every resource must include at minimum: `environment = var.env_shortname`.
- Additional tags follow the project tagging policy defined in settings.

## Defender & Security Center

- Enable Microsoft Defender recommendations for supported resource types when available.
- Do not suppress or ignore Defender findings without explicit user instruction.
