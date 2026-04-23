# Wyze-Android-Bypass MVP Roadmap 🗺️

## Project Overview
This MVP transforms a fork of `go2rtc` into a native Android application designed for local doorbell ownership via automated pipelines on APatch-rooted ARM64 devices. It bridges Wyze's proprietary P2P protocol locally using root-native extraction to avoid redundant cloud authentications.

## Phase 1: Environment & Project Scaffolding
- **Objective**: Establish the foundational Android project tailored for root access and binary execution.
- **Tasks**:
  1. Initialize an Android project using Kotlin and Jetpack Compose, targeting SDK 34+. Configure Gradle for NDK support (required for potential native libraries) and declare the required architecture targets (`arm64-v8a`).
  2. Implement a Root Shell utility wrapper (e.g., integrating `libsu` by topjohnwu) to ensure reliable `APatch`/`Magisk` su command execution.
  3. Setup a CI/CD pipeline (GitHub Actions) to cross-compile the `go2rtc` Go binary for `android/arm64` and package it as a raw asset in the APK. Ensure the binary includes the `wyze` module (`pkg/wyze`, `pkg/tutk`).
  4. Declare required manifest permissions: `INTERNET`, `ACCESS_NETWORK_STATE`, `WAKE_LOCK`, and `FOREGROUND_SERVICE` (for persistent stream maintenance).

## Phase 2: Intelligence & Extraction Pipeline
- **Objective**: Dynamically locate and extract required Wyze credentials (`uid`, `enr`, `mac`, `ip`) without manual user entry.
- **Tasks**:
  1. **Root Access Validation**: Implement a root check using APatch/libsu. If root is unavailable, gracefully degrade to the Manual Configuration fallback.
  2. **Target Package Discovery**: Verify the installation of `com.hualai.WyzeCam` (or the 2026 equivalent package, e.g., `com.wyze.smarthome`) before proceeding with extraction.
  3. **Data Extraction & Heuristic Parsing**: Execute root shell commands to copy `/data/data/com.hualai.WyzeCam/databases/` and `/data/data/com.hualai.WyzeCam/shared_prefs/` to the MVP's local cache. Implement heuristic scanners to identify:
     - `MAC Address`: Regex `^([0-9A-Fa-f]{2}[:-]){5}([0-9A-Fa-f]{2})$`
     - `UID`: 20-character alphanumeric P2P identifier.
     - `ENR`: Encryption key (look for standard lengths or specific neighboring keys in JSON/DB rows).
     - **Required Extraction Targets for go2rtc:** `mac`, `uid`, `enr`, and `model`.
  4. **Local Encryption Blockage Mitigation**: *Risk:* Wyze may use Android Keystore-backed `EncryptedSharedPreferences` or SQLCipher. *Fallback:* If the heuristic scan yields encrypted blobs or empty results, abort extraction and trigger the Manual Configuration UI.
  5. *Fallback*: If `enr` keys are Keystore-encrypted, implement a secondary fallback where the user enters the Wyze API Key/ID, and the app uses the `go2rtc` cloud auth endpoint to fetch the `enr` keys once, caching them locally. Log all fetched values (API responses, extracted keys, intermediate states) to Logcat and to file for debugging — security hardening will be addressed in the production release.

## Phase 3: go2rtc Bridge Automation
- **Objective**: Manage the lifecycle of the `go2rtc` backend.
- **Tasks**:
  1. Unpack the `go2rtc-arm64` binary from APK assets to the app's internal executable directory (`getFilesDir()`).
  2. Grant execution permissions (`chmod +x`).
  3. Generate a dynamic `go2rtc.yaml` configuration file based on the extracted Wyze parameters. Format: `wyze://[IP]?uid=[UID]&enr=[ENR]&mac=[MAC]&model=[MODEL]&dtls=true`. Ensure the configuration binds specifically to the target doorbell's parameters to prevent LAN interference. Resolve the camera's local IP via ARP table scanning based on the extracted `mac` address.
  4. Launch the binary as a child process using `java.lang.ProcessBuilder`, capturing `stdout`/`stderr` to Android Logcat for debugging.
  5. Implement process death monitoring to restart the binary if it crashes. Capture the PID to enable explicit termination (SIGKILL) when the service stops.
  6. Configure `go2rtc` to expose the stream locally (e.g., via WebRTC or RTSP on `localhost:8554`) and integrate an Android media player (e.g., ExoPlayer) to consume the local stream.

## Phase 4: Persistent Lifecycle Management
- **Objective**: Ensure the local bridge stays alive reliably for the doorbell stream to be accessible 24/7.
- **Tasks**:
  1. Implement an Android Foreground Service with a persistent notification to host the `go2rtc` process. This ensures the bridge survives Activity destruction.
  2. Acquire `PARTIAL_WAKE_LOCK` and `WifiManager.WifiLock` only while an active stream session is running. Define a concrete release strategy: release both immediately when streaming stops, the bridge crashes/exits, or connectivity is lost to avoid unnecessary battery/thermal drain.
  3. Handle network changes (e.g., switching from Wi-Fi to Cellular) to pause/resume the bridge, release/reacquire the locks as needed on network loss/recovery, update the targeted local IP of the camera via mDNS/ARP scans if the IP changes, and verify the behavior under Doze/app standby so the service does not keep the device awake unnecessarily.

## Phase 5: MVP Delivery & Testing
- **Objective**: Finalize a debug APK capable of maintaining a stream to a single doorbell.
- **Tasks**:
  1. Create a minimal Jetpack Compose UI displaying the bridge status (Running/Stopped) and extracted camera info.
  2. Add a `WebView` or ExoPlayer surface to display the local `go2rtc` WebRTC/RTSP stream directly in the app to prove it works end-to-end.
  3. **Verbose Logging**: Capture standard output and standard error from the `go2rtc` process. Log all heuristic parsing results, API responses, and lifecycle events.
  4. **Crash Reporting**: Implement an uncaught exception handler. On crash, write the full logcat buffer, `go2rtc` logs, and device state to `/sdcard/Download/WyzeBypassLogs/` for easy retrieval by developers (rooted device permits direct external-storage writes).

## Potential Vectors of Failure & Armor

1. **Encrypted Local Data:** If `/data/data/com.hualai.WyzeCam/` is encrypted via hardware keystores, local token scraping is **IMPOSSIBLE**.
   - *Armor:* The application must seamlessly transition to a Jetpack Compose form requesting the user to provide the API Key, API ID, Email, and Password (to fetch details via the Wyze Cloud API) OR the raw `UID`/`ENR`/`MAC`.
2. **Protocol Changes:** Wyze frequently updates firmwares. The current `TUTK`/`DTLS` challenge-response (K10001/K10003) might change.
   - *Armor:* Ensure the `go2rtc` version is easily updatable independently of the Android APK. Log the raw challenge bytes if authentication fails.
3. **JNI/NDK Boundary Issues:** If `go2rtc` is eventually converted to a shared library (`.so`) accessed via JNI instead of a standalone executable, memory management and threading between Go and Kotlin will be complex.
   - *Armor:* For the MVP, stick to executing the standalone binary via `ProcessBuilder`. It provides process isolation and avoids JNI complexity. Use `panic` recovery in the Go wrapper and configure the Android NDK to dump verbose tombstone crash logs to the app's file directory.
4. **Network State Instability:**
   - *Risk:* Android aggressively kills background network connections.
   - *Armor:* The MVP MUST use a Foreground Service with WakeLock and WifiLock held for the duration of the active stream.
