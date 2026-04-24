# AAD Non-Interactive Failed Sign-In Queries

Two KQL queries for `AADNonInteractiveUserSignInLogs` to identify the top apps and users experiencing failed sign-ins.

## Top 10 Failed Apps

Shows which applications and client app types are generating failed non-interactive sign-ins, along with the number of distinct users affected.

```kusto
AADNonInteractiveUserSignInLogs
| extend ErrorCode = tostring(parse_json(Status).errorCode)
| summarize TotalLogins = count(),
            SuccessfulLogins = countif(ErrorCode == 0),
            FailedLogins = countif(ErrorCode != 0),
            Users = make_set(UserPrincipalName)
    by AppDisplayName, ClientAppUsed
| where FailedLogins > 0
| order by FailedLogins desc
| project UserCount = array_length(Users), AppDisplayName, ClientAppUsed
| take 10
```

## Top 10 Failed Users

Shows which user principals have the most failed non-interactive sign-ins, along with the set of apps and client app types involved.

```kusto
AADNonInteractiveUserSignInLogs
| extend ErrorCode = tostring(parse_json(Status).errorCode)
| summarize TotalLogins = count(),
            SuccessfulLogins = countif(ErrorCode == 0),
            FailedLogins = countif(ErrorCode != 0),
            Apps = make_set(AppDisplayName),
            ClientApps = make_set(ClientAppUsed)
    by UserPrincipalName
| where FailedLogins > 0
| order by FailedLogins desc
| project-away TotalLogins
| take 10
```

## Notes

- `ErrorCode == 0` indicates a successful sign-in; any non-zero value is a failure — see [Microsoft Entra sign-in error codes](https://learn.microsoft.com/entra/identity/monitoring-health/concept-sign-ins) for details.
