# Wyze-Android-Bypass MVP Roadmap 🗺️

## Phase 1: Environment & Project Scaffolding
- **Objective**: Establish the foundational Android project tailored for root access and binary execution.
- **Tasks**:
  1. Initialize an Android project targeting SDK 34+.
  2. Implement a Root Shell utility wrapper (e.g., integrating `libsu` by topjohnwu) to ensure reliable `APatch`/`Magisk` su command execution.
  3. Setup a CI/CD pipeline (GitHub Actions) to cross-compile the `go2rtc` Go binary for `android/arm64` and package it as a raw asset in the APK.

## Phase 2: Intelligence & Extraction Pipeline
- **Objective**: Dynamically locate and extract required Wyze credentials (`uid`, `enr`, `mac`, `ip`) without manual user entry.
- **Tasks**:
  1. Write a shell script/function to probe `/data/data/com.hualai.WyzeCam/` for device databases and shared preferences.
  2. Implement SQLite parsing logic via root shell (e.g., `su -c 'sqlite3 ... "SELECT ..."'`) to extract cached device info.
  3. *Fallback*: If `enr` keys are Keystore-encrypted, implement a secondary fallback where the user enters the Wyze API Key/ID, and the app uses the `go2rtc` cloud auth endpoint to fetch the `enr` keys once, caching them locally.

## Phase 3: go2rtc Bridge Automation
- **Objective**: Manage the lifecycle of the `go2rtc` backend.
- **Tasks**:
  1. Unpack the `go2rtc-arm64` binary from APK assets to the app's internal executable directory (`getFilesDir()`).
  2. Grant execution permissions (`chmod +x`).
  3. Generate a dynamic `go2rtc.yaml` configuration file based on the extracted Wyze parameters.
  4. Launch the binary as a child process, capturing `stdout`/`stderr` to Android Logcat for debugging.
  5. Implement process death monitoring to restart the binary if it crashes.

## Phase 4: Persistent Lifecycle Management
- **Objective**: Ensure the local bridge stays alive reliably for the doorbell stream to be accessible 24/7.
- **Tasks**:
  1. Implement an Android Foreground Service with a persistent notification.
  2. Acquire `PARTIAL_WAKE_LOCK` and `WifiManager.WifiLock` to prevent the OS from sleeping the network or CPU while the stream is active.
  3. Handle network changes (e.g., switching from Wi-Fi to Cellular) to pause/resume the bridge or update the targeted local IP of the camera via mDNS/ARP scans if the IP changes.

## Phase 5: MVP Delivery & Testing
- **Objective**: Finalize a debug APK capable of maintaining a stream to a single doorbell.
- **Tasks**:
  1. Create a minimal Jetpack Compose UI displaying the bridge status (Running/Stopped) and extracted camera info.
  2. Add a `WebView` or ExoPlayer surface to display the local `go2rtc` WebRTC/RTSP stream directly in the app to prove it works.
  3. Implement verbose crash logging (saving stack traces and `go2rtc` logs to `/sdcard/Download/WyzeBypassLogs/`).
