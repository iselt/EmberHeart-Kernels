# Device Build Config (`BUILD_CONFIG` / `build.config.<codename>.<os_short>`)

This document covers the **other** build-config mechanism in EmberHeart: the per-device, per-OS shell file that lets you override kernel/module build behavior (defconfig selection, boot image signing, ramdisk compression, DLKM staging, etc.) without touching the composite actions themselves. This is distinct from `configs/*.json` (device metadata used to build the CI matrix) — see `docs/build_config_json.md` (or wherever the metadata doc lives) for that.

## 1. What it is and where it lives

`$BUILD_CONFIG` is an environment variable computed once, in `.github/env_setup/action.yml`:

```bash
BUILD_CONFIG="${KERNEL_PATCHES}/device/${CODENAME}-${OS_SHORT}/build_config/build.config.${CODENAME}.${OS_SHORT}"
```

where `$KERNEL_PATCHES` is `$GITHUB_WORKSPACE/my_patches` — the local clone of `nullptr-t-oss/kernel_patches` created by the "Clone AnyKernel3 and Other Dependencies" step. In other words:

* The file itself is **not in this repository** — it lives in the `nullptr-t-oss/kernel_patches` repo, at:
  ```
  device/<CODENAME>-<OS_SHORT>/build_config/build.config.<CODENAME>.<OS_SHORT>
  ```
* `<CODENAME>` and `<OS_SHORT>` come from the active `configs/*.json` device entry and the selected `manifest_*` block (e.g. `salami-oos16`, `waffle-los23.2`).
* If no such file exists for the current device/OS combo, every consuming step just skips it (`if [[ -f "${BUILD_CONFIG}" ]]`) and pipeline defaults apply — **this file is entirely optional per device/OS pair.**

