# VoidWalker (ksu-frida fork) - Frida 17.9.1 Android 14 Fix

## Summary of Changes

### 1. Frida Version Update (module.gradle)
- Updated `fridaVersion` from `17.4.0` to `17.9.1`
- Changed download URL from appknox/knox-frida-patcher to official frida-project.org releases

### 2. Android 14 Compatibility (build.gradle, Android.mk)
- Target SDK updated to API level 34
- Added `--hash-style=both` linker flags for GNU+hash compatibility
- NDK version: r25c (25.2.9519653)

### 3. Config Schema Extensions (config.h, config.cpp)
- Added `remapper_config` struct for library hiding (Yidun evasion)
- Added `debug_logging` boolean flag
- Default values: remapper disabled, debug logging off

### 4. Runtime Diagnostics (inject.cpp)
- Pointer validity check: warns if handle < 0x1000 (SIGSEGV risk)
- Enhanced error logging with null-safe dlerror() handling
- SELinux execmem hint on gadget load failure

---

## Deploy Commands

```bash
# 1. Build Zygisk variant
cd /workspace
./gradlew clean assembleZygiskRelease

# 2. Push to device
adb push out/magisk_module_zygisk_release/voidwalker-v1.9.1-zygisk-release.zip /data/local/tmp/

# 3. Flash via KernelSU/Magisk
adb shell su -c "magisk --install-module /data/local/tmp/voidwalker-v1.9.1-zygisk-release.zip"

# 4. Reboot
adb reboot
```

---

## Post-Install Configuration

```bash
# Create config directory
adb shell "mkdir -p /data/local/tmp/libsec"

# Push Frida Gadget 17.9.1 arm64 (pre-extracted from build)
adb shell "cd /data/local/tmp/libsec && mv libgadget-arm64.so.xz libsecmon.so.xz && busybox unxz libsecmon.so.xz"

# Push config for target app
adb push config.json.yidun /data/local/tmp/libsec/config.json

# Set permissions
adb shell "chmod 755 /data/local/tmp/libsec"
adb shell "chmod 644 /data/local/tmp/libsec/*"
adb shell "chown root:root /data/local/tmp/libsec/*"
```

---

## Config Template for com.msandroid.mobile

Location: `/data/local/tmp/libsec/config.json`

```json
{
    "targets": [
        {
            "app_name": "com.msandroid.mobile",
            "enabled": true,
            "start_up_delay_ms": 5000,
            "kernel_assisted_evasion": true,
            "injected_libraries": [
                {
                    "path": "/data/local/tmp/libsec/libsecmon.so"
                }
            ],
            "child_gating": {
                "enabled": false,
                "mode": "freeze",
                "injected_libraries": []
            },
            "remapper": {
                "enabled": true,
                "hide_library_name": "libsecond.so"
            },
            "debug_logging": true
        }
    ]
}
```

---

## SELinux Policy Hints

If gadget fails to load with SIGSEGV or connection closes immediately:

```bash
# Check current SELinux mode
adb shell getenforce

# Temporarily set permissive (for testing only)
adb shell su -c "setenforce 0"

# If execmem denial suspected, add allow rule:
adb shell su -c "supolicy --live 'allow zygote self:process execmem'"

# Restore enforcing after testing
adb shell su -c "setenforce 1"
```

---

## Diagnostic One-Liner

```bash
adb logcat | grep -iE "gadget|sigsegv|yidun|VoidWalker|dlopen|xdl"
```

---

## Frida Attach Test

```bash
# Wait 5 seconds after app launch for delay_start_up to complete
frida -U -n com.msandroid.mobile -l your_script.js
```

---

## SHA256 Verification (Frida 17.9.1 arm64)

Official gadget SHA256 (verify after download):
```bash
curl -sL https://github.com/frida/frida/releases/download/17.9.1/frida-gadget-17.9.1-android-arm64.so.xz \
  | xz -d | sha256sum
```

Expected hash format: `SHA256: <64-char-hex>` (check frida-project.org for exact value)

---

## Troubleshooting

| Symptom | Solution |
|---------|----------|
| Connection closes immediately | Increase `start_up_delay_ms` to 8000-10000 |
| SIGSEGV in logcat | Check pointer validity logs, verify SELinux permissive |
| Gadget not found | Confirm `/data/local/tmp/libsec/libsecmon.so` exists |
| Yidun detects injection | Enable `remapper.hide_library_name`, try different name |
| dlopen fails with "execmem" | Add SELinux execmem allowance or use permissive mode |

---

## File Changes Summary

```
module.gradle          - Frida version + URL update
build.gradle           - targetSdkVersion 34, --hash-style flags
Android.mk             - LOCAL_LDFLAGS += --hash-style=both
config.h               - Added remapper_config, debug_logging fields
config.cpp             - Parse new config fields
inject.cpp             - Runtime checks, enhanced error logging
config.json.yidun      - New template for com.msandroid.mobile
```
