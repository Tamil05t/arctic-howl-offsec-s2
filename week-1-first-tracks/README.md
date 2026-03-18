<div align="center">

```
██╗    ██╗███████╗███████╗██╗  ██╗    ██╗
██║    ██║██╔════╝██╔════╝██║ ██╔╝   ███║
██║ █╗ ██║█████╗  █████╗  █████╔╝    ╚██║
██║███╗██║██╔══╝  ██╔══╝  ██╔═██╗     ██║
╚███╔███╔╝███████╗███████╗██║  ██╗    ██║
 ╚══╝╚══╝ ╚══════╝╚══════╝╚═╝  ╚═╝   ╚═╝
     F I R S T   T R A C K S
```

<img src="https://img.shields.io/badge/Status-COMPLETED-brightgreen?style=for-the-badge"/>
<img src="https://img.shields.io/badge/Score-40%20%2F%2040-red?style=for-the-badge"/>
<img src="https://img.shields.io/badge/Questions-6%20%2F%206-darkred?style=for-the-badge"/>
<img src="https://img.shields.io/badge/Category-Malware%20Analysis%20%7C%20PCAP%20Forensics-black?style=for-the-badge"/>
<img src="https://img.shields.io/badge/Difficulty-Easy-orange?style=for-the-badge"/>

</div>

---

## `> MISSION DOSSIER`

```
╔══════════════════════════════════════════════════════════════════════════╗
║  CHALLENGE     : Week 1 — First Tracks                                  ║
║  EVENT         : OffSec Arctic Howl — Gauntlet Season 2                 ║
║  CATEGORY      : Malware Analysis · PCAP Forensics · Incident Response  ║
║  DIFFICULTY    : Easy                                                    ║
║  POINTS        : 40 / 40                                                 ║
║  QUESTIONS     : 6 / 6 Correct                                           ║
║  REPORT DATE   : March 7, 2026                                           ║
║  OPERATOR      : Tamil05t                                                ║
╚══════════════════════════════════════════════════════════════════════════╝
```

---

## `> EVENT NARRATIVE`

> *The Cascade Expanse is no longer ruled by instinct alone. Ashka, an Arctic Wolf, was among the greatest cybersecurity hunters the Expanse had ever known — defending the Tundra Realm through instinct, reading subtle signals, sensing danger, and striking before threats could surface.*
>
> *When unusual activity rippled through the Tundra data center, Ashka moved to investigate — but the adversary was already there. Two steps ahead. From the shadows, Ashka was struck down and taken. When the alarms faded, she was gone.*
>
> *Her disappearance marked the beginning of a far greater threat.*

**Only those who adapt will survive. Only those who endure will uncover the truth. And only the strongest will reach the heart of the storm.**

---

## `> INCIDENT SCENARIO`

```
TARGET ORGANIZATION : Cascade Law Archive
PLATFORM            : macOS (Apple Silicon — M3 Pro Virtual)
TRIGGER             : Sudden cold spike in outbound network traffic
VECTOR              : New developer onboarding — cloned internal Xcode project
ARTIFACT PROVIDED   : capture.pcap
```

At the **Cascade Law Archive**, the IT department detected a sudden surge in outbound network traffic shortly after onboarding a new developer. While the firm primarily operates on Windows systems, the new hire had requested a Mac laptop and confirmed cloning a starter Xcode project from an internal Git repository during onboarding. No intentional software downloads were reported.

**Objective:** Analyze `capture.pcap` to understand the full Mac malware infection chain — from initial compromise through propagation.

---

## `> CHALLENGE QUESTIONS`

| # | Question |
|---|----------|
| Q1 | What URL did the malware download the first stage from? What user-agent sent the request? |
| Q2 | How does the C2 server obfuscate its payloads? |
| Q3 | Analyze the `looz` payload. What information does it extract from the victim machine? |
| Q4 | Analyze the `cozfi_xhh` payload. What information does it extract from the victim machine? |
| Q5 | How does the malware attempt to infect other devices? Which payload is responsible? |
| Q6 | What file contained the initial malware? How is the initial payload obfuscated? |

---

## `> QUICK FINDINGS SUMMARY`

| Question | Answer |
|----------|--------|
| **Q1 — Initial URL + UA** | `http://bu1knames.io/a` via `curl/8.7.1` |
| **Q2 — Obfuscation** | Base64 encoding within POST parameters to `/i` endpoint |
| **Q3 — looz** | System fingerprint stealer + modular dropper (OS, CPU, firewall, SIP, locale, browser) |
| **Q4 — cozfi_xhh** | Apple Notes + Reminders exfiltration → ZIP → `bu1knames.io/n` |
| **Q5 — Propagation** | `jez` — Git pre-commit hook injector |
| **Q6 — Initial File** | `xcassets.sh` — triple hexadecimal (xxd) encoding |

---

## `> ATTACK CHAIN OVERVIEW`

```
┌─────────────────────────────────────────────────────────────────────┐
│                    INFECTION CHAIN — WEEK 1                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  [1] Developer clones trojanized Xcode project (main.zip)           │
│             ↓                                                       │
│  [2] xcassets.sh executes (triple hex → curl bu1knames.io/a)        │
│             ↓                                                       │
│  [3] C2 delivers 7× Base64 AppleScript payloads                     │
│             ↓                                                       │
│  [4] looz    → System profiling → POST /i                           │
│      seizecj → Secondary profiling                                  │
│      cozfi_xhh → Apple Notes + Reminders theft → POST /n           │
│      txzx_vostfdi → Persistence                                     │
│      jez     → Git pre-commit hook injection                        │
│             ↓                                                       │
│  [5] Infection spreads to all local repos → other developers        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## `> C2 INFRASTRUCTURE`

```
DOMAIN   : bu1knames.io
PROTOCOL : HTTP (unencrypted)
PORT     : 80
```

| Endpoint | Purpose |
|----------|---------|
| `/a` | Initial beacon + first-stage payload download |
| `/l` | Environment / location fingerprinting |
| `/s/<name>` | Named payload distribution |
| `/i` | System info exfiltration (POST) |
| `/n` | Apple Notes + Reminders upload (`?s=<serial>`) |

---

## `> TOOLS USED`

| Tool | Purpose |
|------|---------|
| `Wireshark` | Visual PCAP analysis, HTTP object extraction |
| `tshark` | Command-line packet filtering & field extraction |
| `xxd` | Multi-layer hex encoding/decoding |
| `base64` | 7× nested Base64 decoding |
| `Python 3` | Automated decoding scripts |
| `Kali Linux VM` | Isolated analysis sandbox |
| `grep / sed / awk` | Log parsing & data extraction |
| `curl` | C2 communication reconstruction |
| `find` | File system forensics |

---

## `> FILES IN THIS DIRECTORY`

| File | Description |
|------|-------------|
| [`README.md`](./README.md) | ← You are here — Event info & challenge overview |
| [`INVESTIGATION_REPORT.md`](./INVESTIGATION_REPORT.md) | Full forensic walkthrough, all 6 answers, IOCs, detection rules |

---

<div align="center">

```
[ WEEK 1 — COMPLETE ] [ 6/6 ] [ 40/40 pts ] [ TLP:WHITE ]
```

</div>
