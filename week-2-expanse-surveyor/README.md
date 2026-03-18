<div align="center">

```
██╗    ██╗███████╗███████╗██╗  ██╗    ██████╗
██║    ██║██╔════╝██╔════╝██║ ██╔╝    ╚════██╗
██║ █╗ ██║█████╗  █████╗  █████╔╝      █████╔╝
██║███╗██║██╔══╝  ██╔══╝  ██╔═██╗     ██╔═══╝
╚███╔███╔╝███████╗███████╗██║  ██╗    ███████╗
 ╚══╝╚══╝ ╚══════╝╚══════╝╚═╝  ╚═╝    ╚══════╝
     E X P A N S E   S U R V E Y O R
```

<img src="https://img.shields.io/badge/Status-COMPLETED-brightgreen?style=for-the-badge"/>
<img src="https://img.shields.io/badge/Score-60%20%2F%2060-red?style=for-the-badge"/>
<img src="https://img.shields.io/badge/Questions-7%20%2F%207-darkred?style=for-the-badge"/>
<img src="https://img.shields.io/badge/Category-Android%20Malware%20%7C%20APK%20Forensics%20%7C%20PCAP%20Analysis-black?style=for-the-badge"/>
<img src="https://img.shields.io/badge/Difficulty-Intermediate-orange?style=for-the-badge"/>

</div>

---

## `> MISSION DOSSIER`

```
╔══════════════════════════════════════════════════════════════════════════╗
║  CHALLENGE     : Week 2 — Expanse Surveyor                              ║
║  EVENT         : OffSec Arctic Howl — Gauntlet Season 2                 ║
║  CATEGORY      : Android Malware · APK Forensics · Network Analysis     ║
║  DIFFICULTY    : Intermediate                                            ║
║  POINTS        : 60 / 60                                                 ║
║  QUESTIONS     : 7 / 7 Correct                                           ║
║  REPORT DATE   : March 18, 2026                                          ║
║  OPERATOR      : Tamil05t                                                ║
╚══════════════════════════════════════════════════════════════════════════╝
```

---

## `> EVENT NARRATIVE`

> *Returning from a foreign realm, an Expanse Surveyor crossed back into the Cascade Expanse carrying new field notes, environmental observations, and images of systems never before cataloged. Initial debriefs showed nothing out of the ordinary. The data appeared clean.*
>
> *At home, the Surveyor installed the Research Gallery application on his Android to organize and review the findings. The app was used to offload photos and annotations captured during the expedition — seemingly harmless fragments of discovery.*
>
> *Within 48 hours of reconnecting to their home network, anomalies surfaced. Outbound connections appeared where none should exist. Obfuscated traffic pulsed at irregular intervals, synchronized with no known service. The home network security system flagged the activity and escalated the alert. The device was immediately quarantined.*
>
> *A forensic sweep followed. Application artifacts and network logs were preserved before the signal went dark. Whatever crossed back from the foreign expanse had already begun to adapt — learning the environment it had entered.*

**Your task:** Analyze the recovered artifacts and determine what was introduced, how it activated, and what it attempted to communicate inside the Cascade Expanse.

---

## `> INCIDENT SCENARIO`

```
TARGET DEVICE   : Android (quarantined)
APPLICATION     : Research Gallery (.apk)
TRIGGER         : Anomalous outbound connections post-installation
ARTIFACTS       : .har (network traffic) + .apk (decompiled)
THREAT CLASS    : Android Spyware — Modular C2-Driven RAT
C2 MECHANISM    : GitHub Gist Dead-Drop → ngrok Relay
```

---

## `> CHALLENGE QUESTIONS`

| # | Question |
|---|----------|
| Q1 | How does the malware obtain the C2 address? What is the domain? What source file contains the malicious code? |
| Q2 | What are the steps the malware uses to decode the real C2 address? |
| Q3 | What reconnaissance did the malware perform? List at least 2 specific filenames discovered. |
| Q4 | How does the application know which endpoint to call at what moment? |
| Q5 | What are the contents of the large requests sent to the server? Extract and describe any files. |
| Q6 | The final payload is executed repeatedly. What data is it collecting and why is it so insistent? |
| Q7 | Why did the anomaly in Q6 occur? |

