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
- All resources wire diagnostics to the main Log Analytics workspace in their environment (e.g., `la-pada-iadwh-main-dev`).
- Pipeline health checks are handled by the `scheduled-pinging.yml` template.

## Azure DevOps Pipelines

- Template location: `blueprint/templates/`.
- Variables use `${placeholder}` syntax (substituted at pipeline generation time).
- Two-stage pattern: `Terraform_PLAN` → `Terraform_APPLY` (with manual approval gate).
- Artifacts: `.tfplan` files published between stages.
- All pipeline steps run under the project service principal via `AzureCLI@2`.

## Well-Architected Framework (WAF) Compliance

Every Azure resource should be reviewed against the five WAF pillars. Use these as a checklist when creating or modifying infrastructure:

### Reliability
- **Availability Zones**: Use zone-redundant SKUs for production-critical resources (e.g., `ZRS`/`GZRS` for storage, zone-redundant App Service plans, AKS with zone spread).
- **Health probes**: Configure health check endpoints for load-balanced and app-hosting resources.
- **Multi-region readiness**: Document geo-replication or failover strategy for resources that support it. Active-active is preferred over active-passive where cost allows.
- **BGP / dynamic routing**: Use dynamic routing over static for VPN and ExpressRoute gateways.

### Cost Optimization
- **SKU right-sizing**: Select the lowest-cost SKU that meets performance and SLA requirements. Document the rationale when choosing premium tiers.
- **Expensive resources should be optional**: Gate costly components (Firewall, VPN Gateway, Bastion) behind feature toggles so non-production environments can omit them.
- **Cost tracking tags**: Every resource must include tags enabling cost attribution (see Tagging Strategy below).
- **Start/stop schedules**: Apply to non-production compute resources (VMs, Batch pools) — handled via the existing scheduling pipeline.

### Performance Efficiency
- **Autoscaling**: Enable autoscale for resources that support it (VMSS, App Service, AKS, Container Apps). Define sensible min/max/default values.
- **Accelerated networking**: Enable on VMs and VMSS where the SKU supports it.
- **Service endpoints**: Use service endpoints or private endpoints to minimise network hops and latency to PaaS services.

### Operational Excellence
- **NSG Flow Logs**: Enable for all NSGs in production to support traffic analysis and troubleshooting.
- **Infrastructure documentation**: Every module should include a brief header comment or README describing its purpose, dependencies, and expected WAF posture.
- **Input validation**: Use `validation {}` blocks on variables with known constraints (e.g., allowed SKU values, CIDR ranges, naming patterns).

> **Note:** Security and Operational Excellence diagnostic requirements are defined in `terraform-security.instructions.md`. Do not duplicate them here.

## Tagging Strategy

All resources must include the following tags at minimum:

| Tag | Source | Purpose |
|-----|--------|---------|
| `environment` | `var.env_shortname` | Environment identification |
| `project` | `var.project_shortname` | Project attribution |
| `managed-by` | `"terraform"` | Identifies IaC-managed resources |

Additional recommended tags (define in settings when applicable):

| Tag | Purpose |
|-----|---------|
| `owner` | Team or individual accountable for the resource |
| `cost-center` | Financial cost attribution |

Tags are defined in settings and applied consistently via module variables. Never hardcode tag values in blueprint code.

## CAF Compliance Checklist

When adding any new Azure resource, verify:

- [ ] **Naming** follows the conventions table above. If the resource type is not listed in the naming table, do not invent a naming pattern. Stop and ask for the correct convention or use the nearest approved pattern documented in the project.
- [ ] **Tags** include at least `environment`, `project`, and `managed-by`.
- [ ] **Managed identity** is used instead of service principal secrets for runtime auth.
- [ ] **Private endpoint** is configured (public access disabled by default).
- [ ] **Diagnostic setting** wires to Log Analytics (see `terraform-security.instructions.md`).
- [ ] **Encryption** at rest and in transit is enabled (TLS 1.2+, infrastructure encryption where available).
- [ ] **RBAC** via AD groups — no direct individual identity assignments.
- [ ] **Availability zones** are enabled for production-critical resources.
- [ ] **Resource organisation** places the resource in the correct subscription and resource group.

## Azure Verified Modules

- Prefer [Azure Verified Modules](https://azure.github.io/Azure-Verified-Modules/) when starting a new module, unless existing project patterns conflict.
- When suggesting a verified module, confirm it aligns with the project's provider version pins before recommending.
