# Wyze-Android-Bypass MVP Roadmap 🗺️

## 1. Project Overview
This MVP transforms a fork of `go2rtc` into a native Android application designed for local doorbell ownership via automated pipelines on APatch-rooted ARM64 devices. It bridges Wyze's proprietary P2P protocol locally using root-native extraction to avoid redundant cloud authentications.

## Phase 1: Environment & Project Scaffolding
- **Objective**: Establish the foundational Android project tailored for root access and binary execution.
- **Tasks**:
  1. Initialize an Android project using Kotlin and Jetpack Compose, targeting SDK 34+. Configure Gradle for NDK support (required for potential native libraries) and declare the required architecture targets (`arm64-v8a`).
  2. Implement a Root Shell utility wrapper (e.g., integrating `libsu` by topjohnwu) to ensure reliable `APatch`/`Magisk` su command execution.
  3. Setup a CI/CD pipeline (GitHub Actions) to cross-compile the `go2rtc` Go binary for `android/arm64` and package it as a raw asset in the APK. Ensure the binary includes the `wyze` module (`pkg/wyze`, `pkg/tutk`).
  4. Declare required manifest permissions: `INTERNET`, `ACCESS_NETWORK_STATE`, `WAKE_LOCK`, and `FOREGROUND_SERVICE` (for persistent stream maintenance).
  5. Gain read access to `/data/data/` for cross-application intelligence gathering.

## Phase 2: Intelligence & Extraction Pipeline (The Scraping Engine)
- **Objective**: Dynamically locate and extract required Wyze credentials (`uid`, `enr`, `mac`, `ip`) without manual user entry.
- **Tasks**:
  1. **Root Access Validation**: Implement a root check using APatch/libsu. If root is unavailable, gracefully degrade to the Manual Configuration fallback.
  2. **Target Package Discovery**: Attempt to locate the official Wyze application package (e.g., `com.hualai` or similar 2026 variants) before proceeding with extraction.
  3. **Data Extraction & Heuristic Parsing**: Execute root shell commands to copy `/data/data/com.hualai/databases/` and `/data/data/com.hualai/shared_prefs/` to the MVP's local cache. Implement heuristic scanners to identify:
     - `mac`: Device MAC address (Regex `^([0-9A-Fa-f]{2}[:-]){5}([0-9A-Fa-f]{2})$`)
     - `uid`: 20-character alphanumeric P2P identifier.
     - `enr`: DTLS Encryption Key (look for standard lengths or specific neighboring keys in JSON/DB rows).
     - `model`: Camera Model
  4. **Local Encryption Blockage Mitigation**: *Risk:* Wyze may use Android Keystore-backed `EncryptedSharedPreferences` or SQLCipher. *Fallback:* If the heuristic scan yields encrypted blobs or empty results, abort extraction and trigger the Manual Configuration UI.
  5. *Fallback Mechanism*: If the official app is not installed, or data is encrypted/inaccessible due to newer security measures, present a Jetpack Compose form for manual Wyze API credential entry (API ID, API Key, Email, Password), and the app uses the `go2rtc` cloud auth endpoint to fetch the `enr` keys once, caching them locally. Log all fetched values to Logcat and to file for debugging. Implement robust Try/Catch blocks during the SQLite reading phase. Do not crash on inaccessible databases.

## Phase 3: go2rtc Bridge Automation
- **Objective**: Manage the lifecycle of the `go2rtc` backend.
- **Tasks**:
  1. Unpack the `go2rtc-arm64` binary from APK assets to the app's internal executable directory (`getFilesDir()`).
  2. Grant execution permissions (`chmod +x`).
  3. Generate a dynamic `go2rtc.yaml` configuration file based on the extracted Wyze parameters in the app's isolated storage directory (`/data/data/com.wyzebypass.mvp/files/`). Format the target string: `wyze://[IP]?uid=[UID]&enr=[ENR]&mac=[MAC]&model=[MODEL]&dtls=true`. Ensure the configuration binds specifically to the target doorbell's parameters to prevent LAN interference.
  4. **IP Resolution:** Resolve the camera's local IP address using ARP table scanning based on the extracted `mac` address.
  5. Launch the binary as a child process using `java.lang.ProcessBuilder`, capturing `stdout`/`stderr` to Android Logcat for debugging.
  6. Implement process death monitoring to restart the binary if it crashes. Capture the PID to enable explicit termination (SIGKILL) when the service stops.
  7. Configure `go2rtc` to expose the stream locally (e.g., via WebRTC or RTSP on `localhost:8554`) and integrate an Android media player (e.g., ExoPlayer) to consume the local stream.

