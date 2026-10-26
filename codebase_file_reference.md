# 📂 ShadowShield — Complete Codebase & File Reference Guide

> **A Granular File-by-File Technical Breakdown of Every Module, Endpoint, Script, and Asset in the ShadowShield Engine**

---

## 🗺️ 1. High-Level Directory Map

```text
shadow-shield/
├── core/                         # Backend OOP Core Engine (PHP 8+)
│   ├── Config.php                # System constants, DB credentials, thresholds & security keys
│   ├── Database.php              # Dual-State PDO router & Multi-Lab workspace manager
│   ├── DeceptionEngine.php       # Synthetic SQLi, Virtual LFI & Canary Vault responder
│   ├── Interceptor.php           # Deep HTTP request inspector & threat scoring engine
│   └── SocAuth.php               # SOC authentication gate & brute-force protection
├── api/                          # Asynchronous Telemetry, C2 & Forensics Endpoints
│   ├── beacon.php                # Two-way browser telemetry receiver & C2 dispatcher
│   ├── stream.php                # Real-time JSON data feed for the SOC Dashboard
│   ├── c2.php                    # Active Countermeasures (C2) command queue endpoint
│   ├── export.php                # Court-ready PDF/Print Dossier & JSON Threat Intel exporter
│   ├── canary_download.php       # Weaponized offline HTML bait file generator
│   ├── canary_tripwire.php       # Remote callback listener for offline file detonations
│   └── bridge.php                # Polyglot HTTP REST bridge for Node.js, Python, Go & .NET
├── assets/                       # Frontend Visualization, Styling & Stealth Sensors
│   ├── css/
│   │   └── dashboard.css         # Dark Cyberpunk SOC UI layout, animations & print styles
│   └── js/
│       ├── telemetry.js          # Client-side hardware fingerprinter, WebRTC probe & C2 agent
│       └── graph-engine.js       # 60 FPS Canvas Node Graph, 5-Axis Skill Radar & Audio synth
├── shadow-agent.php              # Universal 1-line drop-in connector for production PHP sites
├── dashboard.php                 # Security Operations Center (SOC) Command & Control UI
├── index.php                     # Built-in vulnerable sandbox portal for testing & demos
└── schema.sql                    # MySQL schema definitions & synthetic honeytoken seed data
```

---

## 🧠 2. Core Backend Engine (`core/`)

The `core/` directory contains the Object-Oriented PHP classes (`namespace ShadowShield\Core`) that power request inspection, database switching, and vulnerability emulation.

### 2.1. `core/Config.php`
* **Role:** Central Configuration & Security Constants Store.
* **Description:** Holds all environment-specific parameters in a single immutable `final class` so no credentials or magic numbers are hardcoded across the codebase.
* **Key Constants:**
  * `PROD_DB_HOST`, `PROD_DB_NAME`, `PROD_DB_USER`, `PROD_DB_PASS`: Connection parameters for the legitimate production database.
  * `SHADOW_DB_HOST`, `SHADOW_DB_NAME`, `SHADOW_DB_USER`, `SHADOW_DB_PASS`: Connection parameters for the isolated Shadow Honeypot & Telemetry database.
  * `THREAT_SCORE_THRESHOLD` ($\theta = 20$): The cumulative threat score at which a session is silently detached from production and locked into Shadow Mode.
  * `SECRET_SALT`: Cryptographic salt used for generating deterministic `SHA-256` session tokens and `HMAC` authentication signatures.
  * `SOC_MASTER_PASSKEY`: Master password required to unlock `dashboard.php`.
  * `HOST_ADMIN_SESSION_KEY`: Session key name (e.g., `is_super_admin`) used to automatically grant SOC access to logged-in administrators of the protected host website.
  * `PROTECTED_SITE_URL`: Target URL opened by the "Protected Site" button in the SOC header.

---

