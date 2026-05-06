# Validating macOS Scheduled Scans in Microsoft Defender

Verifying that scheduled antivirus scans are running on a macOS fleet, using Microsoft Defender XDR Advanced Hunting.

## Goal

Confirm that **Quick Scans** are completing on macOS devices on a regular cadence over the last 30 days, and visualise the trend to spot gaps in coverage (rollout days, configuration drift, devices that stopped reporting).

## Query

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
// 3. Join with our Mac list to filter out other OS platforms
| join kind=inner MacDevices on DeviceId
// 4. Group by day to see the trend
| summarize DailyScanCount = count() by bin(Timestamp, 1d)
| render columnchart with (title="MacOS Quick Scan Historical Trend", xtitle="Date", ytitle="Total Scans")
```

## How it works

1. **Build a macOS device list.** `DeviceInfo` is filtered to `OSPlatform == "macOS"` and reduced to a unique set of `DeviceId` / `DeviceName` pairs. This is used later as a filter, so non-Mac scan events are excluded from the result.
2. **Pull scan completion events.** `DeviceEvents` is filtered to `ActionType == "AntivirusScanCompleted"` for the last 30 days. The scan metadata lives inside `AdditionalFields` as a JSON blob, so it has to be parsed.
3. **Filter to Quick Scans.** `Fields.ScanTypeIndex` is extracted and cast to a string. Defender reports values like `"Quick"` and `"Full"` here — `"Quick"` is the scheduled, lightweight scan that should run frequently.
4. **Inner join to the macOS list.** This drops any scan event that did not originate from a known macOS device.
5. **Bin per day and chart.** `bin(Timestamp, 1d)` groups completions by day; `render columnchart` produces the visual.

## Result

<img width="1984" height="1186" alt="image" src="https://github.com/user-attachments/assets/2cf320cc-a0ba-4a1d-80c4-89711674abfa" />