## Phase 4: Persistent Lifecycle Management
- **Objective**: Ensure the local bridge stays alive reliably for the doorbell stream to be accessible 24/7.
- **Tasks**:
  1. Implement an Android Foreground Service with a persistent notification to host the `go2rtc` process. This ensures the bridge survives Activity destruction.
  2. Utilize `WakeLocks` (`PARTIAL_WAKE_LOCK` and `WifiManager.WifiLock`) to prevent the CPU from sleeping only while an active stream session is running. Define a concrete release strategy: release both immediately when streaming stops, the bridge crashes/exits, or connectivity is lost to avoid unnecessary battery/thermal drain.
  3. Implement a `BroadcastReceiver` to handle network changes (e.g., switching from Wi-Fi to Cellular) to pause/resume the bridge, release/reacquire the locks as needed on network loss/recovery, update the targeted local IP of the camera via mDNS/ARP scans if the IP changes, and verify the behavior under Doze/app standby so the service does not keep the device awake unnecessarily.

## Phase 5: MVP Delivery & Testing
- **Objective**: Finalize a debug APK capable of maintaining a stream to a single doorbell.
- **Tasks**:
  1. Create a minimal Jetpack Compose UI displaying the bridge status (Running/Stopped) and extracted camera info.
  2. Bind an Android video player (e.g., ExoPlayer for RTSP or WebRTC WebView) to the localhost stream directly in the app to prove it works end-to-end.
  3. **Verbose Logging**: Capture standard output and standard error from the `go2rtc` process. Log all heuristic parsing results, API responses, and lifecycle events.
  4. **Crash Reporting**: Implement an uncaught exception handler. On crash, write the full logcat buffer, `go2rtc` logs, and device state to `/sdcard/Download/WyzeBypassLogs/` for easy retrieval by developers (rooted device permits direct external-storage writes).

## Potential Vectors of Failure & Armor

1. **Database Encryption:**
   - *Risk:* In 2026, Wyze may have moved to SQLCipher or Keystore-backed encryption for their `/data/data/` storage. Local token scraping is **IMPOSSIBLE**.
   - *Mitigation:* The application must seamlessly transition to a Jetpack Compose form requesting the user to provide the API Key, API ID, Email, and Password (to fetch details via the Wyze Cloud API) OR the raw `UID`/`ENR`/`MAC`.
2. **JNI/NDK Boundary Crashes:**
   - *Risk:* Running a Go binary (`go2rtc`) inside an Android context via JNI can lead to signal faults (SIGSEGV) if memory boundaries are crossed or if Go tries to use unavailable Android syscalls.
   - *Mitigation:* The MVP is a debug build. Configure the Go runtime and Android NDK to dump verbose tombstone crash logs to the app's file directory. Use `panic` recovery in the Go wrapper. For the MVP, stick to executing the standalone binary via `ProcessBuilder`. It provides process isolation and avoids JNI complexity.
3. **DTLS Certificate Pinning / Protocol Changes:**
   - *Risk:* The TUTK/Wyze P2P protocol is reverse-engineered. Firmware updates could alter the handshake or DTLS requirements. The current `TUTK`/`DTLS` challenge-response (K10001/K10003) might change.
   - *Mitigation:* Ensure the `go2rtc` core can be updated independently of the Android UI. Log the exact byte sequences of failed handshakes to assist in rapid patching.
4. **Network State Instability:**
   - *Risk:* Android aggressively kills background network connections.
   - *Mitigation:* The MVP MUST use a Foreground Service.
