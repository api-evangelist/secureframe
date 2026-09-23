---
name: secureframe-asset-framework-scoping
description: Scope cloud resources, devices and code repositories into or out of a compliance framework's audit in Secureframe. Use when preparing an asset inventory for a SOC 2, ISO 27001 or CMMC audit.
api: Secureframe Public API
base_url: https://api.secureframe.com
operations:
  - companyFrameworksIndex
  - cloudResourcesIndex
  - cloudResourcesCompanyFrameworkAssetScopesIndex
  - cloudResourcesCompanyFrameworkAssetScopesCreate
  - devicesIndex
  - devicesCompanyFrameworkAssetScopesCreate
  - repositoriesIndex
  - repositoriesCompanyFrameworkAssetScopesCreate
mcp_tools:
  - list_frameworks
  - list_cloud_resources
  - list_cloud_resource_framework_asset_scopes
  - create_cloud_resource_framework_asset_scope
  - list_devices
  - create_device_framework_asset_scope
  - list_repositories
  - create_repository_framework_asset_scope
---

# Scoping assets into a framework audit

Audit scope is the argument you have with an auditor before you have it. Secureframe
models it as a `framework_asset_scope` — a join between one asset and one framework —
and each asset type carries its own subresource for it.

## 1. Identify the framework

`companyFrameworksIndex` — `GET /frameworks`. Keep the framework id; every scope you
create is relative to it.

## 2. Enumerate the assets

Three parallel inventories, each with the same shape:

| Asset | List | Scope subresource |
|---|---|---|
| Cloud resource | `GET /cloud_resources` | `GET|POST /cloud_resources/{cloud_resource_id}/framework_asset_scopes` |
| Device | `GET /devices` | `GET|POST /devices/{device_id}/framework_asset_scopes` |
| Repository | `GET /repositories` | `GET|POST /repositories/{id}/framework_asset_scopes` |

`GET /cloud_resources` documents an enormous `type` enum — several hundred values from
`ec2_instance` and `s3_bucket` through `microsoft_sql_servers` and
`check_point_quantum_network`. Filter on it rather than pulling the whole inventory.

Relevant search parameters on these lists include `in_audit_scope`,
`manually_scoped_reason` and `out_of_audit_scope_reason` — read them before writing,
because they tell you whether an asset was scoped deliberately or by rule.

## 3. Read the existing scope before adding one

`cloudResourcesCompanyFrameworkAssetScopesIndex` —
`GET /cloud_resources/{cloud_resource_id}/framework_asset_scopes`. Creating a scope that
already exists is not idempotent and there is no upsert.

## 4. Create the scope

`cloudResourcesCompanyFrameworkAssetScopesCreate` —
`POST /cloud_resources/{cloud_resource_id}/framework_asset_scopes` (and the device /
repository equivalents).

## Why this flow is slow, and plan for it

This is the flow that runs headlong into three of Secureframe's published constraints at
once:

- **No bulk operations.** The reference states plainly: "Our API does not directly
  support bulk updates - only one object can be updated per request." Scoping an
  inventory is one POST per asset.
- **500 requests per minute per IP address.** Not per key, not per company — per IP. You
  cannot buy headroom by issuing more API keys, and every process behind one NAT egress
  shares the budget. A 10,000-asset inventory is at minimum 20 minutes of wall clock.
- **No idempotency key and no `Retry-After`.** A retried POST creates a duplicate scope,
  and 429 comes back with no header telling you when to resume.

Pace deliberately below the ceiling, checkpoint your progress against the scope
`Index` operation rather than against your own loop counter, and on 429 back off with
jitter.

## Also worth knowing

There is no un-scope operation in the contract. `PUT /cloud_resources/{id}` and
`PUT /repositories/{id}` update the asset itself; removing a framework asset scope is
not a published operation.