It is a plain bash script, sourced (not executed) with `set -a; . "${BUILD_CONFIG}"; set +a`, so every variable and function it defines is exported into the environment for the rest of that step (and, since it's re-sourced at each step, for every later step too).

## 2. Where it gets sourced

`$BUILD_CONFIG` is sourced independently in **five** separate steps across the two build actions — once per stage that needs device-specific overrides, since each stage runs as its own shell block:

| Action | Step | Purpose of that stage |
|---|---|---|
| `.github/kernel` | "Build Kernel" | defconfig merge + kernel image compile |
| `.github/kernel` | "Create Boot Image" | `mkbootimg` + AVB boot signing |
| `.github/kernel` | "Create system_dlkm" *(implied by the `SYSTEM_DLKM_*` block around line 1394)* | system_dlkm partition staging/signing |
| `.github/kernel_modules` | out-of-tree module build step | defconfig/module compile, `EXT_MODULES` list |
| `.github/kernel_modules` | vendor_boot creation step | ramdisk compression + `mkbootimg` for `vendor_boot.img` |
| `.github/kernel_modules` | "Create vendor_dlkm" | vendor_dlkm partition staging/signing |

Because it's re-sourced per stage rather than once globally, a hook function only needs to exist in the file — nothing needs to explicitly "register" per stage.

## 3. Supported hooks and variables

Everything below is **optional**. If a variable isn't set, the pipeline falls back to a GKI-appropriate default (shown in the "Default if unset" column). Hooks that reference a shell function name (`PRE_DEFCONFIG_CMDS`, etc.) work by `export -f your_function` followed by `SOME_CMDS_VAR=your_function` — the action does `eval "${SOME_CMDS_VAR}"` at the right point.

### 3.1 Kernel build (`.github/kernel`)

| Variable | Type | When it runs / is used | Default if unset |
|---|---|---|---|
| `PRE_DEFCONFIG_CMDS` | function name | Before `make ${CONFIGS[@]}` | none |
| `POST_DEFCONFIG_CMDS` | function name | After `make ${CONFIGS[@]}`, before `savedefconfig` snapshot | none |
| `MAKE_FLAGS` | bash array | Base `make` flags for every kernel `make` invocation | `LLVM=1 LLVM_IAS=1 ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- LD=$LD DTC=$DTC DTC_FLAGS=-@ HOSTCC="ccache clang" HOSTCXX="ccache clang++" CC="ccache clang"` |
| `MAKE_FLAGS_EXTRA` | bash array | Appended after `MAKE_FLAGS` on every invocation | empty |
| `CONFIGS` | bash array | Defconfig + fragment list passed to `make ... "${CONFIGS[@]}"` | `gki_defconfig`, plus (Monolithic builds only) any of `vendor/${CODENAME}_GKI.config`, `vendor/${SOC}_GKI.config`, `vendor/oplus/${SOC}_GKI.config`, `vendor/debugfs.config` that exist in-tree. `kernel.config` is always appended last regardless of override. |
| `PRE_BOOT_MKBOOTIMG_CMDS` | function name | Before `mkbootimg` runs for `boot.img` | none |
| `BOOT_EXTRA_CMDS` | function name | Right after `PRE_BOOT_MKBOOTIMG_CMDS`, still before `mkbootimg` | none |
| `BOOT_MKBOOTIMG_ARGS` | bash array | Full argv passed to `mkbootimg` for `boot.img` | `--header_version 4 --os_version 13.0.0 --os_patch_level 2099-12-01 --pagesize 4096 --kernel $IMAGE --output $OUT/boot.img` (assumes a v4, no-ramdisk boot image) |
| `POST_BOOT_MKBOOTIMG_CMDS` | function name | After `mkbootimg`, before signing | none |
| `AVB_SIGN_BOOT_CMDS` | function name | Fully replaces the built-in signing step when set | none — built-in `avbtool` signing runs instead |
| `AVB_SIGN_BOOT_IMG` | `0`/`1` | Gate for built-in signing (only checked if `AVB_SIGN_BOOT_CMDS` unset) | `1` (sign) |
| `AVB_SIGN_BOOT_ALGO` | string | AVB signing algorithm | `SHA256_RSA4096` |
| `AVB_SIGN_BOOT_KEY` | path | AVB signing key | GKI test key (`$KERNEL_PLATFORM/tools/mkbootimg/gki/testdata/testkey_rsa4096.pem`) |
| `SYSTEM_DLKM_EXTRA_CMDS` | function name | After DLKM staging tree is populated, before image build | none |
| `SYSTEM_DLKM_EXTRA_FILES` | function name | Decides which extra files land under `${SYSTEM_DLKM_STAGING_DIR}/etc`; **when set, replaces** the default `build.prop`/`fs_config_*` copy entirely | copies `${PREBUILTS}/etc/build.prop`, touches empty `fs_config_dirs`/`fs_config_files` |
| `SYSTEM_DLKM_PROPS_FILE` | path | Extra props appended into the generated `system_dlkm.prop` | none (a `system_dlkm.prop` already in `${PREBUILTS}` takes priority regardless) |

`SYSTEM_DLKM_STAGING_DIR` is set *by the pipeline* (`${OUT}/system_dlkm_staging`) and is available for your hook functions to read — you don't set it yourself.

### 3.2 Module build (`.github/kernel_modules`)

Shares `PRE_DEFCONFIG_CMDS`/`POST_DEFCONFIG_CMDS`/`MAKE_FLAGS`/`MAKE_FLAGS_EXTRA`/`CONFIGS`/`SYSTEM_DLKM_*` with the same semantics as above (this action also builds a system_dlkm variant in mixed/monolithic builds), plus these module-specific ones:

| Variable | Type | When it runs / is used | Default if unset |
|---|---|---|---|
| `EXT_MODULES` | bash array | The full list of out-of-tree vendor module source dirs to `make -C` / `modules_install` | A fixed list of ~26 QCOM/OPLUS/NXP/ST vendor driver paths (mmrm-driver, display-drivers, camera-kernel, audio-kernel, `wlan/qcacld-3.0/.${QCACLD_DEV_NAME}`, etc. — see the action for the full list) |
| `LZ4_RAMDISK` | `0`/`1` | Whether the `vendor_boot` ramdisk is LZ4- or gzip-compressed | `1` (LZ4) |
| `LZ4_RAMDISK_COMPRESS_ARGS` | string | Extra args to the `lz4` compressor (only used when `LZ4_RAMDISK=1`) | `-12 --favor-decSpeed` |
| `PRE_VENDOR_BOOT_MKBOOTIMG_CMDS` | function name | Before `mkbootimg` for `vendor_boot.img` | none |
| `VENDOR_BOOT_EXTRA_CMDS` | function name | Right after, still before `mkbootimg` | none |
| `VENDOR_BOOT_MKBOOTIMG_ARGS` | bash array | Full argv passed to `mkbootimg` for `vendor_boot.img` | `--header_version 4 --pagesize 4096 --vendor_cmdline "$VENDOR_CMDLINE" --dtb $OUT/dtb.img --vendor_bootconfig $OUT/vendor-bootconfig.img --vendor_boot $OUT/vendor_boot.img --vendor_ramdisk $OUT/ramdisk.$RAMDISK_EXT` |
| `POST_VENDOR_BOOT_MKBOOTIMG_CMDS` | function name | After `mkbootimg`, before signing | none |
| `AVB_SIGN_VENDOR_BOOT_CMDS` | function name | Fully replaces built-in signing when set | none |
| `AVB_SIGN_VENDOR_BOOT_IMG` | `0`/`1` | Gate for built-in signing | `1` |
| `AVB_SIGN_VENDOR_BOOT_ALGO` | string | AVB signing algorithm | `SHA256_RSA4096` |
| `AVB_SIGN_VENDOR_BOOT_KEY` | path | AVB signing key | GKI test key |
| `VENDOR_DLKM_EXTRA_CMDS` | function name | After vendor_dlkm staging populated, before image build | none |
| `VENDOR_DLKM_EXTRA_FILES` | function name | Which extra files land in the vendor_dlkm staging tree | pipeline default file set |
| `VENDOR_DLKM_PROPS_FILE` | path | Extra props appended into `vendor_dlkm.prop` | none |

`VENDOR_DLKM_STAGING_DIR` (and `SYSTEM_DLKM_STAGING_DIR` for the modules-side DLKM pass) are set by the pipeline for hook functions to consume, same as the kernel action.

## 4. Override semantics (important)

Every one of these follows the same bash idiom:

```bash
if [[ ! -v SOME_VAR ]]; then
  SOME_VAR=(...)   # pipeline default
fi
```

or, for the `*_CMDS` hooks:

```bash
if [[ -n "${SOME_CMDS:-}" ]]; then
  eval "${SOME_CMDS}"
fi
```

That means:

* **Array/scalar overrides (`MAKE_FLAGS`, `CONFIGS`, `BOOT_MKBOOTIMG_ARGS`, `EXT_MODULES`, `LZ4_RAMDISK`, the `AVB_SIGN_*` toggles, etc.) are all-or-nothing.** If `BUILD_CONFIG` sets the variable at all (even to an empty array), the pipeline's built-in default for that variable is skipped entirely — there's no merging. `CONFIGS` is a partial exception: even when you override it, `kernel.config` is still appended after your array in the kernel action.
* **`*_CMDS` hooks are pure injection points**, not overrides of pipeline logic — they run *in addition to* the surrounding steps, at the specific point named (pre/post defconfig, pre/post mkbootimg, etc.), except for `AVB_SIGN_BOOT_CMDS`/`AVB_SIGN_VENDOR_BOOT_CMDS`, which **replace** the built-in `avbtool` signing entirely when set.
* Because the hooks are `eval`'d function names, define the function first, e.g.:
  ```bash
  my_post_defconfig() {
    echo "CONFIG_SOME_FEATURE=y" >> "${OUT}/.config"
  }
  POST_DEFCONFIG_CMDS=my_post_defconfig
  ```

## 5. Minimal example

`device/salami-oos16/build_config/build.config.salami.oos16` in `kernel_patches`:

```bash
# Disable a config not needed on this device before the defconfig merge
op11_pre_defconfig() {
  echo "# CONFIG_SOME_UNUSED_FEATURE is not set" >> "${KERNEL_DIR}/arch/arm64/configs/vendor/salami_GKI.config"
}
PRE_DEFCONFIG_CMDS=op11_pre_defconfig

# Use gzip instead of lz4 for the vendor_boot ramdisk on this device
LZ4_RAMDISK=0

# Sign boot.img with a project key instead of the GKI test key
AVB_SIGN_BOOT_KEY="${KERNEL_PATCHES}/device/salami-oos16/keys/boot_key.pem"
```

Nothing in `configs/*.json` needs to change for this — the file is picked up automatically once it exists at the codename/os_short path the pipeline already computes.

## 6. Debugging checklist

| Symptom | Likely cause |
|---|---|
| Your overrides never seem to apply | Check the `$BUILD_CONFIG` path was computed correctly — it depends on `$CODENAME` (from `configs/*.json`) and `$OS_SHORT` (from the selected `manifest_*.os_short`), and the file must exist at exactly `device/<codename>-<os_short>/build_config/build.config.<codename>.<os_short>` in `kernel_patches`. |
| `CONFIGS` override didn't drop the default `gki_defconfig` | Expected if you appended instead of replaced — remember `[[ ! -v CONFIGS ]]` is all-or-nothing; setting `CONFIGS=("myconfig")` fully replaces the default list (though `kernel.config` is still appended after). |
| A `*_CMDS` hook silently does nothing | Confirm you exported the function (`set -a` makes plain variable assignments exported, but a `function foo(){...}` still needs `export -f foo` if you want it visible inside any subshells the hook itself spawns) and that the variable holding the function name has no typos. |
| Boot/vendor_boot signing uses the GKI test key unexpectedly | `AVB_SIGN_BOOT_KEY`/`AVB_SIGN_VENDOR_BOOT_KEY` weren't set, or `AVB_SIGN_BOOT_CMDS`/`AVB_SIGN_VENDOR_BOOT_CMDS` weren't used to fully replace signing when you needed to. |
