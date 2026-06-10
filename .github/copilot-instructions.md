# Repository Constitution — Copilot Instructions

> These rules are absolute. They apply to every file, every language, and every interaction in this repository. Domain-specific instructions extend — never override — this document.

## Repository Purpose

This repository manages Azure infrastructure using Terraform in a **blueprint + settings** architecture:

- **`blueprint/`** — Environment-agnostic infrastructure code: reusable modules, resource definitions, helpers, and pipeline templates.
- **`settings/`** — Environment-specific configuration data (`dev`, `test`, `snd`, `prod`). Settings are consumed by blueprint modules via `import.tfsettings-*.tf` files.

The two layers are deliberately separated. Blueprint code must never contain environment-specific values. Settings must never contain resource logic.

## Security — Non-Negotiable

1. **Never hardcode secrets, keys, passwords, tokens, or connection strings.** Store them in Key Vault; reference via data sources or module outputs.
2. **Never disable or remove diagnostic settings.** Every resource must wire to Log Analytics.
3. **Never weaken network restrictions** (e.g., setting `public_access_enabled = true`) unless the user explicitly requests it with justification.
4. **Prefer managed identities** over service principal secrets for runtime authentication.
5. **Prefer RBAC** over access policies (Key Vault, Storage, etc.).
6. **Private Endpoints are the default.** Public access is the exception, controlled by explicit toggles.
7. **AD group-based RBAC** is the access model. Do not assign roles directly to individual identities.

## Code Quality Principles

- **Idempotency is mandatory.** Every solution must be safe to apply repeatedly without side effects.
- **Backward compatibility by default.** Never rename resources, variables, or outputs unless the user explicitly instructs a breaking change.
- **Minimal blast radius.** Prefer targeted changes over sweeping refactors.
- **No dead code.** Do not leave commented-out blocks, unused variables, or orphaned outputs.
- **Descriptive naming.** Names must convey purpose. Avoid abbreviations unless they are project-standard (see naming conventions in `azure.instructions.md`).

## Architecture Rules

- **`for_each` over `count`** for resources that require stable identity across plan/apply cycles.
- **Flatten + map** pattern for nested iteration (flatten to list, then convert to map with unique keys).
- **Dynamic blocks** for optional nested configuration — guard with `for_each = condition ? [1] : []`.
- **Three-layer variable flow**: Settings → `import.tfsettings-*.tf` → Feature module variables. Never bypass this chain.
- **Two-layer module design**: `accounts/` for core resource + access control, `services/` for sub-resources (containers, shares, queues).

## File & Folder Conventions

| Layer | Contains | Example |
|---|---|---|
| `blueprint/{module}/` | Resource definitions, provider config, variables, imports | `blueprint/storage/` |
| `blueprint/modules/` | Reusable child modules called by blueprint layers | `blueprint/modules/storage/` |
| `blueprint/helpers/` | PowerShell utility scripts | `helpers/install-prerequisites.ps1` |
| `blueprint/templates/` | Azure DevOps YAML pipeline templates | `templates/terraform-plan-apply.yml` |
| `settings/{env}/{module}/` | `.tf` files exporting environment-specific locals/variables | `settings/dev/storage/` |

## What Copilot Must Always Do

1. **Preserve the blueprint ↔ settings separation.** Configuration values go in `settings/`; logic goes in `blueprint/`.
2. **Follow existing patterns.** Before generating new code, study adjacent files in the same module for style, naming, and structure.
3. **Respect the import mechanism.** New modules must include `import.tfsettings-*.tf` and corresponding `.TEMPLATE` files with `###env_shortname###` placeholders.
4. **Never generate all four environment variants** when creating settings files (`dev`, `test`, `snd`, `prod`) — only the one specified by the user. 
5. **Explain trade-offs** when multiple valid approaches exist.
6. **Plan before agent execution.** For any change proposed to be performed by an agent/subagent, Copilot must first present a concrete plan (scope, files, risks, rollback) and wait for explicit user approval.
7. **No implicit execution.** Copilot must not run agent/subagent implementation steps until the user explicitly accepts the plan.
8. **If no approval, no changes.** Without explicit approval, only analysis/clarification is allowed.