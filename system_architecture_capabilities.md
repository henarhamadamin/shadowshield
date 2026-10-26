# 🛡️ ShadowShield — System Mechanics, Architecture & Capabilities Whitepaper

> **A Deep Technical Overview of How ShadowShield Operates and What It Achieves in Active Cyber Defense**

---

## 📖 1. Executive Summary: The Paradigm Shift

Conventional perimeter defenses—such as Web Application Firewalls (WAFs) and Intrusion Prevention Systems (IPS)—operate on a **binary blocking paradigm**. When an attacker sends a malicious payload, a traditional firewall drops the request and returns a `403 Forbidden` or `406 Not Acceptable` status code. While this stops the immediate request, it provides the adversary with instant oracle feedback:
1. The attacker immediately knows a defense mechanism is active.
2. The attacker rotates their IP via VPN/Proxy, clears cookies, and crafts obfuscated bypasses until they find an unmonitored vector.
3. The defender gains zero insight into the attacker's true identity, tooling, skill level, or post-exploitation objectives.

**ShadowShield** replaces binary blocking with **Active Cyber Deception, Behavioral Biometrics, and Real-Time Command-and-Control (C2)**. Instead of alerting the attacker, ShadowShield silently shifts malicious sessions into a parallel **Shadow Environment**, feeding them believable synthetic data while extracting their hardware fingerprint, real network identity, mouse movements, and keystroke dynamics.

---

## ⚙️ 2. How ShadowShield Works (The 5-Stage Operational Lifecycle)

ShadowShield processes every incoming interaction through a five-stage pipeline that executes in milliseconds without degrading legitimate user performance.

### Stage 1: Deep Request Inspection & Threat Scoring
Before the host application processes a request, the **Interceptor Engine (`core/Interceptor.php`)** inspects all incoming vectors simultaneously:
* **Query Parameters (`$_GET`) & Form Bodies (`$_POST`)**
* **Cookies (`$_COOKIE`) & Hidden Honeytoken Flags** (e.g., tampering with `legacy_admin_debug=0` to `1`)
* **HTTP Headers** (`User-Agent`, `Authorization`, `X-Api-Key`, and proxy forwarding headers)

Each matched signature carries a specific **Threat Weight ($w_i$)**. The system computes the session's cumulative threat score ($S_{\text{threat}}$):

$$S_{\text{threat}} = \sum_{i=1}^{n} w_i$$

* **SQL Injection Probe:** $w = 35$
* **Cross-Site Scripting (XSS):** $w = 30$
* **Path Traversal / LFI:** $w = 40$
* **Cookie Honeytoken Tampering:** $w = 45$
* **Stolen Canary Token Usage:** $w = 100$
* **Offline Bait File Detonation:** $w = 150$

### Stage 2: The Dual-State Routing Switch
ShadowShield maintains two distinct operational states governed by a configurable threshold ($\theta = 20$ by default) inside **`core/Database.php`**:
* **State A — Production Mode ($S_{\text{threat}} < \theta$):** The visitor is treated as a legitimate user and interacts normally with the real application and production database.
* **State B — Shadow Honeypot Mode ($S_{\text{threat}} \ge \theta$):** The moment the threshold is crossed, the session is flagged as `trapped` within the active lab workspace. All subsequent database queries and file operations for that session are transparently rerouted to the **Shadow Deception Engine**—completely isolating the production database from harm.

### Stage 3: Synthetic Vulnerability Emulation
Once inside Shadow Mode, the **Deception Engine (`core/DeceptionEngine.php`)** emulates realistic vulnerabilities so the attacker believes their exploit succeeded:
* **SQLi Emulation:** If the attacker injects syntax errors (e.g., a stray `'` quote), the engine returns a realistic MySQL driver error (`SQLSTATE[42000]: Syntax error...`). If they inject a tautology (`' OR 1=1 --`) or `UNION SELECT`, the engine returns records from the synthetic `shadow_users` table containing fake password hashes and planted **Canary API Keys**. If they inject `SLEEP(n)`, the engine artificially delays the HTTP response to mimic a blind time-based SQL injection.
* **Virtual Filesystem (LFI) Emulation:** Requests for `../../../../etc/passwd`, `../.env`, or `C:\Windows\win.ini` return dynamically generated Linux system files or cloud configuration files populated with traceable AWS and HashiCorp Vault honeytokens.

