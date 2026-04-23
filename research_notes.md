# Wyze-Android-Bypass MVP Research Notes (Architect)

## Overview
This document compiles intelligence gathered on Wyze authentication, token handling, and local stream interception mechanisms as of late 2024 / 2026+. It focuses heavily on how `go2rtc` implements the `wyze` native protocol (via TUTK/DTLS) and the requirements for an APatch-rooted Android app to bypass standard Wyze cloud dependency for local streams.

## go2rtc Wyze Protocol Integration
`go2rtc` recently added native Wyze P2P stream support (`pkg/wyze`), bypassing the need for third-party bridges like `docker-wyze-bridge` or `wz_mini_hacks` in many cases.

### Core Mechanisms (TUTK / DTLS)
1. **Cloud Authentication**:
   - Endpoint: `https://auth-prod.api.wyze.com/api/user/login`
   - Uses email/password or MD5-hashed passwords to receive an `access_token`.
   - Endpoint: `https://api.wyzecam.com/app/v2/home_page/get_object_list` (and `/app/v2/device/get_iotc_info`)
   - Retrieves the list of cameras, extracting the critical `mac`, `p2p_id` (UID), `enr` (Encryption key for DTLS), and `ip`.
2. **Local P2P Handshake (TUTK / IOTC)**:
   - Connects locally to the camera's IP on port `0` (dynamic/probed) or discovered via UDP broadcast.
   - Requires the 20-character P2P UID (`uid`).
   - Uses DTLS (Datagram Transport Layer Security) for encryption (`dtls.DialDTLS(host, port, uid, authKey, enr)`).
3. **Session Authentication (K-Auth)**:
   - `doKAuth()` sequence in `pkg/wyze/client.go`:
     - **K10000**: Initiates auth sequence.
     - **K10001**: Receives challenge from the camera.
     - **K10002**: Sends challenge response encrypted with the `enr` key (`generateChallengeResponse`).
     - **K10003**: Receives auth success/failure and camera configuration (e.g., audio encoding support).
4. **AV Stream Handshake**:
   - Sends AV Login.
   - Probes for codecs (e.g., H264, AAC).
   - Starts reading AV frames via `tutk.Packet`.

### Required Data for Local Connection
To connect directly to a Wyze camera locally without hitting the cloud, the following data points are strictly necessary per device:
- **`ip`**: Local IPv4 address.
- **`uid`**: P2P ID (e.g., `WYZEUID1234567890AB`).
- **`enr`**: DTLS encryption key.
- **`mac`**: MAC address of the camera.
- **`model`**: Device model (e.g., `HL_CAM4`, `HL_DB2`).

## Android Root Extraction Strategy (APatch/Magisk)
Since the MVP goal is to function without user-entered credentials by scraping data from an installed Wyze app (`com.hualai.WyzeCam`), we must rely on root access.

### App Data Storage
1. **Shared Preferences**: `com.hualai.WyzeCam` likely stores user session tokens and potentially cached device lists in `/data/data/com.hualai.WyzeCam/shared_prefs/`.
2. **Databases**: SQLite databases in `/data/data/com.hualai.WyzeCam/databases/` (e.g., `wyze_device.db` or similar) will contain cached `mac`, `p2p_id`, and `enr` values.
3. **Keystore Encryption**: Modern Wyze apps (v2.50.0+) likely use Android Keystore to encrypt sensitive Shared Preferences (e.g., `EncryptedSharedPreferences`).
   - **Bypass**: Extracting data might require hooking the application runtime via Xposed/LSPosed or directly querying the SQLite databases if `enr` strings are stored in plaintext.

### Token Extraction Pipeline
- **Method A (Direct File Read via Root)**:
  - Run `su -c 'cat /data/data/com.hualai.WyzeCam/shared_prefs/some_pref.xml'`
  - Search for `access_token` or cached `enr` values.
  - Risk: High likelihood of encryption.
- **Method B (Memory / Runtime Extraction)**:
  - Utilize APatch modules to inject a payload into `com.hualai.WyzeCam` to dump the decrypted `enr` and `uid` keys.
- **Method C (Local MITM / Intent Sniffing)**:
  - Monitor local broadcasts or utilize root to intercept the app's local network traffic to the camera to sniff the `enr`/`uid` during the initial handshake, though DTLS makes sniffing the payload difficult without the key.

## go2rtc.yaml Automation Pipeline
The MVP requires an automated bridge generation.
- The Android app must generate a `go2rtc.yaml` dynamically based on the extracted `uid` and `enr`.
- Example format:
  ```yaml
  streams:
    doorbell_local: wyze://192.168.1.50?uid=WYZEXXXXXXXX&enr=YYYYYYYYYYYY&mac=AABBCCDDEEFF&model=HL_DB2&dtls=true
  ```
- By forcing the `ip` parameter, we ensure the bridge targets the LAN IP, preventing internet-bound traffic.

## Architectural Boundaries
- **Cross-Compilation**: `go2rtc` must be cross-compiled to `android/arm64` (aarch64). The resulting binary must be packaged within the APK's `lib/arm64-v8a/` directory or extracted to the app's internal storage and executed via `Runtime.getRuntime().exec()`.
- **NDK vs. Binary Executable**: Running the Go binary as a standalone child process is vastly simpler than building it as a C-shared library and calling it via JNI, though lifecycle management (orphaned processes) becomes critical.
