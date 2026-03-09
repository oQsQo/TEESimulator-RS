<p align="center">
  <h1 align="center">🔐 TEESimulator</h1>
  <p align="center"><b>Full TEE Emulation for Rooted Android</b></p>
  <p align="center">Hardware attestation. Software keys. Zero detection.</p>
  <p align="center">
    <img src="https://img.shields.io/badge/version-v4.0-blue?style=for-the-badge" alt="v4.0">
    <img src="https://img.shields.io/badge/Android-10%2B-green?style=for-the-badge&logo=android" alt="Android 10+">
    <img src="https://img.shields.io/badge/Telegram-community-blue?style=for-the-badge&logo=telegram" alt="Telegram">
  </p>
</p>

---

> [!NOTE]
> **This is a personal fork of [JingMatrix/TEESimulator](https://github.com/JingMatrix/TEESimulator)** with additional hardening, native Rust certificate generation, key persistence, and anti-detection features. For the upstream project, see the original repo.

---

## 🧬 What is TEESimulator?

TEESimulator is a **complete software simulation** of Android's hardware-backed [Trusted Execution Environment](https://source.android.com/docs/security/features/trusty) for [Key Attestation](https://developer.android.com/privacy-and-security/security-key-attestation). Instead of patching certificates from the real TEE after the fact, TEESimulator intercepts Binder IPC at the `ioctl` level and generates entire certificate chains from scratch — signed by your keybox, with correct attestation extensions, indistinguishable from hardware-generated keys.

The result: **apps that verify hardware attestation see a legitimate, unmodified device** — even on rooted hardware with an unlocked bootloader.

> **This is not TrickyStore.** TEESimulator replaces TrickyStore and its forks entirely. It shares the same config paths for drop-in compatibility, but the architecture is fundamentally different: native Rust certificate generation, binder-level interception via `lsplt`, per-UID rate limiting, key persistence, and a multi-layer defense against detector apps.

---

## 🔥 Why TEESimulator?

🔐 **Native Cert Generation** — v4.0 generates X.509 certificate chains in Rust with `ring` and manual DER encoding. No BouncyCastle overhead, no Java crypto quirks, byte-perfect issuer chain linkage.

🎯 **Binder-Level Interception** — Hooks `ioctl()` on `libc.so` via `lsplt` inside the `keystore2` process. Intercepts `generateKey`, `importKey`, and `getKeyEntry` transactions before the HAL ever sees them.

🛡️ **Detector Resistant** — Per-UID rate limiting blocks DuckDetector-style keygen flooding. Oversized challenges rejected with real KeyMint error codes. Chain consistency verified byte-for-byte.

💾 **Key Persistence** — Generated keys survive reboots. Apps that store attestation keys (banking, biometrics) don't break after a restart.

🔧 **Drop-In Replacement** — Same config paths as TrickyStore (`/data/adb/tricky_store/`). Swap the module ZIP, keep your keybox and target list.

---

## ✨ Features

**Core Attestation Engine**
- [x] **Full certificate chain generation** — leaf + intermediates + root, signed by your keybox
- [x] **Native Rust certgen** — `libcertgen.so` built with `ring`, `rsa`, and manual DER assembly
- [x] **BouncyCastle fallback** — unsupported curves (P-224, P-521, Curve25519) fall back to Java
- [x] **ASN.1 attestation extensions** — OID 1.3.6.1.4.1.11129.2.1.17 with all AOSP-specified tags
- [x] **Multi-keybox support** — different keybox files per app group via `target.txt`

**Interception Layer**
- [x] **Binder ioctl hook** — `lsplt` PLT hook on `libc.so` inside `keystore2` process
- [x] **generateKey / importKey / getKeyEntry** — all three transaction types intercepted
- [x] **256KB native payload cap** — oversized binder payloads bypass interception cleanly
- [x] **Challenge validation** — rejects >128-byte attestation challenges with `INVALID_INPUT_LENGTH`

**Hardening**
- [x] **Per-UID rate limiter** — 2 hardware keygens per 30s burst window, software fallback on overflow
- [x] **importKey eviction guard** — retained patch chains prevent generate-then-import cache attacks
- [x] **Key persistence** — file-backed storage with file-level locking, survives reboots and keybox rotations
- [x] **Global exception handler** — uncaught exceptions logged, daemon stays alive

**Configuration**
- [x] **Live config reload** — `FileObserver` watches all config files, changes apply immediately
- [x] **Security patch spoofing** — per-package `system`, `vendor`, `boot` patch levels with dynamic templates
- [x] **Lifecycle scripts** — KSU Action button clears key cache, uninstall removes all traces

---

## 📋 Requirements

> [!IMPORTANT]
> TEESimulator requires root access and a valid `keybox.xml` for hardware-level attestation results. Without a keybox, the module generates software-level certificates that won't pass strict hardware attestation checks.

**You need:**
1. Android 10 or above
2. A supported root manager (KernelSU, Magisk, or APatch)
3. A hardware-backed `keybox.xml` placed at `/data/adb/tricky_store/keybox.xml`

---

## 📱 Compatibility

### Root Managers

| Manager | Status | Notes |
|---|---|---|
| KernelSU | ✅ Tested | Full support including Action button and lifecycle scripts |
| Magisk | ✅ Supported | Standard module install |
| APatch | ✅ Supported | Standard module install |

### Tested Devices

| Device | Android | TEE | Status |
|---|---|---|---|
| Redmi 14C (2409BRN2CA) | 14 (SDK 34) | Beanpod KeyMaster | ✅ Daily driver |

> Tested against DuckDetector, Luna, Play Integrity, and Key Attestation Demo. If you test on a different device, [open an issue](https://github.com/Enginex0/TEESimulator/issues) with your results.

---

## 🚀 Quick Start

1. **Download** the latest release ZIP from [Releases](https://github.com/Enginex0/TEESimulator/releases)
2. **Install** via your root manager (KSU / Magisk / APatch) and reboot
3. **Place your keybox** at `/data/adb/tricky_store/keybox.xml`
4. **Configure targets** in `/data/adb/tricky_store/target.txt`
5. **Verify** — check Play Integrity or run Key Attestation Demo

TEESimulator replaces TrickyStore, TrickyStoreOSS, and their forks. Existing config files are compatible.

---

## ⚙️ Configuration

All configuration files live at `/data/adb/tricky_store/` and are monitored by `FileObserver` — changes take effect immediately without rebooting.

### The `keybox.xml` Root of Trust

This file provides the master cryptographic identity. It contains a private key and a hardware-backed certificate chain from a real device. TEESimulator signs all generated certificates with this key, making them appear legitimate to verifiers.

```xml
<?xml version="1.0"?>
<AndroidAttestation>
    <Keybox DeviceID="...">
        <Key algorithm="ecdsa|rsa">
            <PrivateKey format="pem">...</PrivateKey>
            <CertificateChain>...</CertificateChain>
        </Key>
    </Keybox>
</AndroidAttestation>
```

### Target Packages (`target.txt`)

Controls which apps get intercepted and what simulation mode to use.

#### Mode Suffixes

*   **`!` → Force Generation** — Creates a complete software-based virtual key. Full TEE simulation.
*   **`?` → Force Leaf Hacking** — Real TEE key generated, but its attestation certificate is intercepted and patched.
*   **No symbol → Automatic** — Module selects the best mode for your device.

#### Multi-Keybox

Specify different keybox files for different app groups. Apps listed after a `[filename.xml]` line use that keybox. Apps before any declaration use the default `keybox.xml`.

```
# Default keybox
com.google.android.gms!
io.github.vvb2060.keyattestation?

# Switch to a different keybox for the following apps
[aosp_keybox.xml]
com.google.android.gsf

# Another keybox
[demo_keybox.xml]
org.matrix.demo
```

### Security Patch Level (`security_patch.txt`)

Configure the `osPatchLevel`, `vendorPatchLevel`, and `bootPatchLevel` reported in attestation certificates. This only affects attestation data — it does not change actual system properties.

#### Global and Per-Package

Settings at the top of the file are global defaults. Add `[package.name]` to override for specific apps.

#### Keys

| Key | Scope |
|---|---|
| `system` | OS patch level |
| `vendor` | Vendor patch level |
| `boot` | Boot/kernel patch level |
| `all` | Shorthand — sets all three at once |

#### Special Keywords

| Keyword | Effect |
|---|---|
| `today` | Current date, dynamically resolved on each attestation |
| `YYYY-MM-DD` templates | Semi-dynamic — `YYYY-MM-05` resolves to the 5th of the current month |
| `no` | Omit this patch level tag entirely from the attestation |
| `device_default` | Use the device's real hardware value |
| `prop` | Read from `ro.build.version.security_patch` (matches what detectors see via getprop) |

#### Example

```
# Global — default for all apps
system=YYYY-MM-05
vendor=device_default
boot=no

# Override for GMS
[com.google.android.gms]
system=2024-10-01

# Custom config for a demo app
[org.matrix.demo]
all=2025-09-15
boot=device_default
```

---

## 💬 Community

<p align="center">
  <a href="https://t.me/superpowers9">
    <img src="https://img.shields.io/badge/⚡_JOIN_THE_GRID-SuperPowers_Telegram-black?style=for-the-badge&logo=telegram&logoColor=cyan&labelColor=0d1117&color=00d4ff" alt="Telegram">
  </a>
</p>

---

## 🙏 Credits

- **[JingMatrix](https://github.com/JingMatrix/TEESimulator)** — original author of TEESimulator and the interception architecture
- **[5ec1cff](https://github.com/5ec1cff/TrickyStore)** — TrickyStore, the project that pioneered keystore interception on Android
- **[LSPlt](https://github.com/LSPosed/LSPlt)** — PLT hook library used for binder interception
- **[ring](https://github.com/briansmith/ring)** — Rust cryptography library powering native cert generation

---

## 📄 License

This project is licensed under the [GNU General Public License v3.0](LICENSE).

---

<p align="center">
  <b>🔐 Because the best attestation is the one the TEE never generated.</b>
</p>
