# Porting EmberHeart to New Devices

This guide explains how to add support for a new device (OnePlus, Xiaomi/Redmi/POCO, or any other GKI-based device) to the EmberHeart CI/CD pipeline.

First star this repo, create a fork and follow the instructions given below.

> [!NOTE]
> This guide describes the **current** pipeline (`.github/env_setup`, `.github/kernel`, `.github/kernel_modules`, `.github/telegram` composite actions + `.github/workflows/build-kernel-release.yml`). It replaces an earlier version of this doc that referenced a single `actions/action.yml` and a flat manifest schema — those no longer exist in this repo.

## 0. Prerequisites

To port a new device you need, at minimum:

1. **Device Model Code** — a short identifier used as the config filename and the `model` field (e.g. `OP12`, `REDMI_15`).
2. **SoC Codename** — the chipset codename (e.g. `kalama` for SD 8 Gen 2, `pineapple` for SD 8 Gen 3, `strait` for the MediaTek chip used in some Redmi devices).
3. **Device Codename** — the internal device codename used by the vendor manifest (e.g. `salami` for the OnePlus 11, `sky` for the Redmi 12 5G). This is what CI uses to namespace the per-device build directory and locate patches/keys.
4. **Source Branch** and **Manifest XML** for the kernel/vendor source tree (from the vendor's kernel manifest repo, or a custom one hosted under `nullptr-t-oss/kernel_patches`).
5. **Boot/vendor_boot partition sizes** in bytes for the device (found in its `boot_partition_size`/`vendor_boot_partition_size` in the vendor's `BoardConfig.mk` or an existing dump, or from a working AOSP/LineageOS device tree).

Unlike some other GKI kernel projects, **EmberHeart does not patch a stock `boot.img`** — `.github/kernel` builds a fresh `boot.img` from scratch with `mkbootimg` (v4 header, no ramdisk) using the freshly compiled kernel `Image`, and signs it with `avbtool`. You do not need to supply a stock boot image for the standard flow.

## 1. Create the Device Config File

The `set-model` job in `build-kernel-release.yml` scans `configs/` automatically — every `*.json` file becomes a selectable device. To add a device, create `configs/<MODEL>.json`.

### Schema

```json
{
  "model": "OP12",
  "product": "OnePlus 12 5G",
  "soc": "pineapple",
  "codename": "waffle",
  "branch": "oneplus/sm8650",
  "boot_partition_size": "201326592",
  "vendor_boot_partition_size": "201326592",
  "qcacld_dev_name": "kiwi_v2",
  "clang": "manifest",
  "manifest_oos15": {
    "url": "oneplus_12_v.xml",
    "os": "OxygenOS 15",
    "os_short": "oos15"
  },
  "manifest_oos16_custom": {
    "url": "https://raw.githubusercontent.com/nullptr-t-oss/kernel_patches/refs/heads/main/kernel_manifest/oneplus_12_b.xml",
    "os": "OxygenOS 16",
    "os_short": "oos16"
  },
  "android_version": "android14",
  "kernel_version": "6.1",
  "hmbird": false
}
```

### Field reference

| Field | Meaning |
|---|---|
| `model` | Must exactly match the filename stem (`configs/OP12.json` → `"model": "OP12"`) and the `model:` dropdown option you add in step 5. |
| `product` | Human-readable name, shown in job titles, Telegram notifications, and release notes. |
| `soc` | Chipset codename. Letters/digits/`_`/`-` only. |
| `codename` | Device codename. Used to build the per-device workspace dir (`$CODENAME-$OS_SHORT`) and to locate device-specific patches and the optional `BUILD_CONFIG` hook file (`kernel_patches/device/<codename>-<os_short>/...` — see the build-config-hooks doc). |
| `branch` | Git branch on the manifest source. Use `"dummy"` if the device only ever builds from a fully custom manifest (as `REDMI_12`/`REDMI_15` do). |
| `boot_partition_size` / `vendor_boot_partition_size` | Byte sizes used when AVB-signing `boot.img`/`vendor_boot.img`. |
| `qcacld_dev_name` | QCACLD-3.0 Wi-Fi variant name (e.g. `kiwi_v2`) if the device has a Qualcomm WLAN chip; use `""` for MediaTek devices or devices without this driver. |
| `clang` | `"manifest"` to auto-detect the prebuilt Clang shipped in the synced source tree, or a specific toolchain id resolved via `kernel_toolchains/clang_setup` (matches the workflow's `clang_version` dropdown values). |
| `manifest_<name>` | One entry per selectable OS/manifest combo (see below). The `<name>` (lower-cased) must match one of the workflow's `manifest` dropdown options (`oos14`, `oos15`, `oos16`, `oos16_custom`, `los_22_2`, `los_23_2`, `custom`). |
| `android_version` | Base AOSP platform version (`android12`…`android14`). |
| `kernel_version` | Upstream Linux kernel version (`5.10`, `5.15`, `6.1`). |
| `hmbird` | Reserved for forward compatibility — currently has no consumer in the pipeline. Set `false`. |

Each `manifest_<name>` object needs `url`, `os`, and `os_short`:

* `url` — either a full `https://.../*.xml` URL, or (for stock OnePlus manifests only, when `use_custom_repo_parser` is left on) a bare filename resolved against `https://raw.githubusercontent.com/OnePlusOSS/kernel_manifest/<branch>/`.
* `os` — human-readable OS label for notifications/release notes (`"OxygenOS 16"`, `"LineageOS 23.2"`, `"PixelOS"`).
* `os_short` — short slug used in the build directory name and release tag (`oos16`, `los23.2`, `pos`).

You can add extra scalar fields to a `manifest_<name>` object (e.g. a different `boot_partition_size`) to override the top-level value only when that specific manifest is selected — see `REDMI_12.json` for a real example.

You only need to add the `manifest_*` entries for OS versions you actually intend to build.

## 2. Device-Specific Kernel Patches

Patch application is **directory-driven**, not hardcoded per device in the workflow:

* **Common-kernel patches** (`.github/kernel`, "Add my kernel patches to common kernel"): if `${KERNEL_PATCHES}/kernel_patches/<codename>-<os_short>/common/*.patch` exists in the `nullptr-t-oss/kernel_patches` repo, every patch in it is dry-run tested and applied automatically (conflicting/already-applied patches are skipped, not failed). **No workflow edits are needed** — just create the directory and add patches for your `<codename>-<os_short>`.
* **MSM-kernel patches** (`.github/kernel_modules`, "Add my kernel patches to msm kernel"): same directory-driven mechanism, at `kernel_patches/<codename>-<os_short>/msm/*.patch` — **but this step is currently gated to `env.SOC == 'kalama'` only** (see the `if:` condition on that step). If you're porting a non-`kalama` SoC and need MSM-side patches applied, you'll need to update that condition in `.github/kernel_modules/action.yml`.

  > [!WARNING]
  > That step's condition also checks `!contains(env.MANIFEST_NAME, 'CUSTOM')`, but nothing in the current pipeline ever sets an `env.MANIFEST_NAME` variable — so in practice that half of the condition is always true and has no effect. Don't rely on it; if you need to skip MSM patches for a custom manifest, gate it on `env.MANIFEST` or `inputs.manifest` instead.

* **"Wild kernel" optimization patches** (`.github/kernel`, "Add other kernel patches to common kernel"): a fixed, kernel-version-gated set of generic perf patches from `TheWildJames/kernel_patches`, applied whenever `use_opt_patches` is enabled. These aren't per-device — nothing to configure here when porting.

Device-specific build behavior beyond patches (custom `MAKE_FLAGS`, `CONFIGS`, boot image signing keys, ramdisk compression, DLKM staging, etc.) is controlled by the optional `BUILD_CONFIG` hook file at `kernel_patches/device/<codename>-<os_short>/build_config/build.config.<codename>.<os_short>` — see the build-config-hooks doc for the full hook reference. This is also entirely opt-in per device/OS.

## 3. New SoC Considerations

If your device uses a SoC not yet represented in `configs/`, keep in mind:

* `CONFIGS` defconfig fragment auto-discovery (Monolithic builds) looks for `vendor/${CODENAME}_GKI.config`, `vendor/${SOC}_GKI.config`, and `vendor/oplus/${SOC}_GKI.config` inside the synced kernel tree — make sure your vendor source actually ships one of these, or provide a `CONFIGS` override via `BUILD_CONFIG`.
* The MSM-kernel patch gate described in §2 defaults to `kalama`-only; extend it if your new SoC needs the same treatment.
* `EXT_MODULES` (out-of-tree vendor modules, `.github/kernel_modules`) defaults to a fixed Qualcomm/OPLUS driver list. A non-Qualcomm SoC (e.g. MediaTek, like `POCO_X6_LOS`'s `mt6897`/`strait`) will need an `EXT_MODULES` override via `BUILD_CONFIG` if any out-of-tree modules are needed at all.

## 4. Nethunter & Driver Customization

Nethunter support is **device-agnostic** — the workflow injects the necessary kconfig flags directly into the in-tree defconfig regardless of device:

1. **Kernel side** (`.github/kernel`, "Add Nethunter inline configs"): enables USB serial (generic/CH341/FTDI/PL2303), Airspy & HackRF SDR support, and related USB/Bluetooth HCI config.
2. **Module side** (`.github/kernel_modules`, "Add nethunter specific drivers..."): builds ATH9K, MT76, and RTL88xx Wi-Fi drivers as loadable modules (`=m`).

**Do you need to change anything?**
* **No** — if standard Nethunter support (external Wi-Fi injection, HID attacks, common SDR dongles) is all you need.
* **Yes** — only if you need a specific driver not already added, in which case extend the relevant step in `.github/kernel/action.yml` / `.github/kernel_modules/action.yml`.

## 5. Add the Device to the Workflow Dropdown

Adding `configs/<MODEL>.json` does **not** automatically make it buildable — `set-model` filters against the workflow's `model:` input, so you must add it there too, in `.github/workflows/build-kernel-release.yml`:

```yaml
      model:
        description: 'Select device to be built'
        required: true
        type: choice
        options:
          - OP11
          - OP11_LOS
          - OP12
          - OP12R
          - POCO_F6_LOS
          - POCO_X6_LOS
          - REDMI_15
          - REDMI_12
          - YOUR_NEW_MODEL   # <-- add here
        default: OP11
```

There is currently no "build all devices" option — each run builds exactly one device (the matrix from `set-model` is filtered down to a single `include` entry before the `build` job's `strategy.matrix` runs).

## 6. Build Execution

Once `configs/<MODEL>.json` exists and the model has been added to the dropdown:

1. Go to the **Actions** tab in GitHub.
2. Select **"Build and Release EmberHeart Kernel and Kernel Modules"**.
3. Click **Run workflow**.
4. Set **Model** to your new device and **Manifest** to one of the `manifest_<name>` keys you defined for it.
5. Watch the "Parse op_config_json" and "Set Manifest & grab OS details" step logs in the `Setup Build Environment` job to confirm the expected values (codename, os_short, partition sizes, etc.) were parsed correctly before the source sync even starts.

## 7. Troubleshooting Common Porting Issues

| Issue | Cause | Solution |
|---|---|---|
| `set-model` fails / device not selectable | `configs/<MODEL>.json` missing, or `model` field doesn't match the filename | Confirm the file exists and its `"model"` value matches the filename stem exactly. |
| `No manifest found for key 'manifest_xxx' in config JSON` | Selected `manifest` input has no matching `manifest_<name>` object in the device's JSON | Add the manifest block, or pick a different `manifest` value. |
| `Input 'soc'/'branch' contains invalid characters` | Typo/whitespace in the JSON | `soc` allows `[A-Za-z0-9_-]`, `branch` allows `[A-Za-z0-9._/-]` — no spaces. |
| `repo sync` fails | Incorrect manifest filename/branch, or wrong `use_custom_repo_parser` setting for a bare-filename manifest | Verify the manifest name/branch against the upstream `kernel_manifest` repo you're pointing at. |
| Common-kernel patches never apply | Patch directory not at `kernel_patches/<codename>-<os_short>/common/` (codename/os_short mismatch) | Double-check `$CODENAME` and `$OS_SHORT` in the "Set Manifest & grab OS details" step logs and match the directory name exactly. |
| MSM-kernel patches never apply on a non-`kalama` device | The MSM patch step is hardcoded `if: env.SOC == 'kalama'` | Edit the `if:` condition in `.github/kernel_modules/action.yml` to include your SoC, per §2/§3. |
| Boot/vendor_boot signs with the GKI test key unexpectedly | No custom `AVB_SIGN_BOOT_KEY`/`AVB_SIGN_VENDOR_BOOT_KEY` set | Set it via the device's optional `BUILD_CONFIG` hook file. |
| Wi-Fi module fails to build/rename on a new device | Wrong or missing `qcacld_dev_name`, or the device isn't Qualcomm-based (needs `EXT_MODULES` override instead) | Confirm the correct QCACLD-3.0 variant, or override `EXT_MODULES` via `BUILD_CONFIG` for non-Qualcomm SoCs. |