### 2.2. `core/Database.php`
* **Role:** Dual-State PDO Connection Router & Multi-Lab Workspace Manager.
* **Description:** Manages singleton PDO connections and dynamically routes database queries based on whether the current visitor is a normal user or a trapped attacker. It also manages the lifecycle of isolated testing workspaces (`Labs`).
* **Key Methods:**
  * `getShadowConnection(): PDO` — Returns the PDO instance connected to the Shadow Honeypot database and automatically verifies runtime schema migrations.
  * `getProdConnection(): PDO` — Returns the PDO instance connected to the real production database.
  * `getConnection(): PDO` — **The Dual-State Switch.** Checks `isShadowMode()`; if `true`, returns the Shadow DB connection; if `false`, returns the Production DB connection.
  * `ensureLabSchema(PDO $db): void` — Automatically provisions the `honeypot_labs` table and adds the `lab_id` foreign key column to `threat_sessions` without requiring manual SQL migrations.
  * `getActiveLabId(): int` — Retrieves the ID of the currently active Lab receiving live traffic.
  * `createNewLab(?string $customName): int` — Archives the current lab (`is_active = 0`) and initializes a fresh, clean Lab workspace (`is_active = 1`).
  * `getAllLabs(): array` — Returns all saved labs along with aggregated counts of trapped sessions and intercepted attack events.
  * `activateLab(int $labId): void` — Switches live traffic reception back to a previously archived lab.
  * `isShadowMode(): bool` — Checks if the visitor's session is flagged as trapped within the currently active lab (`$_SESSION['shadow_trapped_lab_' . $labId]`).
  * `engageShadowModeForCurrentLab(int $sessionId): void` — Locks the current PHP session into Shadow Mode for the active lab.
  * `getSessionTokenForCurrentLab(): string` — Generates a unique, lab-scoped `SHA-256` session token so starting a new lab cleanly resets visitor tracking states.

---

