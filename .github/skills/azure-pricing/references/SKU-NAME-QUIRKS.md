# SKU Name Quirks Reference

The Azure Retail Prices API stores `skuName` values in formats that often differ from the ARM SKU name used in portal, Bicep, Terraform, and the `sku` parameter of the MCP pricing tool. This file consolidates known mismatches as a quick lookup before constructing tool queries.

## Quick Lookup Table

| Service | ARM / Portal / IaC SKU | API `skuName` | Query Approach |
|---------|------------------------|---------------|----------------|
| **App Service** | `P1v3` | `P1 v3` | Use `filter: "skuName eq 'P1 v3'"` |
| **App Service** | `P2v3` | `P2 v3` | Use `filter: "skuName eq 'P2 v3'"` |
| **App Service** | `P3v3` | `P3 v3` | Use `filter: "skuName eq 'P3 v3'"` |
| **App Service** | `P1mv3` | `P1mv3` | Use `filter: "skuName eq 'P1mv3'"` |
| **App Service** | `S1` | `S1` | Works directly as `sku` |
| **App Service** | `B1` | `B1` | Works directly as `sku` |
| **Azure SQL Database** | `GP_Gen5_2` | `2 vCore` | Use `filter` — see SQL section below |
| **Azure SQL Database** | `GP_Gen5_4` | `4 vCore` | Use `filter` — see SQL section below |
| **Azure SQL Database** | `GP_Gen5_8` | `8 vCore` | Use `filter` — see SQL section below |
| **Azure SQL Database** | `BC_Gen5_4` | `4 vCore` | Use `filter` with `productName` containing `Business Critical` |
| **Azure SQL Database** | `GP_S_Gen5_2` | `2 vCore` | Use `filter` with `productName` containing `Serverless` |
| **Azure Cache for Redis** | `C0` (Basic/Standard) | `C0 Cache Instance` | Use `filter` — see Redis section below |
| **Azure Cache for Redis** | `C1` (Basic/Standard) | `C1 Cache Instance` | Use `filter` — see Redis section below |
| **Azure Cache for Redis** | `C2` (Basic/Standard) | `C2 Cache Instance` | Use `filter` — see Redis section below |
| **Azure Cache for Redis** | `P1` (Premium) | `P1 Cache Instance` | Use `filter` — see Redis section below |
| **Azure Cache for Redis** | `P2` (Premium) | `P2 Cache Instance` | Use `filter` — see Redis section below |
| **API Management** | `Developer` | `Developer` | Works directly as `sku` |
| **API Management** | `Basic` | `Basic` | Works directly as `sku` |
| **API Management** | `Standard` | `Standard` | Works directly as `sku` |
| **API Management** | `Premium` | `Premium` | Works directly as `sku`; per-unit pricing — see APIM section below |
| **Service Bus** | `Basic` | `Basic` | Works; priced per operation |
| **Service Bus** | `Standard` | `Standard` | Works |
| **Service Bus** | `Premium` | `Premium` | Per messaging unit — see Service Bus section below |
| **Azure Container Apps** | N/A | N/A | `service` parameter not supported — use OData `filter` only |
| **Virtual Machines** | `Standard_D4s_v5` | `Standard_D4s_v5` | Works directly as `sku` — no quirks |
| **Storage Account** | `Standard_LRS` | `LRS` | Use `filter: "skuName eq 'LRS' and serviceName eq 'Storage'"` |
| **Storage Account** | `Standard_GRS` | `GRS` | Use `filter` |
| **Storage Account** | `Standard_ZRS` | `ZRS` | Use `filter` |
| **Storage Account** | `Premium_LRS` | `Premium LRS` | Use `filter: "skuName eq 'Premium LRS'"` |
| **PostgreSQL Flexible** | `Standard_D4s_v3` | `D4s v3` | Use `filter: "skuName eq 'D4s v3' and serviceName eq 'Azure Database for PostgreSQL'"` |

---

## Service-Specific Detail

### App Service

**Root cause:** All multi-character App Service plan tier names store the version suffix with a space in the API (`P2v3` → `P2 v3`, `P1mv3` stays `P1mv3`).

**Rule of thumb:** If the ARM name ends in `v2` or `v3`, insert a space before `v`. `B1`, `B2`, `S1`, `S2` tiers have no version suffix and work directly.

```
# Works — no version suffix
sku: "B1"      → API skuName: "B1"
sku: "S2"      → API skuName: "S2"

# Fails — missing space
sku: "P2v3"    → no results

# Correct — use filter
filter: "skuName eq 'P2 v3' and serviceName eq 'Azure App Service' and armRegionName eq 'uksouth'"
```

Multiply the returned hourly `retailPrice` × 730 for monthly cost.

---

### Azure SQL Database

**Root cause:** The API uses plain English vCore counts as `skuName`, not ARM tier codes. Multiple tiers (General Purpose, Business Critical, Serverless, Hyperscale) all return `skuName: "4 vCore"` — the tier is differentiated only by `productName`.

**ARM → API mapping:**

