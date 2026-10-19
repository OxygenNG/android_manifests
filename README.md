# cepheus manifests

Local manifests for building custom Android ROMs on the Xiaomi Mi 9
(codename `cepheus`, SoC SM8150).

Each branch is one ROM. Pick the branch matching what you want to build.

| Branch | ROM | Android | Notes |
|---|---|---|---|
| `uwu-17.0` | uwuAOSP 17.0 | 17 (`android-17.0.0_r1`) | In progress |
| `aviumui-16.2` | AviumUI 16.2.2 | 16 (`lineage-23.2`) | Boots with working audio |

## Usage

```bash
# 1. Init the ROM's own manifest first
#    (see the branch table below for the URL and branch)
repo init -u <rom-manifest-url> -b <rom-branch> --git-lfs --depth=1

# 2. Drop in the local manifest
mkdir -p .repo/local_manifests
curl -fsSL -o .repo/local_manifests/cepheus.xml \
  https://raw.githubusercontent.com/OxygenNG/android_manifests/<this-branch>/cepheus.xml

# 3. Sync
repo sync -c -j8
```

> **Use the patched repo.** The upstream `repo` loses the `--depth=1`
> constraint after an interrupted fetch and silently turns into a full
> clone, which balloons `.repo` past 250 GB instead of ~28 GB. uwuAOSP
> maintains a fix in `UwUniverse/git-repo`
> (commit `5e3f2e68 project: Preserve shallow depth after an incomplete fetch`).

---

## Branch: `uwu-17.0`

### Init

```bash
repo init -u https://github.com/uwuAOSP/platform_manifests.git \
     -b uwu-17.0 --git-lfs --depth=1
mkdir -p .repo/local_manifests
curl -fsSL -o .repo/local_manifests/cepheus.xml \
  https://raw.githubusercontent.com/OxygenNG/android_manifests/uwu-17.0/cepheus.xml
repo sync -c -j8
```

### Build

```bash
source build/envsetup.sh
lunch uwu_cepheus-cp2a-eng     # or -userdebug / -user
m uwu -j10
```

### Project sources

| Path | Repository | Branch |
|---|---|---|
| `device/xiaomi/cepheus` | `OxygenNG/android_device_xiaomi_cepheus-uwu` | `oxygenng/uwu-a17` |
| `vendor/xiaomi/cepheus` | `OxygenNG/vendor_xiaomi_cepheus` | `oxygenng/uwu-a17` |
| `hardware/xiaomi` | `LineageOS/android_hardware_xiaomi` | `lineage-24.0` |
| `kernel/xiaomi/sm8150` | `JoK3rLeE/android_kernel_xiaomi_sm8150` | `17.0-stable` |

The device and vendor repos are forks of the PixelOS trees with the
uwuAOSP port applied. Their commit messages document every change.

### Port notes

- `uwu_cepheus.mk` must declare `UWU_DEVICE_TYPE := phone` and
  `UWU_SUPPORTS_TELEPHONY := true` before inheriting
  `vendor/uwu/config/common.mk`, which hard-errors otherwise.
- `BOARD_BOOT_HEADER_VERSION := 2` is required. Android 17's soong fsgen
  aborts with *"BOARD_RECOVERY_MKBOOTIMG_ARGS or
  BOARD_BOOT_HEADER_VERSION must set --header_version"* when unset.
- Use `TARGET_RELEASE=cp2a`. The `bp4a` flag set is missing
  `RELEASE_WEBAPP_MODULE`, which leaves `framework-webapp` disabled while
  `prebuilts/module_sdk/WebApp` still references it.
- LiveDisplay is dropped: uwuAOSP ships only the AIDL variant, not the
  LineageOS HIDL `vendor.lineage.livedisplay@2.1`.
- Several prebuilt modules only exist as 64-bit. uwuAOSP builds both ABIs,
  so their `arm` variants and dependencies were removed.
- `vendor/uwu-stub/Android.bp` must stay enabled — it provides the
  graph-only placeholders (`MotionTestDefaults`, `rdroidtest.defaults`,
  `tradefeed_*`) that pruned CTS/platform_testing modules still reference.

### Download size

With the patched repo: ≤28 GB transfer, ≤125 GB checkout. A stock repo
can produce 250 GB+ of `.repo` git objects.

