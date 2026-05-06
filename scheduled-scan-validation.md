# Validating Scheduled Scans on macOS and Linux in Microsoft Defender

Verifying that scheduled antivirus scans are running on macOS and Linux fleets, using Microsoft Defender XDR Advanced Hunting.

## Goal

Confirm that **Quick Scans** are completing on non-Windows endpoints on a regular cadence over the last 30 days, and visualise the trend to spot gaps in coverage (rollout days, configuration drift, devices that stopped reporting).

The same query pattern works for both platforms — only the `OSPlatform` filter changes.

## macOS query

```kusto
// 1. Identify your macOS fleet first
let MacDevices = DeviceInfo
| where OSPlatform == "macOS"
| summarize by DeviceId, DeviceName;
// 2. Pull ALL scan events for those devices from the last 30 days
DeviceEvents
| where Timestamp > ago(30d)
| where ActionType == "AntivirusScanCompleted"
| extend Fields = parse_json(AdditionalFields)
| extend ScanType = tostring(Fields.ScanTypeIndex)
| where ScanType == "Quick"
// 3. Join with the macOS list to filter out other OS platforms
| join kind=inner MacDevices on DeviceId
// 4. Group by day to see the trend
| summarize DailyScanCount = count() by bin(Timestamp, 1d)
| render columnchart with (title="MacOS Quick Scan Historical Trend", xtitle="Date", ytitle="Total Scans")
```

## Linux query

```kusto
// 1. Identify your Linux fleet first
let LinuxDevices = DeviceInfo
| where OSPlatform == "Linux"
| summarize by DeviceId, DeviceName;
// 2. Pull ALL scan events for those devices from the last 30 days
DeviceEvents
| where Timestamp > ago(30d)
| where ActionType == "AntivirusScanCompleted"
| extend Fields = parse_json(AdditionalFields)
| extend ScanType = tostring(Fields.ScanTypeIndex)
| where ScanType == "Quick"
// 3. Join with the Linux list to filter out other OS platforms
| join kind=inner LinuxDevices on DeviceId
// 4. Group by day to see the trend
| summarize DailyScanCount = count() by bin(Timestamp, 1d)
| render columnchart with (title="Linux Quick Scan Historical Trend", xtitle="Date", ytitle="Total Scans")
```

## Combined query (both platforms in one chart)

If a side-by-side view is more useful, group by `OSPlatform` as a series:

```kusto
let TargetDevices = DeviceInfo
| where OSPlatform in ("macOS", "Linux")
| summarize OSPlatform = any(OSPlatform) by DeviceId;
DeviceEvents
| where Timestamp > ago(30d)
| where ActionType == "AntivirusScanCompleted"
| extend ScanType = tostring(parse_json(AdditionalFields).ScanTypeIndex)
| where ScanType == "Quick"
| join kind=inner TargetDevices on DeviceId
| summarize DailyScanCount = count() by bin(Timestamp, 1d), OSPlatform
| render columnchart with (title="macOS vs Linux Quick Scan Historical Trend", xtitle="Date", ytitle="Total Scans", kind=unstacked)
```


## Result

<img width="1984" height="1186" alt="image" src="https://github.com/user-attachments/assets/2cf320cc-a0ba-4a1d-80c4-89711674abfa" />