### Stage 4: Stealth Behavioral Telemetry & Hardware Fingerprinting
In the background, ShadowShield injects a lightweight, zero-dependency JavaScript sensor (**`assets/js/telemetry.js`**) into the browser. Every $2.5\text{ seconds}$, this sensor transmits an asynchronous telemetry packet to **`api/beacon.php`** containing:
1. **Hardware Fingerprint:** A composite hash derived from `HTML5 Canvas 2D` pixel rendering geometry, `WebGL` GPU vendor/renderer strings, CPU core count, and screen color depth.
2. **WebRTC STUN Probe:** Local `LAN` IP addresses, `mDNS (.local)` hostnames, and public `WAN (SRFLX)` IP addresses leaked via the browser's `RTCPeerConnection` stack.
3. **Behavioral Biometrics:** Normalized $(x, y)$ mouse movement vectors, click coordinates, keystroke flight-time intervals (in milliseconds), `Backspace` corrections, clipboard `Paste` events, and live **Browser DevTools (`F12`)** open detection.

### Stage 5: Live SOC Visualization & Active Countermeasures
The **Security Operations Center (`dashboard.php`)** polls **`api/stream.php`** to render the entire battlefield in real time at $60\text{ FPS}$ on an interactive HTML5 Canvas force-directed graph, empowering the defender to monitor, profile, and counter-attack the intruder live.

---

## 🚀 3. What ShadowShield Can Do (Core Capabilities)

### 3.1. Unmasking Attackers Across Cookie Wipes & Incognito Mode (`Fingerprint Persistence`)
When skilled attackers suspect they are being monitored, they clear browser cookies (`PHPSESSID`), open a private **Incognito** window, or switch VPN exit nodes.
* **What ShadowShield Does:** As soon as the new session loads the page, `telemetry.js` calculates the device's hardware fingerprint and sends it to `api/beacon.php`.
* **Result:** ShadowShield recognizes that the GPU/Canvas hardware signature matches a previously trapped session, automatically re-engages **Shadow Mode** before the attacker even types a new payload, adds a $+30$ evasion penalty score, and draws a dashed orange **`🔗 FP MATCH`** correlation link between the old and new sessions on the SOC graph.

### 3.2. Piercing Through VPNs & Proxies (`WebRTC & Timezone Unmasking`)
Attackers frequently hide behind commercial VPNs or proxy chains.
* **What ShadowShield Does:** It combines server-side proxy header inspection (`X-Forwarded-For`, `Via`) with client-side WebRTC ICE candidate extraction and hardware `Intl.DateTimeFormat` timezone/locale analysis.
* **Result:** Even if an attacker connects from a European VPN IP, ShadowShield reveals their internal network IP (`192.168.x.x`), their true WAN IP (if WebRTC is unpatched), and their actual hardware timezone (e.g., `Asia/Baghdad`) directly in the **Threat Forensics** panel.

### 3.3. Tracking Stolen Keys & Detonating Offline Bait Files (`Canary Honeytokens`)
ShadowShield plants high-value bait credentials inside its synthetic SQLi dumps and virtual `.env` files, alongside downloadable backup archives (`CorpNexus_Master_Vault_2026.html`).
* **In-Band Canary Detection:** If the attacker copies a stolen key (e.g., `ak_live_998123_HONEYTOKEN_TRAP_01`) and attempts to authenticate with it via a form, URL parameter, or `Authorization` header (using `curl`, Postman, or Burp Suite), ShadowShield triggers a critical **`Canary_Trap`** alert ($+100\text{ score}$) and feeds them a fake Superadmin Cloud Vault response.
* **Offline File Detonation:** When the attacker downloads the bait HTML vault file to their computer and opens it locally from their hard drive, a silent beacon inside the file phones home to `api/canary_tripwire.php`. Because it runs under the `file:///` protocol, it captures and transmits the attacker's **local filesystem path** (e.g., `file:///C:/Users/AttackerName/Downloads/...`), exposing their **Operating System username** and **real home IP address** (if they disconnected their VPN after downloading).

### 3.4. Automated Attacker Skill Profiling (`5-Axis Pentagon Radar Chart`)
Not all threats are equal. ShadowShield analyzes behavioral telemetry and payload sophistication to classify the adversary into one of four personas:
* **`👶 SCRIPT KIDDIE`:** Relies heavily on clipboard copy-pasting (`Ctrl+V`) and basic payloads (`' OR 1=1`).
* **`🤖 AUTOMATED SCANNER / BOT`:** Exhibits headless browser signatures (`navigator.webdriver`), tool User-Agents (`sqlmap`, `nikto`, `curl`), or zero mouse movement with superhuman typing intervals ($< 8\text{ ms}$).
* **`🕵️ TACTICAL PENTESTER`:** Manually crafts payloads, uses `Backspace` edits, probes multiple attack vectors, and opens browser **DevTools (`F12`)**.
* **`💀 APT / ELITE OPERATOR`:** Combines advanced payloads (`UNION SELECT`, `SLEEP()`, LFI traversal, Canary token pivot) with evasion techniques (cookie resets, VPNs, and DevTools inspection).