---

## `> QUICK FINDINGS SUMMARY`

| Question | Answer |
|----------|--------|
| **Q1 — C2 Discovery** | GitHub Gist dead-drop → C2: `446d9f29543f.ngrok-free.app` · File: `PeriodicTaskManager.java` |
| **Q2 — Decoding Steps** | Fetch Gist → Base64 ×15 → XOR with key `blastoise` → real C2 URL |
| **Q3 — Recon** | FileScanner → DCIM/Documents/Download · Files: `20251013_170000.JPG`, `20251012_214700.mp4`, `c8750f0d.0` |
| **Q4 — Endpoint Selection** | C2 server pushes DEX payload via `GET /cdn/assets`; app loads via `InMemoryDexClassLoader` |
| **Q5 — Large Requests** | `POST /api/backup/chunk` — JPG (623 KB, Las Vegas GPS) + MP4 (34 MB, Las Vegas GPS) |
| **Q6 — Repeated Payload** | LocationTracker × 15 — collecting GPS, cell tower, WiFi; insistent due to `no_last_known_location` |
| **Q7 — Anomaly Cause** | Lacks `ACCESS_BACKGROUND_LOCATION`; exploits PASSIVE_PROVIDER; GPS activated by YouTube |

---

## `> ATTACK CHAIN OVERVIEW`

```
┌──────────────────────────────────────────────────────────────────────┐
│                    INFECTION CHAIN — WEEK 2                          │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  [1] Victim installs trojanized "Research Gallery" APK               │
│             ↓                                                        │
│  [2] PeriodicTaskManager fetches GitHub Gist (dead-drop)             │
│             ↓                                                        │
│  [3] 3856-char string decoded: Base64 ×15 → XOR 'blastoise'         │
│             ↓                                                        │
│  [4] C2 resolved: https://446d9f29543f.ngrok-free.app/cdn/assets    │
│             ↓                                                        │
│  [5] GET /cdn/assets → protobuf-wrapped DEX returned                │
│             ↓                                                        │
│  [6] InMemoryDexClassLoader executes module:                         │
│      FileScanner    → POST /telemetry/inventory (file list)          │
│      MetaDataParser → POST /api/backup/chunk   (media + GPS)        │
│      LocationTracker→ POST /api/location/update (GPS coords) ×15    │
│             ↓                                                        │
│  [7] Device quarantined — signal goes dark                           │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## `> C2 INFRASTRUCTURE`

```
DEAD-DROP  : https://gist.githubusercontent.com/0wizlr/
             a2e4ba3849d1366678c2df925ee2cc4e/raw?file=gistfile1.txt
C2 DOMAIN  : 446d9f29543f.ngrok-free.app
RELAY      : ngrok (tunneled)
PROTOCOL   : HTTPS
```

| Endpoint | Purpose |
|----------|---------|
| `GET /cdn/assets` | Polling — returns protobuf DEX module |
| `POST /telemetry/inventory` | File system recon results |
| `POST /api/backup/chunk` | Media file exfiltration (chunked) |
| `POST /api/location/update` | GPS + cell tower data |

---

## `> TOOLS USED`

| Tool | Purpose |
|------|---------|
| `jadx` / `apktool` | APK decompilation — Java + smali analysis |
| `Wireshark` / HAR viewer | Network traffic analysis |
| `Python 3` | Base64 ×15 + XOR decoding script |
| `ExifTool` | Image/video metadata extraction |
| `protoc` | Protobuf payload parsing |
| `Kali Linux VM` | Isolated analysis sandbox |
| `grep / find` | Source code pattern matching |

---

## `> FILES IN THIS DIRECTORY`

| File | Description |
|------|-------------|
| [`README.md`](./README.md) | ← You are here — Event info & challenge overview |
| [`INVESTIGATION_REPORT.md`](./INVESTIGATION_REPORT.md) | Full forensic walkthrough, all 7 answers, IOCs, detection rules |

---

<div align="center">

```
[ WEEK 2 — COMPLETE ] [ 7/7 ] [ 60/60 pts ] [ TLP:WHITE ]
```

</div>
