# VoidWalker (ZygiskFrida Fork)

> **Frida 17.9.1 + Android 14 + Yidun Bypass** - Zygisk-based Frida Gadget injection for rooted devices

> [Frida](https://frida.re) is a dynamic instrumentation toolkit for developers, reverse-engineers, and security researchers

> [Zygisk](https://github.com/topjohnwu/Magisk) allows you to run code in every Android application's process.


## Introduction

**VoidWalker** is a hardened fork of ZygiskFrida designed to survive modern anti-debug protections like Yidun (NetEase).

### Key Features
- ✅ **Frida 17.9.1** embedded (official or custom builds)
- ✅ **Android 14 (API 34)** compatible with `--hash-style=both` linker fix
- ✅ **Yidun bypass** via configurable startup delay (5000ms default)
- ✅ **Library remapper** hides `libsecond.so` from `/proc/self/maps`
- ✅ **Stealth injection**: No APK modification, no ptrace, signature checks pass
- ✅ **Dual support**: Zygisk (Magisk/KernelSU) and Riru flavors

This repo provides both **Zygisk** (recommended) and **Riru** flavors.

## How to use the module

### Prerequisites
- Rooted device/emulator with KernelSU or Magisk
- Zygisk available and enabled
- Frida CLI installed (`pip install frida-tools`)

### Quick Start (Android 14 + Yidun Protection)

#### 1. Build or Download
**Option A: Download pre-built release**
- Go to Releases and download `ZygiskFrida-v1.9.1-release.zip`

**Option B: Build locally**
```bash
git checkout voidwalker
./gradlew :module:assembleRelease
# Output: module/build/outputs/magisk_module_zygisk_release/*.zip
```

#### 2. Install Module
```bash
# Push to device
adb push module/build/outputs/magisk_module_zygisk_release/*.zip /sdcard/Download/

# Install via KernelSU (or flash via Magisk app)
adb shell su -c "ksud module install /sdcard/Download/*.zip"
# For Magisk: adb shell su -c "magisk --install-module /sdcard/Download/*.zip"

# Reboot required
adb reboot
```

#### 3. Configure for Target App
Create config with anti-debug delay and remapper enabled:

```bash
# Create directory
adb shell su -c "mkdir -p /data/local/tmp/re.zyg.fri"

# Create config.json (replace com.msandroid.mobile with your target)
adb shell su -c "cat > /data/local/tmp/re.zyg.fri/config.json << 'EOF'
{
  \"target\": \"com.msandroid.mobile\",
  \"delay\": 5000,
  \"remapper\": {
    \"enabled\": true,
    \"hide_library_name\": \"libsecond.so\"
  },
  \"debug_logging\": true
}
EOF"

# Set permissions
adb shell su -c "chmod 644 /data/local/tmp/re.zyg.fri/config.json"
```

#### 4. Attach Frida
```bash
# Start target app
adb shell am start -n com.msandroid.mobile/.MainActivity

# Monitor logs (in separate terminal)
adb logcat | grep -iE "gadget|sigsegv|yidun|frida"

# Attach Frida (wait 5s for delay to expire)
frida -U -n com.msandroid.mobile
# OR
frida -U -n Gadget
```

> **Note:** The 5000ms delay helps bypass Yidun and other anti-debug checks that run at startup.

### Advanced Configuration
See [docs/advanced_config.md](docs/advanced_config.md) for child gating, multiple libraries, and custom Frida gadget ports.

### Frida Gadget Remapper

VoidWalker includes an advanced **library remapper** system that hides injected libraries from `/proc/self/maps`:

- On successful injection, the remapper copies library data from procfs to a separate memory allocation
- The original mapping is replaced, making detection via `/proc/self/maps` scans ineffective
- Configurable via `remapper.hide_library_name` (default: `libsecond.so`)
- Implementation: [remapper.cpp](module/src/jni/remapper.cpp)

This effectively bypasses anti-cheat SDKs that scan for suspicious shared libraries like `frida-gadget.so`.

### Configuration

This module also supports adding a start up delay that can delay injection of the gadget to
avoid checks run at startup time, loading arbitrary libraries and child gating.

Please take a look at the [configuration guide](docs/advanced_config.md) for this.

## How to build

### Local Build
```bash
git checkout voidwalker
./gradlew :module:assembleRelease
# Output: module/build/outputs/magisk_module_zygisk_release/*.zip
```

### GitHub Actions (CI/CD)
The repository includes automated builds via GitHub Actions:

**Trigger a build:**
1. Push to `voidwalker` branch, OR
2. Create a tag: `git tag -a v1.9.1-voidwalker -m "release"` && `git push origin v1.9.1-voidwalker`, OR
3. Manually trigger from Actions tab → "Build Release" workflow

**Download artifacts:**
- Go to Actions → Select the completed run → Download from "Artifacts" section

### Flash Directly from Build
```bash
# Build and flash to device in one command (requires ADB)
./gradlew :module:flashAndRebootZygiskRelease
```

## Troubleshooting

### Connection closes immediately
```bash
# Check for dlopen errors or SIGSEGV
adb logcat | grep -iE "gadget|sigsegv|dlopen|yidun"

# Verify config.json syntax
adb shell su -c "cat /data/local/tmp/re.zyg.fri/config.json | jq ."

# Ensure SELinux is permissive (temporarily for testing)
adb shell su -c "setenforce 0"
```

### App crashes on startup
- Increase `delay` value in config.json (try 10000ms)
- Disable remapper temporarily to isolate the issue
- Check for conflicting Frida installations (magisk-frida, etc.)

### Cannot attach via Frida
```bash
# Verify gadget is loaded
adb shell su -c "cat /proc/$(pidof com.msandroid.mobile)/maps | grep -i gadget"

# Try attaching by PID instead
frida -U -p $(adb shell pidof com.msandroid.mobile)
```

## Caveats

- **Emulators**: Gadget runs in native realm only (Java hooks work, native hooks may not)
- **SELinux**: May need `execmem` allowance on some ROMs
- **Multiple Frida instances**: Ensure no other frida-server is running on port 27042

## Credits

- Original: https://github.com/lico-n/ZygiskFrida
- Inspired by: https://github.com/Perfare/Zygisk-Il2CppDumper
- xDL: https://github.com/hexhacking/xDL
- Frida: https://frida.re