| ARM SKU | `skuName` | `productName` contains |
|---------|-----------|------------------------|
| `GP_Gen5_2` | `2 vCore` | `General Purpose - Compute Gen5` |
| `GP_Gen5_4` | `4 vCore` | `General Purpose - Compute Gen5` |
| `GP_Gen5_8` | `8 vCore` | `General Purpose - Compute Gen5` |
| `BC_Gen5_4` | `4 vCore` | `Business Critical - Compute Gen5` |
| `GP_S_Gen5_2` | `2 vCore` | `General Purpose - Serverless` |
| `HS_Gen5_4` | `4 vCore` | `Hyperscale - Compute Gen5` |

**Recommended filter (General Purpose, 4 vCore):**
```
filter: "skuName eq '4 vCore' and contains(productName, 'General Purpose - Compute Gen5') and armRegionName eq 'uksouth'"
```

**Always run two queries — compute and storage are separate meters:**
1. Compute: filter by `skuName` + `productName` as above
2. Storage: `filter: "contains(productName, 'General Purpose - Storage') and serviceName eq 'SQL Database' and armRegionName eq 'uksouth'"`

Storage is priced per GB/month, not hourly.

---

### Azure Cache for Redis

**Root cause:** The `sku` parameter does not resolve Redis SKUs correctly. The API stores them as `C0 Cache Instance`, `C1 Cache Instance`, etc. — not `C0` or `C1`.

**Terraform → API mapping:**

| Terraform `sku_name` | `family` | `capacity` | API `skuName` |
|----------------------|----------|------------|---------------|
| `Basic` | `C` | `0` | `C0 Cache Instance` |
| `Basic` | `C` | `1` | `C1 Cache Instance` |
| `Standard` | `C` | `1` | `C1 Cache Instance` |
| `Standard` | `C` | `2` | `C2 Cache Instance` |
| `Premium` | `P` | `1` | `P1 Cache Instance` |
| `Premium` | `P` | `2` | `P2 Cache Instance` |

> Basic and Standard tiers share the same `skuName` format — they're differentiated by `productName` (`Azure Cache for Redis Basic` vs `Azure Cache for Redis Standard`).

**Recommended filter (Standard C1):**
```
filter: "skuName eq 'C1 Cache Instance' and contains(productName, 'Standard') and serviceName eq 'Redis Cache' and armRegionName eq 'uksouth'"
```

---

### API Management

**SKU names match** — `Developer`, `Basic`, `Standard`, `Premium` all work directly as `sku`. However:

- **Premium pricing is per gateway unit** (not a flat rate). Each Premium unit is priced per hour. Multi-region deployments multiply cost by unit count per region.
- **Consumption tier** is priced per million API calls — `service: "API Management"` with `sku: "Consumption"` returns the per-call rate, not a flat hourly rate.
- **Developer tier** is not SLA-backed and should only be used for non-production.

```
# Premium — returns per-unit hourly rate; multiply by unit count × 730
sku: "Premium", region: "uksouth"

# Consumption — returns per-million-calls rate; multiply by monthly call volume
filter: "skuName eq 'Consumption' and serviceName eq 'API Management' and armRegionName eq 'uksouth'"
```

---

### Service Bus

SKU names match the portal (`Basic`, `Standard`, `Premium`). However:

- **Basic and Standard** are priced per operation/message — not a flat hourly rate. Ask about expected message volume before estimating.
- **Premium** is priced per messaging unit per hour (always-on). Multiply `retailPrice` × 730 per messaging unit.

---

### Azure Container Apps

The `service` parameter returns a 400 error for Container Apps — there is no single ARM SKU. **Always use the OData `filter` parameter:**

```
filter: "serviceName eq 'Azure Container Apps' and armRegionName eq 'uksouth'"
```

See [COST-FORMULAS.md](COST-FORMULAS.md) for the three-component pricing formula (vCPU-seconds + GiB-seconds + requests).

---

### Storage Accounts

The API drops the `Standard_` / `Premium_` prefix from storage SKUs:

| ARM / Bicep SKU | API `skuName` |
|-----------------|---------------|
| `Standard_LRS` | `LRS` |
| `Standard_GRS` | `GRS` |
| `Standard_RAGRS` | `RA-GRS` |
| `Standard_ZRS` | `ZRS` |
| `Premium_LRS` | `Premium LRS` |
| `Premium_ZRS` | `Premium ZRS` |

Use `filter` rather than `sku` for storage to avoid empty results:
```
filter: "skuName eq 'LRS' and serviceName eq 'Storage' and armRegionName eq 'uksouth'"
```

---

## General Fallback Strategy

When a `sku` query returns empty results:

1. Drop the `sku` parameter entirely and query by `service` + `region` only.
2. Scan the returned `skuName` values to find the correct format for your target SKU.
3. Re-query using `filter: "skuName eq '<exact-value>'"` combined with `serviceName` and `armRegionName`.
4. If `serviceName` is unknown, use `service-family` (e.g. `Databases`, `Compute`) to discover valid service names first.
