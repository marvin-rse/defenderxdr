# Sentinel Microsoft Entra ID Data Connector — Sizing Estimate

Estimating the ingestion volume of the Microsoft Sentinel **Microsoft Entra ID** data connector *before* it is enabled, by using **Microsoft Defender XDR Advanced Hunting** as a proxy.

## Why this workaround?

If the Sentinel Entra ID data connector is not yet active, the corresponding Log Analytics tables (`SigninLogs`, `AADNonInteractiveUserSignInLogs`, `AuditLogs`, etc.) do not exist in the workspace, so volume cannot be measured directly.

Microsoft Defender XDR exposes equivalent sign-in data through Advanced Hunting tables that share the same source. Querying those gives a reasonable approximation of the volume that would land in Sentinel once the connector is turned on.

## Table mapping (XDR → Sentinel)

| XDR Advanced Hunting table | Roughly equivalent Sentinel / Log Analytics tables |
| --- | --- |
| `EntraSignInEventsBeta` | `SigninLogs` + `AADNonInteractiveUserSignInLogs` + `AADServicePrincipalSignInLogs` + `AADManagedIdentitySignInLogs` (newer, extended schema) |
| `AADSignInEventsBeta` | Same coverage as above, older schema |

> **Pick one, not both.** `EntraSignInEventsBeta` and `AADSignInEventsBeta` largely overlap. Summing them double-counts events. `EntraSignInEventsBeta` is the more current of the two.

## Queries

Run each query separately in the Microsoft Defender portal (`security.microsoft.com` → Advanced Hunting).

### Entra sign-in events (preferred)

```kusto
EntraSignInEventsBeta
| where Timestamp > ago(30d)
| extend _size = estimate_data_size(*)
| summarize
    AvgSizeKB    = round(avg(_size)/1024.0, 2),
    TotalEntries = count(),
    TotalSizeGB  = round(sum(_size)/1024.0/1024.0/1024.0, 3),
    DailyAvgGB   = round(sum(_size)/1024.0/1024.0/1024.0/30, 3)
```

### AAD sign-in events (older schema, for comparison)

```kusto
AADSignInEventsBeta
| where Timestamp > ago(30d)
| extend _size = estimate_data_size(*)
| summarize
    AvgSizeKB    = round(avg(_size)/1024.0, 2),
    TotalEntries = count(),
    TotalSizeGB  = round(sum(_size)/1024.0/1024.0/1024.0, 3),
    DailyAvgGB   = round(sum(_size)/1024.0/1024.0/1024.0/30, 3)
```

## Interpreting the result

The number returned by `estimate_data_size()` is **not** the number you will be billed for. Treat it as a lower-bound estimate and apply the following adjustments:

1. **Schema overhead (~5–15 %).** The Sentinel `SigninLogs` family includes nested fields that are not present in the leaner XDR schema (`ConditionalAccessPolicies`, full `AuthenticationDetails`, `DeviceDetail`, `NetworkLocationDetails`, etc.). Empirically, the XDR `EntraSignInEventsBeta` estimate runs about **5–15 % lower** than what is actually ingested into the equivalent Sentinel sign-in tables.
2. **Tables not covered by XDR (+10–30 %).** The Entra ID connector ingests several tables that the XDR sign-in tables do not include:
   - `AuditLogs` — directory and admin activity
   - `AADProvisioningLogs`
   - `MicrosoftGraphActivityLogs` (often very large)
   - `AADUserRiskEvents`, `AADRiskyUsers`, `AADServicePrincipalRiskEvents`, `AADRiskyServicePrincipals`
   - `ADFSSignInLogs` (if AD FS is in use)
   - `EnrichedOffice365AuditLogs`, `NetworkAccessTraffic` (optional)

   Their contribution depends heavily on tenant size, automation footprint, and whether Graph Activity Logs are enabled.

## Validation after the connector is enabled

Once data is flowing into Log Analytics, switch to the **billed** view, which is the actual basis for cost:

```kusto
Usage
| where TimeGenerated > ago(30d)
| where IsBillable == true
| where DataType in (
    "SigninLogs", "AADNonInteractiveUserSignInLogs", "AADServicePrincipalSignInLogs",
    "AADManagedIdentitySignInLogs", "AuditLogs", "AADProvisioningLogs", "ADFSSignInLogs",
    "AADUserRiskEvents", "AADRiskyUsers", "MicrosoftGraphActivityLogs"
  )
| summarize BilledGB = round(sum(Quantity)/1024, 3) by DataType
| order by BilledGB desc
```

And for a more granular per-table volume estimate from the workspace itself:

```kusto
union isfuzzy=true
    (SigninLogs                      | extend _Table = "SigninLogs"),
    (AADNonInteractiveUserSignInLogs | extend _Table = "AADNonInteractiveUserSignInLogs"),
    (AADServicePrincipalSignInLogs   | extend _Table = "AADServicePrincipalSignInLogs"),
    (AADManagedIdentitySignInLogs    | extend _Table = "AADManagedIdentitySignInLogs"),
    (AuditLogs                       | extend _Table = "AuditLogs"),
    (AADProvisioningLogs             | extend _Table = "AADProvisioningLogs"),
    (ADFSSignInLogs                  | extend _Table = "ADFSSignInLogs"),
    (AADUserRiskEvents               | extend _Table = "AADUserRiskEvents"),
    (AADRiskyUsers                   | extend _Table = "AADRiskyUsers"),
    (MicrosoftGraphActivityLogs      | extend _Table = "MicrosoftGraphActivityLogs")
| where TimeGenerated > ago(30d)
| extend _size = estimate_data_size(*)
| summarize
    Entries     = count(),
    AvgSizeKB   = round(avg(_size)/1024.0, 2),
    TotalSizeGB = round(sum(_size)/1024.0/1024.0/1024.0, 3)
    by _Table
| order by TotalSizeGB desc
```

`union isfuzzy=true` lets the query run even if some tables are not yet ingested in the workspace.

## Notes

- All size math uses binary units (`/1024`), not decimal (`/1000`).
- `estimate_data_size(*)` measures the row size at query time; billed ingestion can differ.
- `AADNonInteractiveUserSignInLogs` and `MicrosoftGraphActivityLogs` are typically the largest contributors and dominate cost.
- A Microsoft Entra ID P1/P2 license is required to ingest sign-in logs into Sentinel; Workload ID Premium is required for service-principal risk tables.

## References

- [Send data to Microsoft Sentinel using the Microsoft Entra ID data connector](https://learn.microsoft.com/azure/sentinel/connect-azure-active-directory)
- [Microsoft Sentinel data connectors reference](https://learn.microsoft.com/azure/sentinel/data-connectors-reference)
- [Microsoft Sentinel tables and associated connectors](https://learn.microsoft.com/azure/sentinel/sentinel-tables-connectors-reference)