This profile is rendered live on a **5-Axis Canvas Radar Chart** measuring:
1. **`CRAFT`** (Manual typing, DevTools usage, mouse interaction)
2. **`COMPLEXITY`** (Advanced SQLi/LFI syntax & Canary exploitation)
3. **`PERSISTENCE`** (Attack volume & multi-vector probing)
4. **`STEALTH`** (Session resets, Incognito usage, VPN indicators)
5. **`AUTO/SPEED`** (Clipboard paste ratio & automated scanner signatures)

### 3.5. Live Offensive Countermeasures (`Two-Way C2 Console`)
From the **Threat Forensics** panel on the dashboard, a SOC operator can dispatch four real-time countermeasures directly into the trapped attacker's browser:
1. **🧪 Poison Clipboard:** Hooks the browser's `copy` event. Whenever the attacker tries to copy leaked data from the page (`Ctrl+C`), ShadowShield silently replaces their clipboard contents with a tracked Honeytoken trap command and logs what they tried to copy.
2. **🎣 Phish Credentials:** Injects a realistic system re-authentication modal (`PRIVILEGED SESSION EXPIRED — Authenticate Node`) over the attacker's screen. Any username or password the attacker types into the modal is immediately harvested and displayed on the SOC timeline.
3. **🐌 Inject UI Tarpit:** Applies severe input lag ($3.5\text{ seconds}$ of UI freeze and screen blur on every click) and forces a `wait` cursor to frustrate manual pentesters and disrupt timing-sensitive scripts.
4. **💀 Lockout Screen:** Wipes the attacker's DOM and locks their browser into a full-screen red **Intrusion Containment / Kernel Panic** terminal displaying their captured hardware fingerprint and screen geometry.

### 3.6. Multi-Lab Workspaces & Court-Ready Digital Forensics
* **Multi-Lab Management (`New Lab` & `All Labs`):** Operators can launch fresh, isolated deception workspaces for new testing scenarios or incidents without deleting historical data. Previous labs are archived and can be inspected or reactivated (`Set Live`) with a single click.
* **Forensic Dossier Export (PDF & JSON):** Generates a printable **Forensic Incident Dossier** complete with attacker biometrics, reconstructed mouse trajectory canvas, full payload timeline, and a cryptographic **`SHA-256 Chain-of-Custody Integrity Hash`** to prove the evidence was not tampered with, alongside raw `.json` exports for SIEM integration.

---

## 🎭 4. Side-by-Side Scenario: Attacker View vs. ShadowShield SOC View

| Step | What the Attacker Sees & Does | What ShadowShield Does Behind the Scenes |
| :--- | :--- | :--- |
| **1. Initial Probe** | Types `' OR 1=1 --` into the website's search bar. | `Interceptor` scores $+35$, switches the session into **Shadow Mode**, and returns synthetic `shadow_users` records containing `HONEYTOKEN_TRAP_01`. |
| **2. Reconnaissance** | Presses `F12` (DevTools) to inspect cookies and copies the leaked API key (`Ctrl+C`). | `telemetry.js` detects `F12`, logs the mouse path, and updates the **Skill Radar** to `🕵️ TACTICAL PENTESTER [DEVTOOLS]`. |
| **3. Evasion Attempt** | Opens an **Incognito** window to wipe cookies and hides behind a VPN. | Hardware GPU/Canvas fingerprint & WebRTC STUN probe unmask the attacker, auto-trapping the Incognito session and drawing an **`🔗 FP MATCH`** link. |
| **4. Bait Detonation** | Downloads `CorpNexus_Master_Vault_2026.html` and opens it on their desktop. | The offline file phones home to `canary_tripwire.php`, exposing the attacker's local `file:///C:/Users/...` path, OS username, and real IP. |
| **5. C2 Counter-Strike** | Tries to continue testing, but sees an authentication prompt followed by a red containment lockout screen. | The SOC operator clicked **🎣 Phish Credentials** and **💀 Lockout Screen**, harvesting the attacker's input and terminating their browser session. |