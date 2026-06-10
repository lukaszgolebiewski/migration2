---
applyTo: "**/*.{tf,yml,yaml,ps1,json}"
---

# Azure Platform Conventions

## Naming Conventions (CAF-Aligned)

All resources follow this general pattern:

```
{prefix}-{project_shortname}-{key}-{env_shortname}
```

| Resource Type | Prefix | Pattern | Example |
|---|---|---|---|
| Resource Group | `rg-` | `rg-{project}-{key}-{env}` | `rg-pada-iadwh-storage-dev` |
| Storage Account | `sa` | `sa{project}{key}{env}` (no dashes, lowercase) | `sapadaiadwhadfdatadev` |
| Key Vault | `kv-` | `kv-{project}-{env}` | `kv-pada-iadwh-dev` |
| Log Analytics | `la-` | `la-{project}-{key}-{env}` | `la-pada-iadwh-main-dev` |
| App Insights | `appi-` | `appi-{project}-{key}-{env}` | `appi-pada-iadwh-main-dev` |
| Managed Identity | `id-` | `id-{project}-{key}-{env}` | `id-pada-iadwh-adf-dev` |
| Container App | `aca-` | `aca-{project}-{key}-{env}` | `aca-pada-iadwh-api-dev` |
| Container Registry | `acr` | `acr{project}{env}` (no dashes) | `acrpadaiadwhdev` |
| Data Factory | `adf-` | `adf-{project}-{key}-{env}` | `adf-pada-iadwh-main-dev` |
| Batch Account | `ba` | `ba{project}{key}{env}` (no dashes) | `bapadaiadwhcompdev` |
| AI Search | `srch-` | `srch-{project}-{key}-{env}` | `srch-pada-iadwh-main-dev` |
| Private Endpoint | — | `{parent_name}-pe` | `sapadaiadwhdatadev-pe-blob` |
| PE Resource Group | `rg-` | `rg-{project}-{module}-pe-{env}` | `rg-pada-iadwh-adf-pe-dev` |
| Diagnostic Setting | — | `{resource_name}-diagnostic-law` | `sa-pada-iadwh-data-dev-diagnostic-law` |

**Key rules:**
- Storage accounts, batch accounts, and container registries cannot contain dashes — strip them.
- `project_shortname` and `env_shortname` are sourced from global settings, never hardcoded.
- When inventing a `{key}`, keep it short (3–8 chars) and descriptive (`main`, `data`, `comp`, `api`).

## AD Groups

- Pattern: `!PL_PA_DA.{ENV}.{SCOPE}.{ROLE}` (e.g., `!PL_PA_DA.DEV.DATA.READER`).
- Defined in settings, consumed by RBAC modules.
- Never create or reference individual user identities for production role assignments.

## Service Principals & DevOps

- SP naming: `{project_shortname}-DevOps-Terraform-{Env}` (e.g., `pada-iadwh-DevOps-Terraform-Dev`).
- SP credentials stored as Key Vault secrets immediately on creation.
- Azure DevOps service connections reference the SP, not raw credentials.
- Agent pools: `{project_shortname}-{env_shortname}-DeployWin`.

## Subscription & Network Architecture

- **Multi-subscription model**: Backend (state) and project (resources) may be in different subscriptions.
- VNet/Subnet names follow corporate convention: `WE-{ORG}-{ENV}-ISN-VNET{nn}-{suffix}`.
- Subnet IDs are resolved via `data.azurerm_subnet.subnet` in global settings — never hardcode network resource IDs.
- Region: `westeurope` unless explicitly specified otherwise.

## Environment Hierarchy

| Environment | Purpose | Access Model |
|---|---|---|
| `dev` | Active development, permissive access | All DevOps engineers, home IP access |
| `test` | Integration testing, mirrors prod config | Restricted, no home IP access |
| `snd` | Sandbox/experimental features | Feature toggles enabled liberally |
| `prod` | Production workloads | Locked down, minimal public access |

- Feature toggles in settings control per-environment behavior: `azure_devops_agent`, `start_stop_rules`, `allow_devops_engineer_access_from_home`.
- `dev` enables most features; `prod` is the most restrictive.

## Resource Lifecycle

- **Soft delete** is always enabled on Key Vaults. Never auto-purge.
- **Start/stop rules** for VMs are managed via Azure Storage tables and scheduled pipelines.
- **Storage key rotation** is handled by a scheduled pipeline — do not implement manual rotation logic.

## Monitoring & Observability

- Every resource group has a Log Analytics workspace and App Insights instance provisioned in the `core/` module.
- All resources wire diagnostics to `la-{project_shortname}-main-{env_shortname}`.
- Pipeline health checks are handled by the `scheduled-pinging.yml` template.

## Azure DevOps Pipelines

- Template location: `blueprint/templates/`.
- Variables use `${placeholder}` syntax (substituted at pipeline generation time).
- Two-stage pattern: `Terraform_PLAN` → `Terraform_APPLY` (with manual approval gate).
- Artifacts: `.tfplan` files published between stages.
- All pipeline steps run under the project service principal via `AzureCLI@2`.

## Azure Verified Modules

- Prefer [Azure Verified Modules](https://azure.github.io/Azure-Verified-Modules/) when starting a new module, unless existing project patterns conflict.
- When suggesting a verified module, confirm it aligns with the project's provider version pins before recommending.