---

## Branch: `aviumui-16.2`

Builds AviumUI 16.2.2 on `lineage-23.2` (Android 16,
`BP4A.251205.006`). Vanilla only — `cepheus` is a legacy A-only device
with a 6.4 GB super partition (system 3.5G + vendor 1.5G + cust 1G), too
small for a GMS build.

### Init

```bash
repo init -u https://github.com/AviumUI/android.git -b 16.2 --git-lfs --depth=1
mkdir -p .repo/local_manifests
curl -fsSL -o .repo/local_manifests/cepheus.xml \
  https://raw.githubusercontent.com/OxygenNG/android_manifests/aviumui-16.2/cepheus.xml
repo sync -c -j8
```

### Build

```bash
source build/envsetup.sh
lunch lineage_cepheus-bp4a-eng    # or -userdebug
mka bacon -j10
```

### Project sources

| Path | Repository | Branch |
|---|---|---|
| `device/xiaomi/cepheus` | `OxygenNG/android_device_xiaomi_cepheus` | `oxygenng/cepheus-a16` |
| `device/xiaomi/sm8150-common` | `LineageOS/android_device_xiaomi_sm8150-common` | `lineage-23.2` |
| `vendor/xiaomi/cepheus` | `OxygenNG/proprietary_vendor_xiaomi_cepheus` | `oxygenng/cepheus-a16` |
| `vendor/xiaomi/sm8150-common` | `OxygenNG/proprietary_vendor_xiaomi_sm8150-common` | `oxygenng/cepheus-a16` |
| `hardware/xiaomi` | `LineageOS/android_hardware_xiaomi` | `lineage-23.2` |
| `kernel/xiaomi/sm8150` | `JoK3rLeE/android_kernel_xiaomi_sm8150` | `17.0-stable` |

### Fixes applied

Three independent problems had to be solved before the device would boot.

**1. audioroute ABI mismatch.** The stock vendor audio blobs link against
the VNDK v34 `libaudioroute`, but Android 16 ships the AOSP one with an
incompatible ABI. The HAL died in `platform_init`
(`mixer_ctl_get_type+10`, fault addr `0x40`). Fix: rewrite `DT_NEEDED` on
the blobs and ship `libaudioroute-v34` from `hardware/lineage/compat`.

**2. Missing `ssr_setup`.** `init.qcom.rc` declares a `vendor.ssr_setup`
service but the binary was absent from the blob tree.

**3. Stale `locksettings.db`.** This was the actual boot blocker, and it
is not a build problem. After the two fixes above the HAL stopped
crashing, yet the device still hung on the boot animation, restarting
every ~30 s. `system_server` was dying because
`/data/system/locksettings.db` was owned by `root:root` with mode 0600
and zero bytes — a leftover from a previous ROM flash:

```
SQLiteCantOpenDatabaseException: code 14 SQLITE_CANTOPEN: Permission denied
    at LockSettingsStorage.writeKeyValue
    at TrustManagerService.onBootPhase
E/Zygote: Exit zygote because system server (pid 3187) has terminated
```

The audio crash that remained was collateral: the HAL was torn down when
`system_server` died. Because `system_server` dies from a Java exception
it writes no native tombstone, so every tombstone showed only the audio
HAL stack. Fix: `adb shell rm -f /data/system/locksettings.db` and
reboot. A clean flash will not hit this.

### Kernel notes

The kernel must be **JoK3rLeE 17.0-stable** (4.14.357-openela). Tested
alternatives that do not work:

| Kernel | Result |
|---|---|
| LineageOS `lineage-23.2` (4.14.356-openela-rc1) | Hangs on first splash, `boot.reason=reboot,bpfloader-failed`. Only 16 BPF sources, no BTF/bpf_iter/ringbuf. |
| `sm8150-playground` `sixteen` | `cepheus.config` is 40 bytes, missing cs35l41/idtp9220. |
| crDroid 16.0 kernel + our ramdisk | Does not boot. |

`ro.bpf.kver_override=5.10.239` must stay — Android 16 needs the BPF
version spoof plus the matching kernel backport.

### Verified

Boots to `sys.boot_completed=1`, audio HAL stable, all output streams
(low-latency, deep-buffer, afe-proxy) open, playback audible.
