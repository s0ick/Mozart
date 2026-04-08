# Mozart

A hardened fork of [Hiddify](https://github.com/hiddify/hiddify-app) that fixes a critical security vulnerability present in most VPN clients for Android.

## The Problem

Standard VPN clients (v2rayNG, Hiddify, Happ, Exclave) have a critical vulnerability: they expose a local SOCKS5/HTTP proxy on `127.0.0.1` without authentication. Any app on the device can:

1. Scan localhost ports
2. Find the proxy port
3. Send a request through it to `ifconfig.me`
4. **Get the real IP of the VPN server**

This allows malicious SDKs embedded in other apps to discover and report your VPN server IP, leading to it being blocked.

Additionally, `/proc/net/route` is readable without root and exposes the server IP through the routing table (mitigated on Android 10+).

Source: [VLESS-SOCKS5 Vulnerability Research](https://publish.obsidian.md/zapret/VLESS-SOCKS5-vulnerability)

## The Fix

### Dart layer (`config_option_repository.dart`)
- Added `enableMixedProxy` preference (default: `false`)
- In TUN mode, sends `mixedPort: 0` to the Go core, preventing proxy creation

### Go core (`v2/config/builder.go`)
- Added `if hopt.MixedPort > 0` check before creating the mixed inbound
- Without this check, `mixedPort: 0` caused the OS to assign a random port — still vulnerable

### Result
- No SOCKS5/HTTP proxy listens on any port in TUN mode
- Verified: full port scan + SOCKS5 probe returns no results

## Other Changes

- **Safe defaults**: per-app proxy whitelist (Telegram, YouTube, Instagram, Chrome, Firefox), RU domains routed DIRECT
- **Simplified UI**: removed advanced settings (ports, TLS tricks, WARP, core selection) on mobile
- **Rebranded**: new name, package `com.mozart.app`, new icon

## Building

### Prerequisites
- Flutter 3.29+
- Go 1.25.x (not 1.26 — Psiphon-TLS compatibility)
- Android NDK
- gomobile (`go install github.com/sagernet/gomobile/cmd/gomobile@v0.1.11`)

### Build Go core (patched)
```bash
cd hiddify-core
git checkout v4.1.0
git submodule update --init --recursive
# Apply the mixed-port fix to v2/config/builder.go
make android
```

### Build APK
```bash
cp hiddify-core/bin/hiddify-core.aar android/app/libs/
flutter pub get
dart run build_runner build --delete-conflicting-outputs
flutter build apk --split-per-abi
```

## Credits

Based on [Hiddify](https://github.com/hiddify/hiddify-app) (MIT License).
