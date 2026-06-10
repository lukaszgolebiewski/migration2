---
applyTo: "**/*.tf"
---

# Terraform Conventions

## Version & Providers

- Terraform `~> 1.14.0`. Do not downgrade or suggest older syntax.
- Pin every provider to an exact version (`=`). Current pins:
  - `azurerm = 4.62.1`
  - `azuread = 3.8.0`
  - `azuredevops = 1.13.0`
  - `azapi = 2.8.0`
  - `random = 3.8.1`
  - `null = 3.2.4`
  - `archive = 2.7.1`
  - `time = 0.13.1`
- Backend is always `azurerm` with Azure AD authentication (except bootstrap local-init).

## Language Rules

- **`for_each` over `count`** for every resource that needs stable identity across plan/apply cycles. `count` is acceptable only for truly disposable or indexed resources.
- **`terraform_data`** over `null_resource` for lifecycle triggers and replacements.
- **Dynamic blocks** guard optional nested config: `for_each = condition ? [1] : []`.
- **Flatten + map** for nested iteration. Flatten to a list of objects, then convert to a map with composite keys:
  ```hcl
  locals {
    items_list = flatten([for k, v in var.input : [for sub in v.children : { "${k}-${sub.name}" = { ... } }]])
    items_map  = { for item in local.items_list : keys(item)[0] => values(item)[0] }
  }
  ```
- **Avoid `splat` expressions** (`[*]`) when `for` expressions are clearer.
- Avoid `depends_on` unless there is a genuine hidden dependency. Prefer data flow (referencing attributes) to express ordering.

## Module Design

- **Two-layer module structure** inside `blueprint/modules/{name}-module/`:
  - `accounts/` — Core resource creation, diagnostic settings, network rules, RBAC assignments.
  - `services/` — Sub-resources (containers, shares, queues, tables, collections).
- Each layer has its own `variables.tf`, `output.tf`, and resource files.
- Module outputs should be **minimal** — expose only what callers actually need. Do not mirror every attribute.

## Variable Flow (Three Layers)

```
settings/{env}/{module}/  →  import.tfsettings-*.tf  →  blueprint/{module}/variables.tf
```

1. **Settings** export `locals` or `output` blocks with environment-specific values.
2. **Import modules** (`import.tfsettings-*.tf`) reference `../../settings/{env}/{module}` and pass values out.
3. **Blueprint resources** consume them via `module.tfsettings-{name}.{attribute}`.

Never bypass this chain. Blueprint code must never hardcode environment values.

## Import Templates

- Every `import.tfsettings-*.tf` must have a matching `.TEMPLATE` file.
- Templates use `###{env_shortname}###` as the placeholder (replaced at `terraform init` time).
- Example:
  ```hcl
  module "tfsettings-aca" {
    source            = "../../settings/###{env_shortname}###/aca"
    project_shortname = module.tfsettings-global.project.project_shortname
    env_shortname     = module.tfsettings-global.project.env_shortname
  }
  ```
- When creating a new module, generate both `import.tfsettings-{name}.tf` and `import.tfsettings-{name}.tf.TEMPLATE`.
- Always include `import.tfsettings-global.tf` (and its TEMPLATE) in every blueprint module — it provides `project_shortname`, `env_shortname`, networking, and subscription data.

## Settings Files

- Settings live in `settings/{env}/{module}/` as `.tf` files.
- When creating new settings, **generate all four environments**: `dev`, `test`, `snd`, `prod`.
- Settings must never contain resource blocks — only `locals`, `variable`, and `output`.
- Use the `project_shortname` and `env_shortname` variables passed from the global import for naming consistency.

## Provider Configuration

- The `provider.tf` block is duplicated per blueprint module (each module is a root module, not a child).
- Three standard provider aliases:
  - Default (backend subscription).
  - `project` (project subscription — where resources are deployed).
  - `backend` (state storage subscription).
- Key Vault features: `purge_soft_delete_on_destroy = false`, `recover_soft_deleted_key_vaults = true` (except backend alias).

## Formatting & Style

- Use `terraform fmt` canonical style (2-space indent, aligned `=`).
- One resource type per file where practical (e.g., `storage.tf`, `access.tf`, `private-endpoints.tf`).
- File names use lowercase kebab-case: `import.tfsettings-storage.tf`, `private-endpoints.tf`.
- Variable descriptions are mandatory. Type constraints are mandatory.
- Locals that build complex maps should include inline comments explaining the key structure.

## State & Backend

- State key pattern: `{env_shortname}.{subpipeline}.terraform.tfstate`.
- Backend config is supplied via `.backend.tfvars` files in `settings/{env}/bootstrap/`.
- Never hardcode backend details in `provider.tf`.

## Testing Expectations

- Every change must pass `terraform validate` and `terraform plan` without errors.
- Plans should show **no unexpected resource replacements** (stable `for_each` keys).
