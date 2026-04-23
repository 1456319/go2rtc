# Architect's Journal 🏛️

## Missing Intelligence & Vectors of Confusion
1. **2026+ Wyze App Encryption**: It is highly probable that modern iterations of the Wyze app (`com.hualai.WyzeCam`) utilize Android's `EncryptedSharedPreferences` or SQLCipher for their local databases. If the `enr` and `uid` keys are encrypted at rest using keys tied to the Wyze app's Keystore alias, reading the files directly via `su` will yield useless encrypted byte arrays.
   - *Mitigation*: We must build the app with a graceful fallback. If root extraction fails due to encryption, the app must fall back to requesting the user's Wyze API ID and Secret to hit the cloud API *once* to fetch the P2P details, circumventing local app extraction entirely.
2. **Gwell vs TUTK Protocols**: Newer cameras (like Wyze Cam OG, OG Telephoto) may use "Gwell*" protocols that are not supported by the current go2rtc Wyze integration. If the user possesses one of these, the bypass will fail. We need to clearly identify the device model during extraction and abort if it's unsupported.
3. **Dynamic IP Assignments**: If the doorbell camera is on DHCP, its IP will change. The go2rtc YAML requires an IP (`wyze://[IP]?uid=...`).
   - *Mitigation*: The app must actively resolve the IP via mDNS or by matching the extracted MAC address against the local ARP table before regenerating the `go2rtc.yaml` file.

## Potential JNI/NDK Boundary Risks
- **Orphaned Processes**: Executing the Go binary via `Runtime.getRuntime().exec()` creates a child process. If the Android OS forcefully kills our Foreground Service (e.g., due to OOM killer), the child process might be orphaned and continue running, holding onto the port (e.g., 1984 or 8554).
  - *Solution*: We must capture the PID of the go binary and ensure we hook Android application termination signals to SIGKILL the child. Alternatively, we write a shell wrapper that detects if the parent PID is dead and self-terminates.
- **Cross-Compilation CGO Issues**: `go2rtc` relies on standard networking, but if any underlying CGO libraries are utilized for DTLS, compiling for `android/arm64` might require specific NDK toolchains. We must ensure `CGO_ENABLED=0` works for the wyze package.

## Final Thoughts
The path of least resistance for the MVP is treating `go2rtc` purely as a black-box executable rather than integrating its Go code directly into Android via gomobile. The highest risk factor is the extraction of the `enr` key from the rooted device.
