# AADNonInteractiveUserSignInLogs — Conditional Access Ingestion Analysis

A KQL query for Azure Log Analytics / Microsoft Sentinel that correlates daily ingestion volume of `AADNonInteractiveUserSignInLogs` with the number and state of Conditional Access (CA) policies being evaluated per sign-in.

Useful for diagnosing unexpected ingestion spikes in non-interactive sign-in logs — particularly those caused by newly added or re-scoped Conditional Access policies (including report-only policies).

## What It Does

The query produces one row per day with:

| Column | Description |
|---|---|
| `Day` | Calendar day (UTC) |
| `IngestedGB` | Daily billable ingestion volume for the `AADNonInteractiveUserSignInLogs` table, sourced from the `Usage` table |
| `MaxTotalPolicies` | Highest number of CA policies attached to any single sign-in record that day (record-size ceiling) |
| `Enabled` | Distinct CA policies observed with result `success` or `failure` (enforced policies) |
| `ReportOnly` | Distinct CA policies observed in report-only mode (`reportOnlySuccess`, `reportOnlyFailure`, `reportOnlyInterrupted`, `reportOnlyNotApplied`) |
| `NotApplied` | Distinct CA policies that existed but didn't match sign-in conditions |
| `Disabled` | Distinct CA policies in disabled state (`notEnabled`) |
| `Other` | Any unrecognized `result` values (forward-compatibility bucket) |

## Why This Matters

Every Conditional Access policy evaluated during a sign-in — whether it's enforced, in report-only mode, or doesn't apply — adds an entry to the `ConditionalAccessPolicies` array in the sign-in record. Policies in **report-only mode** are the most common silent driver of ingestion cost increases because:

- They're evaluated against every sign-in that matches their base scope
- They emit `reportOnlyNotApplied` even when they don't match conditions
- They don't block or affect any sign-in, so administrators often don't associate them with ingestion growth
- A single new broadly-scoped report-only policy can add thousands of additional policy-evaluation entries per day

By aligning daily ingestion GB with policy state counts, this query surfaces the exact day a new policy appeared and whether it correlates with the cost increase.

## Performance

The query uses `partition by Day (take 500)` to cap the scan at ~500 records per day, regardless of total table size. This keeps execution fast even on tenants with millions of daily sign-ins. The `Usage` table is pre-aggregated and adds negligible cost.

Adjust the `take 500` value if you need higher confidence (e.g., `take 2000`) or faster execution (e.g., `take 200`).

## Prerequisites

- A Log Analytics workspace with diagnostic settings forwarding `AADNonInteractiveUserSignInLogs` from Microsoft Entra ID
- Read access to the `Usage` table (available in every workspace by default)
- Query permissions on the workspace

## The Query

```kusto
let Policies = 
    AADNonInteractiveUserSignInLogs
    | where TimeGenerated >= datetime(2026-04-01)
    | extend Day = bin(TimeGenerated, 1d)
    | partition by Day (take 500)
    | extend CAPolicies = todynamic(tostring(ConditionalAccessPolicies))
    | extend PolicyCount = array_length(CAPolicies)
    | summarize MaxTotalPolicies = max(PolicyCount) by Day;
let PolicyStates = 
    AADNonInteractiveUserSignInLogs
    | where TimeGenerated >= datetime(2026-04-01)
    | extend Day = bin(TimeGenerated, 1d)
    | partition by Day (take 500)
    | extend CAPolicies = todynamic(tostring(ConditionalAccessPolicies))
    | mv-expand CAPolicies
    | extend PolicyName = tostring(CAPolicies.displayName),
             PolicyResult = tostring(CAPolicies.result)
    | extend State = case(
        PolicyResult in ("success","failure"), "Enabled",
        PolicyResult in ("reportOnlySuccess","reportOnlyFailure","reportOnlyInterrupted","reportOnlyNotApplied"), "ReportOnly",
        PolicyResult == "notApplied", "NotApplied",
        PolicyResult == "notEnabled", "Disabled",
        "Other")
    | summarize DistinctPolicies = dcount(PolicyName) by Day, State
    | evaluate pivot(State, sum(DistinctPolicies));
let Volume = 
    Usage
    | where TimeGenerated >= datetime(2026-04-01)
    | where DataType == "AADNonInteractiveUserSignInLogs"
    | summarize IngestedGB = sum(Quantity) / 1000.0 by Day = bin(TimeGenerated, 1d);
Volume
| join kind=fullouter Policies on Day
| join kind=fullouter PolicyStates on Day
| project Day, 
          IngestedGB = round(IngestedGB, 2),
          MaxTotalPolicies,
          Enabled, ReportOnly, NotApplied, Disabled, Other
| order by Day asc
```
## Output
<img width="1527" height="729" alt="image" src="https://github.com/user-attachments/assets/30f260a3-172a-4588-ad15-483560718124" />


## Customization

### Change the time range

Replace `datetime(2026-04-01)` on all three branches with your desired start date, or use a relative range such as `ago(30d)`.

### Adjust the sample size

Modify `(take 500)` to increase or decrease per-day scan depth. Larger values give more accurate distinct-policy counts; smaller values run faster.

### Filter by policy state at source

To focus only on sign-ins where CA actually applied (smaller dataset, faster query), add before the `partition` line:

```kusto
| where ConditionalAccessStatus != "notApplied"
```

## Interpreting the Output

| Pattern | Likely Cause |
|---|---|
| `IngestedGB` grows but `MaxTotalPolicies` and state counts stay flat | More sign-in events (new app, service principal, retry loop) |
| `IngestedGB` grows together with `ReportOnly` count | New report-only CA policy added or existing one re-scoped broader |
| `IngestedGB` grows together with `Enabled` count | New enforced CA policy added |
| `Other` column has unexpectedly high values | Policy result schema has new values — run the diagnostic below |


### Follow-up — Identify New Report-Only Policies

Once a spike day is identified, this query finds which specific report-only policies first appeared around that time:

```kusto
AADNonInteractiveUserSignInLogs
| where TimeGenerated >= datetime(2026-04-01)
| extend Day = bin(TimeGenerated, 1d)
| partition by Day (take 500)
| extend CAPolicies = todynamic(tostring(ConditionalAccessPolicies))
| mv-expand CAPolicies
| extend PolicyName = tostring(CAPolicies.displayName),
         PolicyResult = tostring(CAPolicies.result)
| where PolicyResult startswith "reportOnly"
| summarize FirstSeen = min(Day), Evaluations = count() by PolicyName
| order by FirstSeen desc, Evaluations desc
```

## Notes

- `Usage.Quantity` is reported in megabytes; the query divides by 1000 to produce gigabytes.
- `MaxTotalPolicies` uses `max()` rather than a percentile to reflect the true ceiling of policies attached to a single record. If a single outlier record skews the value, consider `percentile(PolicyCount, 95)` instead.
- Distinct-policy counts per state (`Enabled`, `ReportOnly`, etc.) depend on the sample — a policy that exists but never appears in the sampled records will be missing. Increasing `take` improves completeness.
- The query is intended for diagnostic and cost-analysis use; it is not a security detection on its own.

