# AviumUI 16.2.2 on Xiaomi Mi 9 (cepheus)

Working build notes for Android 16 (lineage-23.2, BP4A.251205.006) on the
Xiaomi Mi 9 / Redmi K20 Pro (codename `cepheus`, SoC SM8150).

The device boots to a fully working system with audio, on the `eng` and
`userdebug` variants, built as **vanilla** (`WITH_GMS=false`).

## Repositories

| Path | Repository | Branch |
|---|---|---|
| `device/xiaomi/cepheus` | `OxygenNG/android_device_xiaomi_cepheus` | `oxygenng/cepheus-a16` |
| `device/xiaomi/sm8150-common` | `LineageOS/android_device_xiaomi_sm8150-common` | `lineage-23.2` |
| `hardware/xiaomi` | `LineageOS/android_hardware_xiaomi` | `lineage-23.2` |
| `kernel/xiaomi/sm8150` | `OxygenNG/kernel_xiaomi_sm8150` | `oxygenng/cepheus-a16` |
| `vendor/xiaomi/cepheus` | `OxygenNG/proprietary_vendor_xiaomi_cepheus` | `oxygenng/cepheus-a16` |
| `vendor/xiaomi/sm8150-common` | `OxygenNG/proprietary_vendor_xiaomi_sm8150-common` | `oxygenng/cepheus-a16` |

## Building

```bash
mkdir -p .repo/local_manifests
cp cepheus.xml .repo/local_manifests/
repo sync -c -j8 --force-sync --no-clone-bundle --no-tags

source build/envsetup.sh
lunch lineage_cepheus-bp4a-userdebug      # or -eng
mka bacon -j10
```

### Vanilla, not GMS

`cepheus` is a legacy A-only device with a 6.4 GB super partition
(system 3.5G + vendor 1.5G + cust 1G). A GMS build needs ~2.4 GB for the
system partition alone and does not fit. `lineage_cepheus.mk` therefore
sets `WITH_GMS=false`.

## Fixes applied

Three separate problems had to be fixed before the device would boot.

### 1. audioroute ABI mismatch (audio HAL crash)

The vendor audio blobs are unmodified stock-firmware binaries linked
against the VNDK v34 `libaudioroute`. Android 16's vendor image provides
the AOSP `libaudioroute.so` instead, whose ABI differs. The HAL died in
`platform_init`:

```
#00 libtinyalsa.so       mixer_ctl_get_type+10       (NULL deref, addr 0x40)
#01 libaudioroute.so     audio_route_update_mixer+28
#02 libaudioroute_ext.so parse_default_ctl+150
#03 libaudioroute_ext.so audio_route_init_ext+400
#04 audio.primary.cepheus.so platform_init+1260
```

Fix: rewrite `DT_NEEDED` on both blobs and ship the v34 variant
(`hardware/lineage/compat`):

```bash
patchelf --replace-needed libaudioroute.so libaudioroute-v34.so \
    proprietary/vendor/lib/libaudioroute_ext.so \
    proprietary/vendor/lib/hw/audio.primary.cepheus.so
```

Plus `libaudioroute-v34` in `PRODUCT_PACKAGES` and in both modules'
`shared_libs`.

### 2. Missing `ssr_setup` blob

`init.qcom.rc` declares a `vendor.ssr_setup` service running
`/vendor/bin/ssr_setup`, but the binary was absent from the blob tree.
Extracted from stock firmware `V12.5.1.0.RFAMIXM` `vendor.img` and added
as a `cc_prebuilt_binary`.

### 3. Stale `locksettings.db` blocked boot (the real blocker)

After the two fixes above the audio HAL no longer crashed in
`platform_init`, but the device still hung on the boot animation,
restarting every ~30 seconds. The audio crash that remained was a
*symptom*, not the cause:

```
/data/system/locksettings.db  →  root:root, 0600, 0 bytes, stale
        ↓
system_server (uid=system) writes it in TrustManagerService.onBootPhase
        ↓
SQLiteCantOpenDatabaseException: code 14 SQLITE_CANTOPEN: Permission denied
        ↓
system_server dies → "Exit zygote because system server has terminated"
        ↓
audio HAL gets closeDevice → NULL deref in adev_close teardown
```

Because `system_server` dies from a Java exception it never writes a
native tombstone, so every tombstone in `/data/tombstones` showed only
the audio HAL stack. The actual cause was only visible in logcat:

```
E/AndroidRuntime: Caused by: android.database.sqlite.SQLiteCantOpenDatabaseException:
                  unable to open database file (code 14 SQLITE_CANTOPEN): Permission denied
    at LockSettingsStorage.writeKeyValue
    at TrustManagerService.onBootPhase
E/Zygote: Exit zygote because system server (pid 3187) has terminated
```

Fix: remove the stale file (it is recreated on boot):

```bash
adb root
adb shell rm -f /data/system/locksettings.db
adb reboot
```

This is a leftover from a previous ROM flash, not a build problem — but a
clean-flash install will not hit it.

## Kernel

The kernel is **JoK3rLeE 17.0-stable** (4.14.357-openela) from
`sm8150-playground`, which carries the BPF 5.10 backport Android 16 needs.

Alternatives that were tested and do **not** work:

| Kernel | Result |
|---|---|
| LineageOS `lineage-23.2` (4.14.356-openela-rc1) | Hangs on first splash, `boot.reason=reboot,bpfloader-failed`, no auto-reboot. Only 16 BPF sources, no BTF/bpf_iter/ringbuf. |
| `sm8150-playground` `sixteen` | Unusable: `cepheus.config` is 40 bytes (only `CONFIG_MACH_XIAOMI_CEPHEUS=y`), missing cs35l41/idtp9220. |
| crDroid 16.0 kernel + our ramdisk | Does not boot. |

`ro.bpf.kver_override=5.10.239` must stay set — Android 16 needs the BPF
version spoof and the matching kernel backport.

The kernel tree carries one build fix: when `CROSS_COMPILE` is empty the
top-level `Makefile` derives `--target` from `CLANG_TARGET_FLAGS_$(SRCARCH)`
instead of erroring out.

## Verified working

- Boots to `sys.boot_completed=1`, boot animation stops
- `android.hardware.audio.service` stable, all output streams
  (low-latency, deep-buffer, afe-proxy) open, playback audible
- Sound card `sm8150-tavil-snd-card` registers correctly
