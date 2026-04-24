# Architect's Journal 🏛️

## Current Status
Research and roadmap generation for Wyze-Android-Bypass MVP.

## Intelligence Gathered
- The `go2rtc` codebase has a robust Wyze implementation (added in v1.9.14) that utilizes native P2P protocols with DTLS encryption.
- Connection requires a specific URL scheme: `wyze://[IP]?uid=[P2P_ID]&enr=[ENR]&mac=[MAC]&model=[MODEL]&dtls=true`.
- The primary cloud endpoints are `https://auth-prod.api.wyze.com` and `https://api.wyzecam.com`.
- Root extraction (APatch) will target `/data/data/` directories for the Wyze app to scrape SQLite databases or SharedPreferences for the `uid`, `mac`, and crucially, the `enr` (encryption key).

## Missing Intelligence & Vectors of Confusion
1. **2026+ Wyze App Encryption & Database Schema**: We lack the exact table structures for `com.hualai.WyzeCam` (or `com.wyze.smarthome`) SQLite databases and do not know if the `enr`, `uid`, and `mac` are stored in plaintext, obfuscated, or strictly encrypted using Android Keystore. It is highly probable that modern iterations utilize Android's `EncryptedSharedPreferences` or SQLCipher. If keys are encrypted at rest using keys tied to the Wyze app's Keystore alias, reading files directly via `su` will yield useless encrypted byte arrays. (Note: we will not be using tools like Frida or memory dumping to circumvent this, but rely purely on file reads).
   - *Mitigation*: We must build the app with a graceful fallback. If root extraction fails due to encryption, the app must fall back to requesting the user's Wyze API ID and Secret to hit the cloud API *once* to fetch the P2P details, circumventing local app extraction entirely.
2. **Package Name**: The exact package name of the 2026 Wyze app must be confirmed during the development phase (historically `com.hualai.WyzeCam` or `com.wyze.smarthome`).
3. **Gwell vs TUTK Protocols**: Newer cameras (like Wyze Cam OG, OG Telephoto) may use "Gwell*" protocols that are not supported by the current go2rtc Wyze integration. We need to verify if any Wyze Doorbells use Gwell instead of TUTK. Currently, Video Doorbell v2 uses TUTK. We need to clearly identify the device model during extraction and abort if it's unsupported.
4. **Dynamic IP Assignments**: If the doorbell camera is on DHCP, its IP will change. The go2rtc YAML requires an IP (`wyze://[IP]?uid=...`).
   - *Mitigation*: The app must actively resolve the IP via mDNS or by matching the extracted MAC address against the local ARP table before regenerating the `go2rtc.yaml` file.

## Conflicting Documentation
1. **Authentication Source:** The `go2rtc` README states: "Requires Wyze account. You need to login once via the WebUI to load your cameras... Internet access is only needed when loading cameras from your account. After that, all streaming is local P2P." However, our goal is *local token scraping* via APatch. If the app relies on scraping, the web API login might be entirely bypassed, contradicting the standard `go2rtc` flow. The roadmap accommodates this by generating the `wyze://` URL manually if scraping succeeds.

## Potential JNI/NDK Boundary Risks
- **Orphaned Processes**: Executing the Go binary via `Runtime.getRuntime().exec()` creates a child process. If the Android OS forcefully kills our Foreground Service (e.g., due to OOM killer), the child process might be orphaned and continue running, holding onto the port (e.g., 1984 or 8554).
  - *Solution*: We must capture the PID of the go binary and ensure we hook Android application termination signals to SIGKILL the child. Alternatively, we write a shell wrapper that detects if the parent PID is dead and self-terminates.
- **Cross-Compilation & JNI/NDK Issues**: `go2rtc` relies on standard networking, but if any underlying CGO libraries are utilized for DTLS, compiling for `android/arm64` might require specific NDK toolchains. We must ensure `CGO_ENABLED=0` works for the wyze package.
- If we were to run `go2rtc` directly within the Android application context, it would require compiling the Go code to a shared library (`.so`) using `gomobile` or `go build -buildmode=c-shared`, and the Android UI would interface via JNI. Handling network socket creation within the Go runtime inside the Android application sandbox may trigger SELinux denials or require specific network capabilities not granted by default, even with root.
- **Background Execution:** Background execution of the Go runtime will be heavily scrutinized by the Android OS; WakeLocks and Foreground Services are mandatory to prevent the process from being suspended.
- **Future MediaCodec Integration**: While the roadmap currently dictates running `go2rtc` as a standalone binary via `ProcessBuilder`, future iterations might require tighter integration (e.g., getting the raw H.264 frames directly into an Android `MediaCodec` surface without the RTSP/WebRTC overhead).
- **Go GC / Dalvik-ART Interaction**: If we move to JNI, we must handle Go's garbage collector interacting with Dalvik/ART. Passing large byte arrays (video frames) across the JNI boundary can cause significant performance overhead and memory leaks if not properly managed with direct `ByteBuffer`s.
- **Recommendation for MVP:** Avoid JNI entirely. Use standard network loopback (`localhost`) to consume the stream from the embedded `go2rtc` process.

## Final Thoughts
The path of least resistance for the MVP is treating `go2rtc` purely as a black-box executable rather than integrating its Go code directly into Android via gomobile. The highest risk factor is the extraction of the `enr` key from the rooted device.