### 2.3. `core/Interceptor.php`
* **Role:** Deep Packet/Request Inspector & Threat Scoring Engine.
* **Description:** Runs at the very beginning of every HTTP request before application logic executes. It scans all user-supplied vectors against weighted regular expression signatures and records malicious activity into the database.
* **Key Methods & Logic:**
  * `inspect(): void` — Main entry point. Plants the hidden cookie trap (`legacy_admin_debug=0`), collects all request inputs, evaluates regex patterns, and triggers `handleThreatDetection()` when anomalies are found.
  * `collectRequestData(): array` — Flattens `$_GET`, `$_POST`, `$_COOKIE`, and critical HTTP headers (`User-Agent`, `Authorization`, `X-Api-Key`) into a unified inspection map.
  * `THREAT_PATTERNS` (Constant Map):
    * `Canary_Trap` ($w = 100$): Detects usage of stolen Honeytoken API keys or AWS/Vault secrets.
    * `SQLi` ($w = 35$): Detects `UNION SELECT`, boolean tautologies (`' OR 1=1`), SQL comments (`--`, `/*`), `INFORMATION_SCHEMA`, and time-based payloads (`SLEEP()`, `BENCHMARK()`).
    * `XSS` ($w = 30$): Detects `<script>`, `javascript:` URIs, `onerror=`, `onload=`, `<iframe>`, and `<svg>` injection vectors.
    * `Path_Traversal` ($w = 40$): Detects directory traversal sequences (`../`, `..\`), `/etc/passwd`, `.env`, `win.ini`, and `php://filter` wrappers.
    * `Recon` ($w = 25$): Detects automated security scanners (`sqlmap`, `nikto`, `nmap`, `gobuster`, `nuclei`, `burp`) via User-Agent signatures.
  * `handleThreatDetection(...)`: Upserts the attacker's record in `threat_sessions`, increments `threat_score`, logs the exact payload in `attack_events`, and engages Shadow Mode once $S_{\text{threat}} \ge \theta$.

---

### 2.4. `core/DeceptionEngine.php`
* **Role:** Synthetic Vulnerability Emulator & Bait Responder.
* **Description:** Generates realistic, interactive fake responses for trapped attackers so they remain engaged inside the honeypot believing their exploits are succeeding.
* **Key Methods:**
  * `emulateSqlQuery(string $rawInput): array` — Simulates a vulnerable MySQL query:
    * **Tarpit Delay:** If the attacker injects `SLEEP(n)`, pauses script execution for $\min(n, 5)\text{ seconds}$ to mimic blind time-based SQL injection.
    * **Syntax Error Simulation:** If the input has unbalanced quotes without a comment terminator, sets `$_SESSION['simulated_sql_error']` with an authentic MySQL driver error message (`SQLSTATE[42000]: Syntax error...`).
    * **Data Dump Simulation:** If the input contains `' OR 1=1` or `UNION SELECT`, queries and returns the entire `shadow_users` table containing fake password hashes and planted `HONEYTOKEN_TRAP` keys.
  * `emulateFileSystem(string $pathInput): ?string` — Simulates Local File Inclusion (LFI) and Path Traversal by returning synthetic contents for `/etc/passwd`, `.env` (containing fake AWS & Vault keys), and `C:\Windows\win.ini`.
  * `emulateVaultAccess(string $submittedKey): array` — Responds to stolen Canary API keys with fake Superadmin SSH bastion details and S3 backup bucket URIs to keep the attacker chasing phantom infrastructure.

---

### 2.5. `core/SocAuth.php`
* **Role:** SOC Console & Telemetry API Authentication Gate.
* **Description:** Prevents unauthorized users or attackers from accessing `dashboard.php` or reading live telemetry streams.
* **Key Methods:**
  * `isAuthenticated(): bool` — Grants access if either the host website's admin session (`Config::HOST_ADMIN_SESSION_KEY`) is active OR the operator has unlocked the console via `SOC_MASTER_PASSKEY` (verified using timing-safe `hash_equals` and `HMAC-SHA256`).
  * `requireApiAuth(): void` — Blocks unauthenticated requests to `api/stream.php`, `api/c2.php`, and `api/export.php` with a `403 Forbidden` JSON error.
  * `requireDashboardAuth(): void` — Renders the cyberpunk **SOC Gate Lock Screen** if the operator is not authenticated, enforcing a $60\text{-second}$ rate-limit lockout after $5$ failed passkey attempts.

---

## 📡 3. API & Telemetry Endpoints (`api/`)

The `api/` directory handles real-time communication between the trapped attacker's browser, the database, and the SOC operator's dashboard.

### 3.1. `api/beacon.php`
* **Role:** Two-Way Stealth Telemetry Receiver & C2 Command Dispatcher.
* **Called By:** `assets/js/telemetry.js` (every $2.5\text{ seconds}$ and on `paste` / `copy` / page load events).
* **Responsibilities:**
  1. **Fingerprint Persistence (Anti-Evasion):** Checks if a new session's hardware `device_fingerprint` matches any previously trapped session in the current lab. If matched, it automatically re-traps the attacker across cookie wipes or **Incognito** windows (`+30` score) and logs an `[EVASION]` event.
  2. **Bot vs. Human Classification:** Evaluates headless browser flags and mouse movement vectors to accurately classify the session as `👤 HUMAN` or `🤖 BOT`.
  3. **WebRTC & VPN Unmasking:** Compares the server connection IP with client-side WebRTC STUN leaked IPs (`LAN`, `Real-WAN`, `mDNS`), hardware `Timezone`, and HTTP proxy headers (`X-Forwarded-For`), logging a `[VPN-LEAK]` event node.
  4. **Biometric Recording:** Stores mouse movement coordinates (`mouse_vectors`), keystroke flight intervals, `Backspace` counts, clipboard `Paste` counts, and `DevTools (F12)` open flags in `behavior_telemetry`.
  5. **C2 Command Delivery:** Queries `c2_commands` for any `pending` countermeasures dispatched by the SOC operator, marks them as `delivered`, and returns them in the JSON response (`{"cmds": [...]}`) for immediate execution in the attacker's browser.

---

### 3.2. `api/stream.php`
* **Role:** Real-Time SOC Telemetry Broadcaster.
* **Called By:** `assets/js/graph-engine.js` (polled every $2.5\text{ seconds}$).
* **Responsibilities:**
  * Protected by `SocAuth::requireApiAuth()`.
  * Queries `threat_sessions`, `attack_events`, and `behavior_telemetry` filtered by the currently viewed `lab_id`.
  * Maps specialized deception tags (`[CANARY]`, `[OFFLINE-CANARY]`, `[EVASION]`, `[C2]`, `[VPN-LEAK]`) into distinct visual node types for the graph engine.
  * Aggregates cumulative `keystrokes`, `pastes`, `backspaces`, `devtools`, and `net_intel` metrics across all telemetry packets for each session so the **Attacker Skill Radar** has complete historical data.
  * Computes `linked_sessions` counts per hardware fingerprint so `graph-engine.js` can draw dashed `🔗 FP MATCH` correlation lines.

---

### 3.3. `api/c2.php`
* **Role:** Active Countermeasures Command Queue Endpoint.
* **Called By:** The 4 C2 action buttons inside the SOC Dashboard's **Threat Forensics** panel.
* **Responsibilities:**
  * Protected by `SocAuth::requireApiAuth()`.
  * Auto-creates the `c2_commands` table if it does not exist.
  * Validates and queues one of four active commands for a target `session_id`:
    * `poison_clipboard` (Clipboard Copy Trap)
    * `credential_trap` (Reverse Phishing Modal)
    * `ui_tarpit` (3.5s Interaction Freeze & Blur)
    * `lock_screen` (Full-Screen Kernel Panic Lockout)
  * Logs a `[C2]` event in `attack_events` so the counter-strike appears immediately as a node on the SOC topology graph.

---

### 3.4. `api/export.php`
* **Role:** Digital Forensics Dossier & Threat Intelligence Exporter.
* **Called By:** The **📄 Forensic Report (PDF)** and **📥 Export JSON** buttons in the SOC Dashboard.
* **Responsibilities:**
  * Protected by `SocAuth::requireApiAuth()`.
  * Compiles a complete incident package for a specific `session_id`, including subject identity, hardware biometrics, average keystroke speed, full mouse trajectory vectors, and chronological payload logs.
  * Calculates a cryptographic **`SHA-256 Chain-of-Custody Integrity Hash`** over the entire payload to verify evidence integrity.
  * **Mode 1 (`format=json`):** Triggers a direct `.json` file download formatted for SIEM/SOAR ingestion.
  * **Mode 2 (`format=dossier`):** Renders a formal, print-ready HTML/PDF Forensic Dossier (`TLP:RED`) featuring an embedded HTML5 Canvas that reconstructs the attacker's exact mouse path and click locations.

---

### 3.5. `api/canary_download.php`
* **Role:** Weaponized Offline Bait File Generator.
* **Called By:** Attackers clicking the exposed backup download link on the target portal.
* **Responsibilities:**
  * Immediately flags the downloading session as `trapped` ($+45\text{ score}$) and logs a `DOWNLOAD` event.
  * Dynamically builds the absolute callback URL to `api/canary_tripwire.php` matching the current host server.
  * Generates and forces the download of `CorpNexus_Master_Vault_2026.html`—an authentic-looking disaster recovery table containing fake root credentials and an embedded, self-executing JavaScript beacon that triggers when opened offline.

---

### 3.6. `api/canary_tripwire.php`
* **Role:** Remote Listener for Offline Bait File Detonations.
* **Called By:** The hidden JavaScript/Image beacon inside `CorpNexus_Master_Vault_2026.html` when the attacker opens the downloaded file on their own computer.
* **Responsibilities:**
  * Configured with permissive CORS headers (`Access-Control-Allow-Origin: *`) to accept beacons originating from local `file:///` origins.
  * Parses the `loc` parameter (`window.location.href`) using regex (`#/Users/([^/]+)/#i` and `#/home/([^/]+)/#i`) to extract the attacker's **Windows/macOS/Linux OS Username** and local directory structure.
  * Captures the attacker's **real IP address** (bypassing browser proxies/VPNs disabled offline), CPU core count, OS platform, and timezone.
  * Adds $+150$ to the session's `threat_score`, logs an `Offline_Detonation` node on the SOC graph, and returns a transparent $1\times 1\text{ GIF}$ pixel so the attacker sees no errors.

---

### 3.7. `api/bridge.php`
* **Role:** Polyglot HTTP REST Bridge for Non-PHP Backends.
* **Called By:** Middleware in **Node.js (Express/Next.js)**, **Python (FastAPI/Flask/Django)**, **Go**, or **.NET** applications.
* **Responsibilities:** Accepts a JSON payload containing the visitor's IP, User-Agent, endpoint, and request parameters; runs ShadowShield's threat inspection and `DeceptionEngine`; logs events to MySQL; and returns a JSON decision (`shadow_mode: true/false` along with synthetic `deception_data`).

---

## 🎨 4. Frontend Sensors, Visualization & Styling (`assets/`)

### 4.1. `assets/js/telemetry.js`
* **Role:** Stealth Client-Side Behavioral Biometrics Collector & C2 Execution Agent.
* **Injected Into:** Target website pages (automatically via `shadow-agent.php` or `<script>` tag).
* **Key Functions:**
  * `generateDeviceFingerprint()`: Renders hidden 2D Canvas geometry and queries the `WEBGL_debug_renderer_info` extension to extract the physical GPU model, hashing them with screen metrics and timezone into a persistent hardware ID.
  * `probeWebRTCLeak()`: Opens a silent `RTCPeerConnection` to Google STUN servers and parses ICE candidates to uncover local `LAN` IPs, `mDNS (.local)` hostnames, and public `SRFLX WAN` IPs behind VPNs.
  * `initMouseTracking()`: Samples normalized $(x, y)$ cursor coordinates every $60\text{ ms}$ (`type: 'm'`) and records exact click coordinates (`type: 'c'`).
  * `initKeystrokeTracking()`: Measures millisecond flight-time intervals between key presses, counts `Backspace` edits, captures clipboard `paste` events, and detects **DevTools (`F12` / `Ctrl+Shift+I` / window outer-inner dimension delta)**.
  * `evaluateBotProbability()`: Distinguishes real human operators from headless browsers (`navigator.webdriver`, `PhantomJS`, `HeadlessChrome`).
  * `flushTelemetry()`: Sends telemetry batches to `api/beacon.php` every $2.5\text{ seconds}$ and processes incoming C2 commands.
  * `executeCountermeasure(cmd)`: Executes live SOC commands inside the attacker's browser (`poison_clipboard`, `credential_trap`, `ui_tarpit`, and `lock_screen`).

---

### 4.2. `assets/js/graph-engine.js`
* **Role:** Real-Time Physics Node Graph, Skill Radar, Mouse Replay & SOC Audio Engine.
* **Loaded By:** `dashboard.php`.
* **Key Functions:**
  * `fetchTelemetryStream()`: Polls `api/stream.php?lab_id=...` every $2.5\text{ seconds}$ and synchronizes the UI state.
  * `syncGraphNodes(sessions)`: Builds the central `SHADOW CORE` node, orbits trapped attacker session nodes around it, spawns color-coded child nodes for each attack vector (`SQLi`, `XSS`, `Path_Traversal`, `Canary_Trap`, `Offline_Detonation`, `VPN_Leak`, `C2_Strike`), and links matching hardware fingerprints with dashed `🔗 FP MATCH` edges.
  * `updatePhysics()` & `renderGraph()`: Runs a $60\text{ FPS}$ `requestAnimationFrame` physics loop with node repulsion, spring attraction, drag-and-drop interactivity, and animated data-pulse particles traveling along active edges.
  * `computeSkillProfile(session)` & `renderSkillRadar(session)`: Evaluates the attacker across $5$ axes (`CRAFT`, `COMPLEXITY`, `PERSISTENCE`, `STEALTH`, `AUTO/SPEED`), assigns a threat persona badge (`👶 SCRIPT KIDDIE`, `🤖 AUTOMATED SCANNER`, `🕵️ TACTICAL PENTESTER`, or `💀 APT / ELITE OPERATOR`), and renders a 5-axis pentagon radar chart on `#skillRadarCanvas`.
  * `renderMouseTrajectory(telemetry)`: Reconstructs the attacker's screen movements (cyan path), clicks (red dots), and final cursor position (green dot) on `#mouseReplayCanvas`.
  * `playSocAlertTone(kind)`: Uses the native browser `AudioContext` oscillator API to synthesize sonar pings for new traps and dual-tone sawtooth alarms for critical Canary/Offline detonations.

---

### 4.3. `assets/css/dashboard.css`
* **Role:** SOC Interface Stylesheet.
* **Description:** Implements a responsive, zero-dependency dark cyberpunk interface (`#060911` background, `#00f2fe` cyan accents, `#ff2a5f` threat highlights), custom scrollbars, glowing modal dialogs for `New Lab` / `All Labs`, C2 control grids, and `@media print` rules.

---

## 🖥️ 5. Entry Points & Database Schema (Root Directory)

### 5.1. `shadow-agent.php`
* **Role:** Universal 1-Line Drop-In Connector for Production PHP Websites.
* **Description:** When included at the top of any real website (`require_once 'shadow-shield/shadow-agent.php';`), it automatically:
  1. Runs `Interceptor::inspect()` to catch attacks across all inputs.
  2. Checks `Database::isShadowMode()` and automatically injects synthetic SQLi table dumps or virtual LFI files into the page when a trapped attacker submits an exploit payload (such as `' OR 1=1 --` in a search bar).
  3. Uses PHP output buffering (`ob_start`) to dynamically inject `window.SHADOWSHIELD_BEACON_URL` and `<script src=".../telemetry.js" defer></script>` right before `</body>` without modifying the host website's source templates.

---

### 5.2. `dashboard.php`
* **Role:** Security Operations Center (SOC) Command & Control Interface.
* **Description:** Protected by `SocAuth::requireDashboardAuth()`. Provides the 3-column live monitoring console:
  * **Top Header:** Live counters (`Active Traps`, `Intercepted Payloads`, `Bot Signatures`, `Last Sync`), Audio Mute/Unmute toggle, Protected Site link, `🗂️ All Labs` archive modal, `➕ New Lab` launcher, and `🔒 Lock` button.
  * **Left Column:** Real-time list of trapped sessions with IP, hardware fingerprint preview, threat score, and Human/Bot badge.
  * **Center Column:** Interactive HTML5 Canvas Force-Directed Attack Topology Graph.
  * **Right Column (`Threat Forensics`):** Hardware identity table, WebRTC leaked IPs, Hardware Timezone, PDF/JSON Export buttons, Live C2 Countermeasure buttons, 5-Axis Attacker Skill Radar, Mouse Trajectory Replay canvas, and chronological Attack Timeline.

---

### 5.3. `index.php`
* **Role:** Built-in Vulnerable Sandbox Target Portal (`CorpNexus v2.4`).
* **Description:** A self-contained testing ground containing 5 interactive trap scenarios so operators can test every capability of ShadowShield locally:
  1. Employee Directory Lookup (**SQLi Trap**)
  2. Internal Audit Note (**XSS Sandbox Trap**)
  3. Server Config & Log Viewer (**Path Traversal / LFI Trap**)
  4. Cloud Vault API Key Gateway (**In-Band Canary Honeytoken Trap**)
  5. Restricted Disaster-Recovery Backup Download (**Weaponized Offline Canary File Trap**)

---

### 5.4. `schema.sql`
* **Role:** Database Initialization & Honeytoken Seed Script.
* **Description:** Creates the `shadow_honeypot` database (`utf8mb4`) and provisions the core tables:
  * `threat_sessions`: Stores attacker IP, User-Agent, hardware fingerprint, threat score, bot flag, lab ID, and containment status.
  * `attack_events`: Logs every intercepted payload, HTTP method, target endpoint, attack category, and deception response.
  * `behavior_telemetry`: Stores JSON arrays of mouse vectors, keystroke dynamics, DevTools flags, WebRTC network leaks, and screen resolution.
  * `shadow_users`: Pre-populated synthetic employee table containing fake bcrypt hashes and traceable `HONEYTOKEN_TRAP` API keys served to attackers during SQLi exploitation.